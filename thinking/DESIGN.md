# 设计：线上事故排查辅助平台 MVP

> 业务一句话：做一个公司线上事故排查辅助平台——后端 AI 复现资深工程师接到事故后的排查动作（查日志、据线索查代码、必要时查库验数据），给出带置信度的可能性 + 证据 + 影响 + 风险，前端展示，人确认后归档成可检索的调查记录。一个月出 MVP。
> 类型：全新项目（greenfield，无既有代码）
> 技术栈：Java / Spring Boot 后端 + Web 前端 + Postgres
> 日期：2026-06-05

## 1. 业务目标

**不是"根因神谕"，不以准确率考核。** 这东西的价值是**省下工程师排查时间**：把资深工程师"接到事故 → 查日志找线索 → 据线索查代码定位 → 必要时查库验数据 → 形成假设"这套动作自动化掉，产出**排序的可能性 + 各自置信度 + 支撑证据链 + 建议下一步**。人始终在回路里做最终判断——所以它不需要永远对，它需要**走对路、把证据摆齐、诚实标置信度**。被人确认过的调查记录沉淀成团队长期资产。

## 2. 调查链（产品的核心 spec）

这就是 AI 要复现的资深工程师 playbook：

```
工程师手输症状（受影响服务 / 时间窗 / 现象级别，可粘堆栈）
  │
  ├─① 日志       查日志平台 API，捞 error / 异常栈 / 关键时间点      → 拿到线索
  │
  ├─② 代码       据日志线索精确取文件 + blame + 最近 commit，
  │              定位可疑方法 / 改动；顺带抽"数据模型线索"
  │              （涉及哪个 entity/DTO、映射到哪张表哪些字段）       → 解释为什么
  │
  ├─③ DB（条件）  用②给出的表/字段，组装只读 SELECT 验证数据对不对    → 确认假设
  │
  └─④ 汇总       排序的可能性 + 每条置信度 + 支撑证据 + 建议下一步
                  → 人确认（accept / reject / edit）→ 归档
```

②→③是串行依赖：**先看懂代码，再有目的地查库**（不是瞎查）。这是整个产品最像资深工程师、也最值钱的地方（见待确认 #3 的编排取舍）。

## 3. 关键架构判断

### 3.1 确定性流水线，不是 agent loop

按事故走固定步骤采集证据 → 把证据打包成结构化 context → 让内部 LLM **一次性输出强约束 JSON**（排序可能性 / 置信度 / 影响 / 爆炸半径 / 风险 / 建议）。不让模型自由决定调哪个工具。

理由：内部 LLM 是否支持 tool-use 待确认；确定性流水线可调试、成本可控、好 demo、出问题能定位到具体环节。agent loop 留 v2。

### 3.2 连接器可插拔，证据强类型化

```java
interface Connector {
    boolean supports(IncidentContext ctx);   // 我能不能为这个事故出证据
    Evidence fetch(EvidenceQuery query);      // 取证并归一化成强类型证据
}
// MVP 实现：LogConnector / CodeConnector / DbConnector
```

每条证据强类型化：`{source, type, timestamp, raw, summary, relevance}`。加新源不动主流程。

### 3.3 证据缺失要降级，绝不假装查过

某个连接器超时/挂了，流水线带着**部分证据继续**，记录里如实标注"日志源不可用 / 无数据"。无数据本身是信号，不是错误。**绝不让 AI 假装查过某个源。**

### 3.4 三个源各自的实现要点

| 源 | 实现 | 风险/约束 |
|---|---|---|
| **日志** | 对接公司日志平台 API（已确认有可查询平台） | 低 |
| **代码** | 走 git 平台 API（GitLab/Gitea，待确认）。**部署版不能本地 clone**（多副本/临时容器/持久卷负担），主服务保持无状态。按线上部署的 commit/分支 ref 取文件 + blame + 最近 commit；读 JPA `@Entity`/`@Table` 或 MyBatis mapper 抽 entity→表映射。十几个 repo，靠日志给的类名/堆栈做精确取文件，不需要全文搜索 | 中 |
| **DB** | 代码驱动的只读 SELECT 验证。**护栏是死规定**：只读账号 + 只允许单 SELECT（AST/关键词校验，不靠 prompt）+ 自动 LIMIT + statement timeout + 生成的 SQL 原文进证据 timeline | 中，有安全护栏 |

## 4. 领域模型（长期资产的核心）

```
Investigation
├─ id, status(open / analyzing / confirmed / closed)
├─ symptom         症状原文 + 结构化字段（服务 / 时间窗 / 级别）
├─ hypotheses[]    假设 + 各自置信度
├─ evidence[]      强类型：{source, type, timestamp, raw, summary, relevance}
├─ conclusion
│   ├─ rootCause          根因
│   ├─ confidence         置信度
│   ├─ impact / blastRadius  影响面 / 爆炸半径
│   ├─ risk               风险
│   └─ suggestedActions[] 建议下一步
├─ humanVerdict    accept / reject / edited + 调查人批注   ← 人确认的结论才是主干
├─ timeline[]      每一步操作的审计日志（含执行过的 SQL 原文）
└─ createdBy, createdAt
```

`humanVerdict` 把"AI 说的"和"人确认的"分开存。日积月累，被人 accept 的高置信记录就是下一步做相似事故推荐 / 检索 / 微调的金标准数据集。

## 5. 模块划分（后端 package）

| 模块 | 职责 |
|---|---|
| `domain` | Investigation / Symptom / Evidence / Conclusion / Hypothesis 等值对象 |
| `connector` | Connector 接口 + 注册 + 配置；`builtin` 下 Log/Code/Db 三实现 |
| `pipeline` | 编排器：按事故跑连接器（含 code→DB 串行）+ 证据打包 + 降级 |
| `ai` | Reasoner + LlmClient（接内部 LLM，强约束 JSON）+ PromptBuilder |
| `record` | InvestigationRepository + 归档 + 历史检索 |
| `api` | REST 控制器 |
| 前端 | 列表页 + 详情页 |

## 6. 约束

- 公司内网**不能用 MCP**。
- LLM **走公司内部 API**（OpenAI 兼容），藏在 `LlmClient` 接口后面可换。

## 7. 待确认问题（先拍，否则 W2 返工）

- [ ] **#1 git 平台是 GitLab 还是 Gitea？** 决定 CodeConnector 的 API 实现。
- [ ] **#2 内部 LLM 是否支持 tool-use / function-calling？** MVP 不依赖（按确定性流水线做）；决定 v2 能否上 agent loop。
- [ ] **#3 code→DB 怎么编排？** 调查链里 DB 查询依赖代码分析的输出（串行）。如果编排器做成并行扇出会更简单，但 DB 拿不到代码的输出。二选一：(A) MVP 里 DB 退化成按症状/ID 的简单只读查、和代码并行；(B) 把 code→DB 做成串行子步骤。倾向 (B)，因为它才是真实 playbook。
- [ ] **#4 eval 口径。** 拿历史事故回放，看"方向对不对 / 真凶证据捞没捞出来 / 老手愿不愿从这接着干"，**不测准确率**。需确认这个口径、并准备 3-5 个已知根因的历史事故。
- [ ] **#5 单团队还是多团队？** MVP 先单团队跑通，还是一开始就要团队隔离 + 配置权限？
- [ ] **#6 部署形态。** 部署到哪（K8s / VPS / 内部 PaaS）？只读 DB 账号、git API token、日志平台凭证怎么下发？

## 8. 排期（4 周）

| 周 | 目标 | 关键产出 | 阻塞依赖 |
|---|---|---|---|
| **W1** | 走通骨架 | Investigation 领域模型 + 持久化 + 假连接器 + 编排器 + REST 端点 + 前端列表/详情，"提交症状→假证据→假分析→真记录→前端展示"端到端部署 | 无，立即开工 |
| **W2** | 三连接器 + AI | LogConnector / CodeConnector(git API) / DbConnector(只读护栏) + 证据打包 + Reasoner 接内部 LLM 出强约束 JSON | 需拍 #1 #2 #3 |
| **W3** | 资产层 | 人确认回路（accept/reject/edit）+ 调查记录归档 + 历史列表/检索/过滤 | 需拍 #5 |
| **W4** | 收口 | 前端把调查链/置信度/风险讲清楚 + 容错降级（部分证据如实标注）+ 历史事故回放 eval | 需拍 #4 + 历史事故素材 |

W1 不依赖任何待确认项，可立即开工。

## 9. 工作量预估

- W1 骨架：~3-4 人天
- W2 三连接器 + AI 定位：~6-7 人天
- W3 确认回路 + 归档 + 检索：~3-4 人天
- W4 前端收口 + 容错 + eval：~4-5 人天
- **不含**：需求对齐会议 / 系统测试 / 上线部署 / 凭证与权限基建 / 历史事故素材整理 / 培训

## 10. 范围外（本次不做，留 v2）

- 监控 / 链路追踪 / 发布变更 / 构建 等其它证据源
- agent tool-use 循环（待内部 LLM 能力确认）
- 代码全文/语义搜索索引（MVP 用 git API 精确取文件已够）
- 多团队配置自助、实时大盘、告警平台自动触发
