# AI 自动化金融研究认知系统 — 设计文档

> 本文档记录了对该系统从零开始的设计讨论，作为后续开发的蓝图。
> 最后更新：2026-05-31

---

## 一、系统目标

将传统买方研究员的日常信息处理工作 **自动化**：

- 每天摄入约 30 份电话会议记录 + 200 份卖方研报
- 多个覆盖 Agent 并行处理各自板块的相关材料
- 自动识别单板块、跨板块、产业链上下游的基本面变化
- 输出每日 delta 报告（今天发生了什么变化，对我的研究有何影响）
- 日积月累地维护一张活的"研究认知图"

**核心价值主张**：不是让 AI 替代分析师，而是让分析师的覆盖广度和信息处理深度大幅提升。

---

## 二、整体架构

```
                    每日文档批次（~230篇）
                           │
          ┌────────────────▼────────────────┐
          │     层 0：文档摄入 & 路由        │
          │  元数据提取 → 覆盖范围映射分桶   │
          └────────────────┬────────────────┘
                           │ fan-out
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
  ┌────────────┐   ┌────────────┐   ┌────────────┐
  │ 消费品     │   │ 半导体     │   │ 新能源     │  ... × N
  │ Coverage   │   │ Coverage   │   │ Coverage   │
  │   Agent    │   │   Agent    │   │   Agent    │
  │（持久化    │   │            │   │            │
  │  Workspace）│  │            │   │            │
  └─────┬──────┘   └─────┬──────┘   └─────┬──────┘
        │                │                │
        ▼                ▼                ▼
    写入结构化        写入结构化        写入结构化
    Signal JSON      Signal JSON      Signal JSON
        │                │                │
        └────────────────┼────────────────┘
                         │ fan-in
              ┌──────────▼──────────┐
              │  跨板块 Synthesizer  │
              │      Agent          │
              │  + 对抗性验证       │
              └──────────┬──────────┘
                         │
                   ┌─────▼─────┐
                   │  Inbox 推送 │
                   │ 今日 delta  │
                   └─────┬─────┘
                         │
              ┌──────────▼──────────┐
              │  持久化层（MCP）     │
              │  Signal Board       │
              │  产业链关系图        │
              │  Coverage 知识文件   │
              └─────────────────────┘
```

**执行时序**：
```
19:00  文档到达，路由分发
19:05  N 个 Coverage Agent 并行启动（Claude Dynamic Workflows 编排）
19:50  Coverage Agent 完成，各自写入 Signal Board
19:51  Synthesizer Agent 启动，读取全部 signals + 产业链图
20:10  Synthesizer 完成，推送今日报告到 Inbox
20:15  用户查看整合报告，审核新关系提案
```

---

## 三、层 0：文档摄入 & 路由

### 路由三步法

```
文档（PDF / TXT）
       │
       ▼
步骤 1：元数据提取（纯规则，0 token 成本）
  - 从标题提取：公司名、股票代码、发布日期
  - 电话会：主办公司 + 日期
  - 行业标签（通常在标题/首行）
       │
       ▼
步骤 2：覆盖范围映射（查表）
  - coverage_map.yaml：定义每个 Agent 覆盖的股票/板块/关键词
  - 一次 O(n) 扫描，文档分配到对应 Agent 的 inbox
       │
       ▼
步骤 3：无法路由的文档（可选）
  - 调用 Haiku 读取摘要，判断属于哪个板块
  - 成本极低（只看标题 + 前200字）
```

**关键原则**：路由必须在 Coverage Agent 启动**之前**完成。每个 Agent 收到的已经是筛好的相关材料，不让每个 Agent 自己去读全部 230 篇。

### coverage_map.yaml 示例

```yaml
coverage_groups:
  consumer_staples:
    stocks: ["600519", "000858", "002304", "603288"]
    sectors: ["白酒", "啤酒", "调味品", "乳制品"]
    keywords: ["茅台", "五粮液", "伊利", "海天"]

  new_energy:
    stocks: ["300750", "002594", "688005"]
    sectors: ["动力电池", "新能源整车", "储能"]
    keywords: ["宁德", "比亚迪", "亿纬锂能"]

  semiconductor:
    stocks: ["688981", "603501", "002371"]
    sectors: ["芯片设计", "晶圆制造", "封测"]
    keywords: ["中芯", "韦尔股份", "卓胜微"]
```

---

## 四、Coverage Agent 设计

### Workspace 结构

每个覆盖组是一个**长寿命 Workspace**，日积月累地积累认知：

```
workspace/{sector-name}/
├── AGENTS.md              ← 人格定义 + 职责范围（注入给 claude/codex）
├── coverage/
│   ├── universe.yaml      ← 覆盖股票清单 + 当前评级
│   ├── {stock_code}.md    ← 每只股票的当前认知状态（两层结构，见下）
│   └── sector_view.md     ← 对整个板块的当前判断
├── inbox/
│   └── {YYYY-MM-DD}/      ← 今天路由过来的相关文档
│       ├── call_01.txt
│       └── sellside_07.txt
└── log/
    └── deltas.md          ← 每日 delta，append-only
```

### 每只股票的知识文件结构：Compiled Truth + Timeline 两层模式

> 来自 gbrain 调研（garrytan/gbrain）：上层重写（当前状态）+ 下层追加（证据日志）。
> 这解决了"记忆要么过时、要么无限膨胀"的经典问题。

```markdown
# 贵州茅台 (600519)

<!-- ★ COMPILED TRUTH — 有新信息就整体重写这一区域 ★ -->

## 核心判断
- 评级：买入
- 核心逻辑：高端白酒消费韧性 + 渠道库存改善进行时
- 最后综合：2026-05-31

## 关键变量追踪（当前状态）
| 变量 | 当前状态 | 判断方向 |
|------|----------|----------|
| Q2 动销 | 强于预期 | 正面 |
| 渠道库存 | 改善中 | 正面 |
| 批价（飞天） | 1480 | 中性 |

## 近期催化剂
- 中报窗口：2026-08

## 风险因素
- 宏观消费降级
- 政务消费监管趋严

---
<!-- ★ TIMELINE — 只追加，永不删除，最新在前 ★ -->

- 2026-05-31 | 茅台电话会议 | 管理层确认Q2旺季动销超预期，批价稳定
- 2026-05-28 | 渠道调研报告（国信证券） | 飞天批价1480，环比+2%，库存水平改善
- 2026-05-20 | 卖方点评（中信证券） | 维持买入，目标价1850
```

**关键设计原则**：
- Agent 更新时：重写横线以上的 Compiled Truth，在 Timeline 追加新条目
- 精准 delta 检测：比较前后两次的 Compiled Truth 表格 diff
- 证据可追溯：所有判断都能在 Timeline 找到来源

### Agent 每日执行流程（注入到 AGENTS.md 的指令）

```
1. 读取 coverage/*.md（我目前对这些股票/行业的理解是什么？）
2. 读取 inbox/today/ 的所有文档（今天来了哪些新材料？）
3. 对每份文档提取关键信息：
   - 是否改变了我对某只股票的判断？
   - 有什么数据 / 管理层表态 / 估值逻辑的变化？
4. 生成 delta：确认/修正/挑战/新信息
5. 更新 coverage/*.md（写回新认知）
6. 写入 shared/signals/today/{sector}.signals.json
7. 调用 inbox_push 推送今日要点
```

---

## 五、Signal Board（MCP Server）

### 为什么用 MCP Server

把研究知识暴露为 MCP Server，所有 Agent 都能读写，是最干净的接口设计：
- Coverage Agent 写入今日发现
- Synthesizer Agent 读取全部信号做跨板块分析
- 未来任何新 Agent 都能即插即用

### Signal JSON 格式

```json
{
  "sector": "new_energy_upstream",
  "signal_type": "cost_input_change",
  "direction": "bearish_cost",
  "magnitude": "significant",
  "detail": "碳酸锂现货价格本周下降15%，创年内新低，电话会3家企业均提及",
  "propagation_to": ["battery_manufacturers", "nev_oem"],
  "confidence": "high",
  "sources": ["call_transcript_catl_2026-05-31.txt", "sellside_guosen_battery_20260531.txt"],
  "timestamp": "2026-05-31T19:45:00Z"
}
```

**`propagation_to` 字段是核心**：Coverage Agent 在自己领域判断信号会传导到哪里，而不是让 Synthesizer 猜。

### MCP Server 暴露的接口

```typescript
// Coverage Agent 调用
update_weight(edge_id, delta, evidence)     // 更新产业链关系强度
propose_relationship(from, to, evidence)   // 提案新关系（待人工审核）
write_signal(sector, signal_json)          // 写入今日信号

// Synthesizer Agent 调用
get_propagation(signal_type, from_sector)  // 这个信号传导到哪些下游？
query_map(nl_query)                        // 自然语言查产业链图
get_today_signals(filter?)                 // 获取今日全部信号

// 人工查看
review_proposals()                         // 查看 AI 提案的新关系
get_map_health()                           // 全图健康状态（哪些边在衰减）
```

---

## 六、产业链关系图（动态学习）

### 两层架构

```
supply_chain_map/
├── structure.yaml         ← 骨架，人工维护，AI 不能直接改
├── dynamics.json          ← 权重层，AI 可自主更新
├── proposals.md           ← AI 提案的新关系，等待人工审核
└── audit_log.jsonl        ← 所有变更的证据链
```

**核心原则**：
- **AI 可自主更新**：`dynamics.json` 里的关系强度、当前状态
- **人工审核才能改**：`structure.yaml` 里的结构性关系

### structure.yaml 示例

```yaml
supply_chains:
  nev_chain:
    edges:
      - id: edge_001
        from: lithium_carbonate
        to: cathode_material
        type: cost_input
      - id: edge_002
        from: cathode_material
        to: battery_cell
        type: cost_input
      - id: edge_003
        from: battery_cell
        to: nev_oem
        type: cost_input

  semiconductor_chain:
    edges:
      - id: edge_010
        from: silicon_wafer
        to: foundry
        type: material_input

cross_sector_correlations:
  - trigger: ppi_upstream_up
    affected: [midstream_processing, downstream_manufacturing]
    direction: cost_pressure
    typical_lag_weeks: "2-4"
```

### dynamics.json 示例

```json
{
  "edge_001": {
    "strength": 0.85,
    "direction": "positive_cost_push",
    "last_confirmed": "2026-05-31",
    "evidence_count": 12,
    "decay_counter": 0,
    "status": "active",
    "latest_note": "锂价下跌15%，传导强度高，宁德未锁长协"
  },
  "edge_002": {
    "strength": 0.62,
    "last_confirmed": "2026-05-28",
    "decay_counter": 3,
    "status": "active",
    "latest_note": "宁德已锁定部分长协价，短期传导略减弱"
  }
}
```

### 置信度衰减机制

```python
# 每天 Synthesizer 运行后执行
for edge in dynamics:
    if edge has today's confirming signal:
        edge.strength += 0.05
        edge.decay_counter = 0
    else:
        edge.decay_counter += 1
    
    if edge.decay_counter > 30:   # 30天没有信号
        edge.strength -= 0.02     # 自然衰减
    
    if edge.strength < 0.2:
        edge.status = 'dormant'   # 提示人工确认是否失效
```

### 初始建图策略

1. 给 AI 你的覆盖范围列表（行业 + 股票）
2. AI 生成初稿 `structure.yaml`（主要产业链关系）
3. 你审核并修正（领域知识用来纠错，不是从零创造）
4. 系统上线后每天自动更新 `dynamics.json`
5. 你只需定期审核 `proposals.md` 里的新关系提案

---

## 七、Synthesizer Agent（跨板块分析）

### 三类输出

**① 产业链传导信号**
```
上游锂价下跌（-15%）
  → 传导至中游：电池厂成本改善（利好）[edge_001 strength: 0.85]
  → 传导至下游：整车成本压力缓解，但传导到终端售价有 1-2 季度滞后
结论：现阶段利好电池厂 > 整车厂
```

**② 个股 vs 板块背离**
```
新能源板块信号整体偏多，但 X 公司电话会管理层措辞谨慎 + 未提指引
背离度：高
建议：单独深入看 X 公司
```

**③ 多板块共振确认**
```
消费品：渠道反馈动销恢复
银行：信贷数据居民消费贷增加
地产：二手房成交量边际好转
→ 三个独立板块共同指向居民消费意愿修复
→ 宏观层面 beta 信号，非结构性
```

### 内置对抗性验证

Dynamic Workflows 的 Adversarial Verification 在这里天然适用：
- Agent A 从信号里推断跨板块结论
- Agent B 专门尝试反驳 Agent A 的结论
- 迭代直到收敛

---

## 八、技术选型

### 编排层：Claude Dynamic Workflows

```javascript
// 每日 workflow 脚本（由 Claude 自动生成）
const sectors = loadCoverageSectors()

// Phase 1: 文档路由（Haiku，快速便宜）
const routed = await parallel(sectors.map(s => () =>
  agent(`路由今日文档到 ${s.name} 覆盖范围`, {
    model: 'haiku', schema: ROUTING_SCHEMA
  })
))

// Phase 2: 并行 Coverage Agents（Sonnet）
const signals = await parallel(sectors.map(s => () =>
  agent(`分析 ${s.name} 今日材料，更新认知`, {
    model: 'sonnet', schema: SIGNAL_SCHEMA,
    isolation: 'worktree'
  })
))

// Phase 3: 对抗性验证 + 综合（Opus）
const synthesis = await pipeline(
  signals,
  s => agent(`跨板块传导分析`, { model: 'opus' }),
  draft => agent(`尝试反驳以下结论，找出错误`, { model: 'sonnet' })
)
```

### 模型分层（成本控制关键）

| 任务 | 模型 | 原因 |
|---|---|---|
| 文档分类/路由 | Haiku | 简单任务，便宜 3 倍 |
| Coverage Agent | Sonnet | 日常分析，性价比最高 |
| Synthesizer | Opus | 复杂跨板块推理 |

### 信号持久化：本地 MCP Server

- 语言：Python（`fastmcp` 库）或 TypeScript
- 存储：文件系统（JSONL + YAML），git 追踪变更
- 接口：上文所列的 MCP 工具集

---

## 九、与 OpenAlice 现有架构的关系

**直接借用**：
- `Workspace` + `bootstrap.sh` 模式 → 每个覆盖组的环境注入
- `inbox_push` 工具 → Agent 推送 delta 报告
- `Pump` 调度器 → 每天定时触发编排器

**跳过不用**：
- `AgentCenter` / `ConnectorCenter`（旧的进程内 AI 路径）
- `VercelAIProvider` / `AgentSdkProvider`（进程内 AI 调用层）
- `Telegram` 连接器

**参考 Anthropic 金融服务模板**：
- `Earnings Reviewer` → Coverage Agent 的 prompt 起点
- `Market Researcher` → Synthesizer Agent 的参考架构

---

## 十、Dream Cycle（夜间自维护）

> 来自 gbrain 调研：日常摄入只是系统的一半，夜间自修复才是让系统长期可用的关键。

每晚在 Coverage Agent 完成后，额外运行一个维护周期：

```
每日 Coverage Agent（摄入新内容）
        +
Dream Cycle 每晚 02:00（自我修复）
```

### Dream Cycle 阶段

```
Phase 1: 修复坏掉的引用和链接
  → 检查 coverage/*.md 里提到的公司/产品，确保都有对应页面

Phase 2: 充实薄知识
  → 某公司只被提到 1 次（stub）→ 自动触发 web enrichment
  → 被提到 3+ 次 → 完整分析

Phase 3: 重新计算产业链关系强度
  → 遍历 dynamics.json，更新 decay_counter
  → 将 strength < 0.2 的边标记为 dormant

Phase 4: 跨 session 模式检测
  → "最近3周锂价持续下跌" 这类跨日的趋势
  → 合并重复的 timeline 条目

Phase 5: Gap Analysis 生成
  → 哪个板块/环节本周没有新材料
  → 哪些知识文件超过 7 天未更新
  → 推送 Gap Report 到 Inbox
```

**Gap Analysis 示例输出**：
```
今日缺口报告：
- 上游锂资源：最后更新 5 天前，近期无相关材料进入
- 半导体设备：本周0篇卖方点评，关注度低
- 茅台渠道数据：已7天未更新，建议主动补充
```

---

## 十一、Signal Board 存储升级（Postgres 方案）

> 来自 gbrain 调研：Postgres + pgvector 一个实例可以搞定向量搜索 + 关键词搜索 + 图谱遍历，不需要专用图数据库或向量数据库。

### 推荐存储方案

用 **PGLite**（WASM 嵌入式 Postgres，2 秒启动，零配置）替代 JSONL 文件：

```sql
-- 产业链节点（公司/板块/商品）
CREATE TABLE supply_chain_nodes (
  slug TEXT PRIMARY KEY,          -- e.g. "lithium_carbonate"
  name TEXT,
  node_type TEXT,                  -- upstream/midstream/downstream
  compiled_truth TEXT,             -- 当前综合状态
  timeline TEXT,                   -- 追加式证据日志
  emotional_weight FLOAT,          -- 显著度 0..1
  updated_at TIMESTAMP
);

-- 产业链边（关系）
CREATE TABLE supply_chain_edges (
  id TEXT PRIMARY KEY,
  from_slug TEXT,
  to_slug TEXT,
  edge_type TEXT,                  -- cost_input/volume_demand/etc
  strength FLOAT,                  -- 0..1
  decay_counter INT DEFAULT 0,
  status TEXT DEFAULT 'active',    -- active/dormant
  last_confirmed DATE,
  latest_note TEXT
);

-- 每日信号
CREATE TABLE daily_signals (
  id SERIAL PRIMARY KEY,
  sector TEXT,
  signal_date DATE,
  signal_type TEXT,
  direction TEXT,
  magnitude TEXT,
  detail TEXT,
  propagation_to TEXT[],
  confidence TEXT,
  sources TEXT[],
  embedding vector(1536)           -- 向量化，支持语义搜索历史信号
);
```

**混合检索**（参考 gbrain 的 RRF Fusion）：
```sql
-- 向量搜索：找语义相关的历史信号
SELECT * FROM daily_signals
ORDER BY embedding <=> $query_vector LIMIT 20;

-- 关键词搜索：精确匹配板块/公司
SELECT * FROM daily_signals
WHERE to_tsvector(detail) @@ to_tsquery('锂 & 电池');

-- 图谱遍历：产业链传导
WITH RECURSIVE chain AS (
  SELECT * FROM supply_chain_edges WHERE from_slug = 'lithium_carbonate'
  UNION ALL
  SELECT e.* FROM supply_chain_edges e JOIN chain c ON e.from_slug = c.to_slug
)
SELECT * FROM chain;
```

---

## 十二、开发路线图（待细化）

### Phase 1：单板块 MVP
- [ ] 实现文档路由（规则 + 关键词匹配）
- [ ] 建立第一个覆盖组 Workspace（选一个最熟悉的板块）
- [ ] 初始化 `structure.yaml`（AI 生成草稿，人工审核）
- [ ] 实现基本版 Signal Board MCP Server（PGLite 后端）
- [ ] 手动跑第一次 Coverage Agent（验证 Compiled Truth + Timeline 格式）

### Phase 2：多板块并行
- [ ] 扩展到 3-5 个覆盖组
- [ ] 用 Dynamic Workflows 编排并行
- [ ] 实现 Synthesizer Agent + Gap Analysis 输出

### Phase 3：动态学习 + Dream Cycle
- [ ] 实现 Dream Cycle（夜间维护周期）
- [ ] 实现 `dynamics.json` 自动更新
- [ ] 实现置信度衰减机制
- [ ] 实现 `proposals.md` 提案流程
- [ ] 建立每日 delta 的评估机制

### Phase 4：完善
- [ ] 模型分层优化（成本控制）
- [ ] 对抗性验证集成
- [ ] 文档摄入接口（对接实际文档来源）
- [ ] 混合搜索（向量 + 关键词 + RRF）的信号检索

---

## 附录 A：关键设计决策记录

| 决策 | 选择 | 理由 |
|---|---|---|
| 路由方式 | 规则优先，AI 兜底 | 成本控制，避免 N 个 Agent 各自读 230 篇 |
| 编排框架 | Claude Dynamic Workflows | 内置 fan-out/fan-in + 对抗性验证，无需自建 |
| 持久化接口 | 本地 MCP Server（PGLite 后端） | 所有 Agent 即插即用，向量+关键词+图谱三合一 |
| 产业链图更新 | 两层（结构人工 + 权重 AI） | 防止 AI 自主引入错误关系 |
| Workspace 设计 | 长寿命持久化 | 日积月累的认知不应每次重建 |
| 信号格式 | 结构化 JSON + Postgres 存储 | 机器可读 + 向量搜索历史信号 |
| 知识文件格式 | Compiled Truth + Timeline 两层 | 上层重写（当前状态），下层追加（证据日志） |
| 夜间维护 | Dream Cycle | 日常摄入 + 夜间自修复，才能长期可用 |

---

## 附录 B：参考项目调研

### garrytan/gbrain（2026-05-31 调研）

**项目**：YC CEO Garry Tan 的个人 AI 知识大脑系统。20K stars，MIT 开源。生产规模：146K 页面，66 个 cron 任务。

**我们采用的设计**：
1. **Compiled Truth + Timeline 两层页面模式**：知识文件分割为「当前状态（重写）」和「证据时间线（追加）」两层
2. **Dream Cycle 夜间维护**：不只是摄入新内容，还主动修复引用、充实薄知识、检测跨日模式
3. **Postgres/PGLite 存储**：不需要专用图数据库或向量数据库，一个 Postgres 搞定所有需求
4. **Gap Analysis**：Synthesizer 输出除"已知"外还明确报告"缺口"

**我们不采用的部分**：
- Markdown-in-git 作为 source of truth（我们的知识是结构化的）
- OAuth 2.1（个人使用不需要）
- 代码分析功能（不在我们场景内）

**官方链接**：https://github.com/garrytan/gbrain
