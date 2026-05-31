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
│   ├── {stock_code}.md    ← 每只股票的当前认知状态（活的文件）
│   └── sector_view.md     ← 对整个板块的当前判断
├── inbox/
│   └── {YYYY-MM-DD}/      ← 今天路由过来的相关文档
│       ├── call_01.txt
│       └── sellside_07.txt
└── log/
    └── deltas.md          ← 每日 delta，append-only
```

### 每只股票的知识文件结构（`{stock_code}.md`）

```markdown
# 贵州茅台 (600519) — 最后更新 2026-05-31

## 核心判断
- 评级：买入
- 核心逻辑：高端白酒消费韧性 + 渠道库存改善进行时

## 关键变量追踪
| 变量 | 上次判断 | 最新数据 | 更新日期 |
|------|----------|----------|----------|
| Q2 动销 | 平稳 | 强于预期 | 2026-05-31 |
| 渠道库存 | 偏高 | 改善中 | 2026-05-28 |
| 批价（飞天） | 1450 | 1480 | 2026-05-30 |

## 近期催化剂
- 中报窗口：2026-08

## 风险因素
- 宏观消费降级
- 政务消费监管趋严
```

**设计原理**：有了这个结构，"今天有没有变化"就是表格的 diff，而不是两段自由文本的语义比较——精确且可追溯。

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

## 十、开发路线图（待细化）

### Phase 1：单板块 MVP
- [ ] 实现文档路由（规则 + 关键词匹配）
- [ ] 建立第一个覆盖组 Workspace（选一个最熟悉的板块）
- [ ] 初始化 `structure.yaml`（AI 生成草稿，人工审核）
- [ ] 实现基本版 Signal Board MCP Server
- [ ] 手动跑第一次 Coverage Agent

### Phase 2：多板块并行
- [ ] 扩展到 3-5 个覆盖组
- [ ] 用 Dynamic Workflows 编排并行
- [ ] 实现 Synthesizer Agent

### Phase 3：动态学习
- [ ] 实现 `dynamics.json` 自动更新
- [ ] 实现置信度衰减机制
- [ ] 实现 `proposals.md` 提案流程
- [ ] 建立每日 delta 的评估机制

### Phase 4：完善
- [ ] 模型分层优化（成本控制）
- [ ] 对抗性验证集成
- [ ] 文档摄入接口（对接实际文档来源）

---

## 附录：关键设计决策记录

| 决策 | 选择 | 理由 |
|---|---|---|
| 路由方式 | 规则优先，AI 兜底 | 成本控制，避免 N 个 Agent 各自读 230 篇 |
| 编排框架 | Claude Dynamic Workflows | 内置 fan-out/fan-in + 对抗性验证，无需自建 |
| 持久化接口 | 本地 MCP Server | 所有 Agent 即插即用，接口统一 |
| 产业链图更新 | 两层（结构人工 + 权重 AI） | 防止 AI 自主引入错误关系 |
| Workspace 设计 | 长寿命持久化 | 日积月累的认知不应每次重建 |
| 信号格式 | 结构化 JSON | 机器可读，才能做真正的跨板块分析 |
