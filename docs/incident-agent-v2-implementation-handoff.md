# v2 实施交接包（给实现方 AI 的施工文档）

> 你是实现方。本文档是你的施工指令，配套阅读：[架构 v2](incident-agent-architecture-v2.md)（做什么、为什么）与 [v1 机制库](incident-agent-harness-optimization.md)（被引用的机制细节与参考实现）。
> 铁律：①在现有代码基础上增量改造，禁止另起炉灶重写；②一个工单一个 PR，每个 PR 跑全量回归；③不用假数据糊弄，工具查不到就如实返回查不到；④所有新工具遵循本文契约的字段名与语义，不自行发明；⑤本文所有新增工具均为只读（READ_ONLY 风险级），不引入任何写操作。

---

## 工单 0：代码库盘点与映射（必须先做，产出 `docs/v2-mapping.md`）

在写任何代码之前，通读现有代码库，产出映射表并等确认。按下面的清单逐行找到现有代码中的对应物（类/模块/配置），标注三种状态之一：`已有`（写明类名）/ `部分`（写明差什么）/ `缺失`：

| v2 组件 | 你要找的东西 |
|---------|-------------|
| 主循环 | agent 的迭代循环：LLM 调用、工具执行、终止条件在哪个类 |
| 工具注册 | 工具如何定义与注册（接口、schema 声明方式） |
| 日志连接器 | 查日志走什么（文件/平台 API），入口类 |
| 代码查询 | 现在的代码搜索实现（grep?），代码存放路径与更新方式 |
| DB 连接器 | 只读查询入口与防护（是否有只读护栏） |
| 结论结构 | 结论的 schema 类与校验逻辑（有无结构校验门） |
| 假设/笔记 | 是否有假设板、调查笔记类机制 |
| 事件/审计 | 调查过程是否有事件流/时间线记录 |
| 会话持久化 | 对话如何存储与恢复 |
| 平台审计日志 | **平台自身**的操作审计（谁在平台上做了什么）存在哪里、是否有查询接口——这是 M1 的命脉，没有就在映射表里显著标注 |
| 平台实体 | 租户/任务/策略/存储 的数据模型与 DB 表 |

映射表交付并确认后，先做工单 0.5，再按工单 1→5 顺序施工。

---

## 工单 0.5：回归基线与测试资产（验收的地基）

后续所有工单的验收依赖本工单产出。在 `cases/` 目录建三档回归集与黄金场景测试资产：

**回归集**：从真实历史工单脱敏挑选 6-8 例，按三档标注（信息充分 / 有锚点 / 无锚点），每例一个子目录：`ticket.md`（工单原文）+ `expected.md`（该档位的期望行为，充分档=根因，无锚点档=正确降级，不是每例都期望根因）。跑一遍现状 agent，记录基线表。

**G1 测试资产**（配合工单 1/3 验收）：`cases/g1-misop/` — 造一份平台审计记录 fixture（含某外部团队操作员在事故时间窗内修改策略/凭据的记录 + 无关噪声操作）、对应任务失败记录、tenants.yaml 中该团队的条目。格式与工单 0 盘点到的真实审计存储同构。

**G2 测试资产**（配合工单 2 验收）：`cases/g2-chain/` — 一个最小样例代码模块（真实项目里选一条现成的 API→Service→Mapper 链亦可,标注入口与期望链），加一条该链某处打出的日志行原文。

**验收**：基线表完成；G1/G2 资产能被人肉走通（人拿着资产能推出期望结论——人推不出的,agent 也不该被要求推出）。

---

## 工单 1（M1）：归属层 + 变更层

**目标**：外部关键词不再引向死路；误操作类事故一查一个准。

**1a. tenant registry**：新增配置文件 `config/tenants.yaml`（schema 见附录 A），加载为内存注册表。提供查询 API：`resolve(text) -> List<TeamMatch>`（按 alias 命中，返回 team + our_observables）。

**1b. audit_query 工具**（契约见附录 B-1）：查询平台审计日志。数据源用工单 0 盘点到的平台审计存储；没有现成审计则先接部署/配置变更记录，并在映射表标注审计缺口。

**1c. 实体解析进入口**：调查开始时（现有的信号蒸馏步骤处）对工单文本跑 `resolve()`，命中结果注入上下文，格式：
```
[entity-resolution] 命中接入方: oracle-dba (别名"Oracle RAC")
  可查证面: audit_query(team=oracle-dba), job_history(tenant=oracle-dba)
  注意: 该团队自身系统不可查——查它在平台上的行为痕迹，不要尝试查它的内部日志
```

**1d. prompt 增量**：system prompt 增加外部团队纪律："注册表中的接入团队,其内部系统一律不可查;调查其行为走平台侧痕迹(audit/job/api 记录);工单中的外部关键词是路由线索,不是调查终点。"

**验收**：黄金场景 G1（附录 D）通过——含外部团队关键词的工单，agent 首先查 audit/job 痕迹而非报"无法查询"；全量回归绿。

## 工单 2（M2）：代码层索引

**目标**：代码检索从"查询时 grep"变"入库时索引、查询时查表/走图"。

实施顺序（每步独立可交付）：
1. **日志语句索引**：扫描所有代码仓的日志调用（`log.xxx("...")` 等），提取格式串骨架 → `(骨架, 文件, 行号)` 表。现有的"日志反查代码"类工具（如有）底层改为先查此表，查不到再退回原实现。
2. **配置键索引**：配置项名 → 读取它的代码位置。新工具 `config_usage(key)`。
3. **调用图**：tree-sitter 或 JavaParser 解析方法调用边，必须桥接：接口→实现（`implements` + DI 注解扫描）、MyBatis 方法→XML statement、HTTP 客户端调用→URL、MQ 生产→消费。产出图存储（SQLite 两张表：nodes/edges 即可起步）。新工具 `trace_chain`（契约附录 B-2）。现有 stack_jump（如有）底层改走图。
4. **ORM 映射索引**：实体类↔表↔字段。`trace_chain` 走到 Repository/Mapper 层时在链节点上附注表名；新工具 `entity_mapping(class_or_table)`。

索引构建为独立命令（如 `--index`），代码更新后重跑；增量优化后做，先全量重建也可接受。

**验收**：黄金场景 G2 通过——从一条报错日志出发，两次工具调用内拿到完整调用链且链上标注涉及的表；索引构建命令幂等；回归绿。

## 工单 3（M3）：拓扑层 + ATTRIBUTION 出口

**3a.** 平台实体图：从工单 0 盘点到的平台数据模型建 `topology` 节点表（租户/任务/策略/存储，节点挂归属 team），跨边界调用在图中标 `boundary=true` 并关联 registry。

**3b.** 结论 schema 增加 `attribution` 字段（契约附录 B-3）；结构校验门增加规则：`attribution.verdict=theirs` 必须携带三件套证据（我方侧正常 + 对方平台痕迹 + 时间线对齐），缺任一则打回。

**3c.** 验证者（如已有对抗验证者）人设追加：核验 attribution 证据三件套的真实性。

**3d. prompt 增量**：submit_conclusion 的 schema 描述与 system prompt 增加 attribution 字段说明（四种判定的含义、theirs 的三件套证据要求）。

**验收**：黄金场景 G1 升级版——结论给出 `theirs(oracle-dba)` 且三件套齐；缺证据的 theirs 被门/验证者打回。

## 工单 4（M4）：假设可执行化

假设板的 `verification` 字段从自由文本升级为结构化（附录 B-4）：声明假设时给出可执行查询（工具名+参数）。harness 收到后**并行执行**这些查询（复用现有并行执行器），结果自动作为工具消息回填，并在假设板标注 `evidence_ref`。模型仍负责判定 CONFIRMED/KILLED——harness 只代跑查询，不代下判断。

**prompt 增量**：set_hypothesis 的 schema 与 system prompt 说明 verification 的结构化写法（工具名+参数+expect），并要求每个假设至少一条可执行验证。

**验收**：单测：含 2 个可执行验证的假设声明 → 两查询并行执行、结果回填；实测：调查轮次相比基线下降（记录数字）。

## 工单 5（M5）：记忆并层

历史调查记录增加实体关联（团队/服务/表），检索支持按实体查（"这个租户过去的事故"）。有 dream 固化机制的按 v1 §9.3 接入；没有则只做实体化检索。

---

## 附录 A：tenants.yaml schema

```yaml
teams:
  - team: oracle-dba              # 唯一 id
    display: 数据库组
    aliases: [Oracle, RAC, DB备份, 数据库组]   # 大小写不敏感匹配
    does_on_platform: [全量备份, 策略配置]
    our_observables:               # 每项是 工具名(固定参数) 模板
      - "audit_query(team=oracle-dba)"
      - "job_history(tenant=oracle-dba)"
    boundary_signals: []           # 可空: status page / oncall 频道 url
    oncall: "xxx"
```

## 附录 B：工具与 schema 契约

**B-1 audit_query**
```json
{"name":"audit_query","params":{"team":"可选","time_from":"ISO8601","time_to":"ISO8601",
 "action":"可选,如 delete/update","target":"可选,实体 id"},
 "returns":"行列表: {at, operator, team, action, target, detail}; 空结果返回 NoData 语义(该时间窗无操作记录=排除性证据)"}
```

**B-2 trace_chain**
```json
{"name":"trace_chain","params":{"entry":"方法全限定名 或 文件:行号","direction":"down|up","max_depth":10},
 "returns":"链节点数组: {order, class, method, file, line, kind: code|mapper|http|mq|boundary,
   table: 可空(mapper 节点), boundary_team: 可空(boundary 节点)};
   断链处如实标注 {kind:'broken', reason:'dynamic-dispatch|reflection|not-indexed'}，禁止假装连续"}
```

**B-3 结论 attribution 字段**
```json
{"attribution":{"verdict":"ours|theirs|shared|unknown",
  "team":"theirs/shared 时必填, registry 的 team id",
  "evidence":{"our_side_normal":["引用..."],"their_platform_traces":["引用..."],"timeline_alignment":"一句话+引用"},
  "handoff":{"to":"oncall","attach":["建议携带的证据引用"]}}}
```

**B-4 假设 verification 结构**
```json
{"id":"H1","hypothesis":"...","status":"TESTING",
 "verification":[{"tool":"audit_query","params":{...},"expect":"命中则支持/命中则否定"}]}
```

## 附录 E：索引存储 DDL 草案（SQLite，起步版）

```sql
-- 工单 2/3 共用；字段可增不可改名
CREATE TABLE graph_nodes (
  id TEXT PRIMARY KEY,            -- 方法全限定名 / 实体 id
  kind TEXT NOT NULL,             -- method|class|mapper_stmt|http_endpoint|mq_topic|table|tenant|job|policy|storage
  repo TEXT, file TEXT, line INTEGER,
  team TEXT,                      -- 归属(挂 registry), 可空
  boundary INTEGER DEFAULT 0      -- 1=边界节点
);
CREATE TABLE graph_edges (
  src TEXT NOT NULL, dst TEXT NOT NULL,
  kind TEXT NOT NULL,             -- call|implements|maps_to|reads_config|http_call|mq_produce|mq_consume|depends_on
  PRIMARY KEY (src, dst, kind)
);
CREATE TABLE log_statements (
  skeleton TEXT NOT NULL,         -- 通配变量后的格式串骨架
  repo TEXT, file TEXT, line INTEGER, level TEXT
);
CREATE TABLE orm_mappings (
  class TEXT NOT NULL, tbl TEXT NOT NULL,
  field TEXT, col TEXT, source TEXT   -- jpa|mybatis
);
CREATE TABLE config_usages (
  cfg_key TEXT NOT NULL, repo TEXT, file TEXT, line INTEGER
);
```

## 附录 C：施工纪律

- 每个 PR：说明对应工单号、改动点、回归结果（含三档 cases 的逐例结果）。
- 新工具描述必须写：何时用、怎么用、输出格式、查空含义（参照 v1 §4.1 五层规范）。
- 索引类代码全部确定性、可单测；LLM 只出现在推理循环与验证者。
- 遇到映射表之外的重大现状偏差：停下补映射表，不要绕过。

## 附录 D：黄金场景（验收基准）

**G1 误操作定责**：工单："Oracle 那边说凌晨的 RAC 全量备份全失败了，是不是你们平台挂了？" 期望轨迹：实体解析命中 oracle-dba → audit_query 发现 02:1x 该团队操作员修改了备份策略/凭据 → job_history 显示任务因凭据失败 → 我方调度与存储正常（证据）→ 结论 `attribution: theirs(oracle-dba)` + 三件套 + 转交建议。失败判据：报"无法定位"或试图查询对方内部系统。

**G2 链式追查**：从日志报错行出发：日志语句索引反查到打点代码 → trace_chain 拉出 API→Service→Mapper 完整链（含表名）→ 对该表跑只读验证 SQL → 结论因果链每跳有引用。失败判据：链在接口/实现或 Mapper 处断裂且未标注 broken。
