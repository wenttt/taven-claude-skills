# incident-agent 架构 v2：平台世界模型

> 定位：本文档是架构主文档，组织原则为**"模型为核、循环为翼"**——核心资产是一张预建的、持续更新的平台世界模型，调查循环是它的消费者。
> [v1 文档](incident-agent-harness-optimization.md) 降格为机制库：grok-build 源码摘录、循环监督机制、参数速查继续在那里维护，本文按节引用。
> 依据：2026-07 市场与学术调研（§6）+ incident-triage 实测反馈（验证者/分级出口/证据原语已落地验证）。

---

## 0. 范式：把"理解世界"从查询时挪到索引时

平台型事故排查的两个结构性难题——证据分布在接入方边界之外、因果链横跨代码/配置/任务/存储多层——共同的解法是**预先把整个平台建成一张可查询的模型**，调查时走图，而不是每次现场发明检索词。

行业收敛的证据：Traversal 的 Production World Model（遥测+代码+拓扑预压缩成 170 万节点因果图，Causal Search Engine 在图上并行测试上千假设）；Cleric 的自动服务映射（"隐式连接图，不只是官方依赖"）+ 并行假设测试；incident.io/Rootly/PagerDuty 共同的"统一上下文层"（服务目录+归属+依赖拓扑+变更历史，跨边界沿归属图路由）。学术侧：RepoGraph（ICLR 2025，仓库级代码图给 agent 带来 SWE-bench 32.8% 相对提升）、CodexGraph（NAACL 2025，agent 直接写图查询）。

平台团队事故响应的第一问不是"根因是什么"而是**"我方还是接入方"**——定责即根因。即便 $1B 估值的产品也不宣称能查进无权限的第三方系统；最佳实践就是：在自己的可观测边界内建全模型 + 边界处沿归属定责转交。

---

## 1. 分层架构

```
L5 产品面      聊天界面 · 审批 · 人工裁决 · 定责报告转交
L4 结论与验证  分级出口(+ATTRIBUTION) · 对抗验证者 · 证据纪律
L3 推理循环    实体解析契约 · 假设板(可执行验证,并行) · 瘦身版监督
L2 查询证据层  图查询工具 · 连接器(日志/DB/审计) · fallback 原语
L1 世界模型    归属层 · 变更层 · 拓扑层 · 代码层 · 记忆层   ← 核心资产
L0 采集接入    日志平台 · 平台审计 · 代码仓 · 配置中心 · 工单/告警
```

调查的标准路径：工单 → L3 实体解析（撞 L1 归属层）→ L2 标准查询扫荡（撞 L1 各层）→ 假设并行验证 → 链内走图、链断处（边界节点）定责 → L4 分级结论。

---

## 2. L1 世界模型：五层一张图

统一为可查询的图（起步可以是几张 SQLite 表 + 内存图，不需要图数据库），随源数据增量更新。

### 2.1 归属层（tenant registry）

平台每个接入团队/系统一条结构化记录：

```yaml
- team: oracle-dba
  aliases: [Oracle, RAC, 数据库组, DB备份]      # 工单/告警里会出现的写法
  does_on_platform: [全量备份, 策略配置]
  our_observables:                               # 我方侧可查的它的行为痕迹
    - audit_log(operator_team=oracle-dba)
    - job_history(tenant=oracle-dba)
    - api_access_log(caller=oracle-dba)
  boundary_signals: [status_page_url, oncall_channel]   # 低成本外部信号，逐个谈
  oncall: xxx
```

作用：①工单实体识别的词典——外部关键词命中 alias 后路由到 our_observables，而不是触发查不到的死路；②边界节点的定责档案；③事故频率统计的挂靠点（哪个接入方最常出事，数据驱动去谈 boundary_signals）。

### 2.2 变更层

平台审计日志（谁、何时、在平台上做了什么操作）+ 部署记录 + 配置变更。多租户平台的 CloudTrail：**接入方误操作发生在平台上，痕迹天然在我方**——这一层让"其他团队误操作"类事故从查不到变成一查一个准。工具：`audit_query(team/time/action)`、`deploy_recent`、`config_changes`。

### 2.3 拓扑层

平台自己的实体图：租户 → 备份任务 → 策略 → 存储目标 → 底层服务，加服务间依赖边。节点带归属（挂 2.1）。跨边界的边显式标注为**边界节点**——链式追查走到这里自动切换定责流程。

### 2.4 代码层（四索引）

代码在本地服务器，入库时解析（tree-sitter / JavaParser），增量重建：

1. **调用图**：方法→方法边，桥接 Java 断链点——`implements`/`@Service` 的接口→实现边、MyBatis 方法→XML statement 边、`@FeignClient`/HTTP URL→跨服务边（连到拓扑层）、MQ producer→topic→consumer 边。`trace_chain(入口)` 即图遍历，一次返回整条链。
2. **日志语句索引**：所有 `log.xxx("format string")` → 代码位置。日志行反查代码从运行时 grep 变 O(1) 查表，且可反向（代码位置→会打什么日志→去日志平台搜什么）。
3. **ORM 映射索引**：`@Table/@Column`/MyBatis resultMap → 类↔表↔字段。链走到 Repository 层自动接"操作哪张表"，验证 SQL 变填空。
4. **配置键索引**：配置项 → 读取它的代码位置（变更类根因直达）。

接口范式参照 CodexGraph：工具暴露结构化图查询，agent 负责判断，不负责发明检索词。运行时有 trace id 时，OpenTelemetry 轨迹是"这次实际怎么走"的第一证据，静态图管"代码上可能怎么走"，互补。

### 2.5 记忆层

历史调查（按 verdict 分级：ACCEPTED 常青）+ dream 固化的事故模式手册（v1 §9），实体与拓扑层节点关联——"这个租户/这张表上次出过什么事"成为一跳查询。

---

## 3. L3 推理循环（瘦身版）

- **契约 = 实体解析**：incident.md 的字段解析为世界模型实体引用（租户节点/服务节点/时间窗），alias 命中即路由；解析不到的关键词才标 unknown。自取漏斗规则沿用 v1 §3.1。
- **扫荡 = 标准查询组**：对已解析实体并行执行固定查询组（该实体的审计变更/任务记录/关联日志/历史事故），零模型调用；结果构成证据包底座。
- **假设板 = 可执行验证**：每个假设的 verification 字段是**可执行查询**（图查询/日志查询/SQL 模板），harness 并行执行、结果自动回填假设状态（TESTING→CONFIRMED/KILLED）。参照 Traversal Causal Search Engine 的"假设在模型上并行测试"。
- **监督机制最小集**：纪律三条 + 无工具调用门禁 + 撞顶强制出结论（v1 §2）。证据来自模型查询后循环显著变短，重型防御（停滞分类器/doom loop）按症状再引入。

## 4. L4 结论与验证

- 分级出口增加 **ATTRIBUTION 维度**：`ours / theirs(team X) / shared / unknown`，与 CONFIRMED/HYPOTHESES/ELIMINATION/BLOCKED 正交。判定 theirs 必须携带边界证据三件套（我方侧行为正常 + 对方在平台上的操作/任务异常痕迹 + 时间线对齐），由验证者核验（v1 §5.2 人设追加一条规则）。
- **定责报告是一等交付物**：判定 + 我方证据链 + 对方在平台上的痕迹 + 转交对象（归属层 oncall）+ 建议携带的材料。产品价值 = 替平台团队完成"证明不是我们 + 该找谁 + 证据打包"。
- 对抗验证者原样沿用（已实测：能抓引用不实并逼补查）。

## 5. 迁移路径（代码持续可用，逐层换心）

| 阶段 | 内容 | 现有代码的变化 |
|------|------|---------------|
| M1 归属+变更层 | tenant registry(YAML 起步) + audit_query 工具 + 实体解析进 STEP 0 | 新增工具与注册表，循环不动 |
| M2 代码层索引 | 日志语句/配置键索引（最简单先做）→ 调用图 + trace_chain → ORM 映射 | code_search 降为 fallback，stack_jump 改走图 |
| M3 拓扑层 | 平台实体图 + 边界节点 + ATTRIBUTION 出口 | 结论 schema 加 attribution 字段 |
| M4 假设可执行化 | verification 字段结构化为查询，并行执行回填 | 假设板升级，复用现有并行执行器 |
| M5 记忆并层 | 历史调查挂实体，dream 固化 | past_investigations 检索加实体维度 |

每阶段跑三档回归集（cases/），M1 结束即可用真实工单验证定责路径——这是当前事故分布的最大头。

## 6. 调研依据

- [Traversal Production World Model](https://www.traversal.com/blog/introducing-production-world-model-ai-readable-model-your-entire-production-environment) · [Causal Search Engine](https://traversal.com/blog/introducing-causal-search-engine-from-correlated-alerts-to-causally-consistent-diagnoses) — 预建因果图 + 图上并行假设
- [Cleric Product](https://cleric.ai/product) · [Hidden Complexity of AI SRE](https://cleric.ai/blog/the-hidden-complexity-of-building-an-ai-sre) — 自动服务映射、并行假设、运营记忆
- [Cleric vs Resolve.ai vs Traversal](https://wetheflywheel.com/en/guides/cleric-vs-resolve-ai-vs-traversal/) — 三家对比与公开指标
- [incident.io AI SRE Guide](https://incident.io/blog/what-is-ai-sre-complete-guide-2026) · [Rootly AI SRE Lifecycle](https://rootly.com/ai-sre-guide/lifecycle) · [PagerDuty 依赖映射](https://www.pagerduty.com/resources/incident-management-response/learn/dependency-mapping-faster-incident-resolution/) — 统一上下文层与归属路由
- [RepoGraph (ICLR 2025)](https://arxiv.org/html/2410.14684v1) · [CodexGraph (NAACL 2025)](https://aclanthology.org/2025.naacl-long.7/) — 代码图对 agent 的效应量与查询接口范式

## 附：v1 机制在 v2 中的位置

| v1 机制 | v2 归宿 |
|---------|--------|
| 对抗验证者 / 分级出口 / 纪律三条 / EventStream | L4/L3 原样沿用（实测验证） |
| 锚点提取 / incident.md 契约 | L3 实体解析（解析目标从文本变模型实体） |
| 假设板 | L3 可执行验证版 |
| stack_jump | L2，底层改走调用图 |
| 骨架反查 / 查空降阶链 | L2 fallback（索引未覆盖的死角） |
| 边界五问 / OutOfScope | 收编入拓扑层边界节点 + ATTRIBUTION 流程 |
| 停滞分类器 / doom loop / 压缩 / 子代理 | 延后菜单不变（v1 §14） |
| 会话生命周期 / 插话四路径 / 记忆 dream | L5/L1 沿用（v1 §8 §9） |
