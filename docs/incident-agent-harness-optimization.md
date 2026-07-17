# incident-locator Agent Harness 优化设计

> 依据：xai-org/grok-build（Apache 2.0，2026-07 开源）全量源码研读。
> 目标：解决 incident-locator 当前的四类问题——不主动调用工具、不理解/遗忘用户信息、结论不准、非理想路径（查不到 / 证据模糊 / 信息不够）处理粗糙。
> 核心设计思想：**agent 的可靠性来自 harness 层的监督机制。grok-build 的基座 system prompt 仅 45 行，可靠性由主循环防守、契约注入、工具层设计、对抗验证、失败状态机五个子系统提供。**

---

## 1. 问题与机制映射

| 现象 | 对应机制 | 章节 |
|------|---------|------|
| 模型叙述"我要查日志"但不调工具 | 无工具调用门禁 + 行为纪律 | §2.1 §2.2 |
| 干一半请示"要继续吗"、提前收工 | 行为纪律 + 续行指令 | §2.2 §2.3 |
| 多轮后遗忘事故时间窗/服务名 | 事故契约 + 每轮回注 + 分层裁剪 | §3 |
| 长调查上下文爆炸、压缩后丢关键信息 | 结构化压缩摘要 + 预压缩 + 降级梯 | §3.3 |
| 工具调用了但不会用、不知道下一步 | 工具层五层设计 + 结果内引导 | §4 |
| 查空后不知所措或直接放弃 | 查空三分类 + 放弃门槛 | §4.2 §6.2 |
| 结论没证据、声称查了实际没查 | 证据纪律 + 对抗验证panel | §5 |
| 查不到就编根因 | 分级降级出口 | §6.1 |
| 反复查同一个地方原地打转 | gap fingerprint 停滞检测 | §6.3 |
| 模型输出陷入复读死循环 | doom loop 检测 | §7.2 |
| 中途插话丢失或污染工具结果 | 三安全点插话机制 | §8.1 |

---

## 2. 主循环监督

### 2.1 无工具调用门禁

**是什么**：主循环里，模型返回纯文本且调查尚未产出结论时，先过一道门再决定是否结束 turn。

**怎么实现**（对应 `Agent` 主循环，grok-build turn.rs:2115 的 TodoGate 模式）：

```java
int gateFires = 0;
while (running) {
    LlmResponse resp = llm.chat(context);
    if (resp.toolCalls().isEmpty()) {
        if (!conclusionPublished && gateFires < 2) {
            gateFires++;
            context.injectSystem("""
                <system-reminder>调查尚未产出结论。当前假设板：%s。
                继续调用工具推进，不要请示用户。</system-reminder>
                """.formatted(hypothesisBoard.summary()));
            continue;   // 强制重新采样
        }
        return resp;    // 放行：已有结论，或门禁达上限
    }
    executeToolCalls(resp.toolCalls());
}
```

要点：
- **每个用户 prompt 最多拦 2 次**（grok-build `max_fires_per_prompt=2`），超过放行——门禁是纠偏，上限防死循环。
- 判据用 harness 可查的事实（结论未发布、假设板有未验证项），与 ConclusionGate 的"验收"分工：门禁管"别停"，验收门管"停得对不对"。
- 每次触发记 `Steered` 事件进 EventStream，可审计。

**grok-build 原实现**（`turn.rs:2112` 无工具调用路径，节选）：

```rust
if tool_calls.is_empty() {
    if !schema_ok && !turn_refused && let Some(gate_cfg) = self.todo_gate_policy() {
        let collected = self.collect_todo_gate_input(req_id).await;
        if let TodoGateDecision::Nudge { reminder, reason } =
            evaluate_todo_gate(&collected.as_input())
        {
            if todo_gate_fires < gate_cfg.max_fires_per_prompt {
                todo_gate_fires += 1;
                self.events.emit(Event::TodoGateFired { fires: todo_gate_fires, .. });
                self.push_system_reminder(&rendered);
                continue;                                  // 强制重新采样
            }
            // 达到上限：如实告知后放行，不硬扣
            self.events.emit(Event::TodoGateExhausted { pending: input.pending.len() });
            self.push_system_reminder(&format!(
                "The agent attempted to end this turn {cap} times with todos still \
                 pending or in_progress. Falling through to user. If you want \
                 autonomous progress, prompt the agent to continue explicitly, \
                 or clean up the todo list."));
        }
    }
    if self.drain_pending_interjections().await {          // 收尾前最后 drain 一次插话
        continue;
    }
    return Ok(TurnOutcome::Completed { .. });
}
```

门禁判定是纯函数，上限检查留在调用方——判定逻辑可独立单测（`acp_session_impl/reminders.rs:88`）：

```rust
pub fn evaluate_todo_gate(input: &TodoGateInput<'_>) -> TodoGateDecision {
    if input.pending.is_empty() && input.in_progress_unbacked.is_empty() {
        return TodoGateDecision::Continue;
    }
    TodoGateDecision::Nudge {
        reminder: build_todo_gate_reminder(&input.pending, &input.in_progress_unbacked),
        reason: TodoGateReason::InFlight,
    }
}
```

nudge 文本内置合法出口（`build_todo_gate_reminder`）——不是单纯逼模型干活，真实阻塞时给出体面的退出路径：

```
You have outstanding todos but ended your turn without a tool call.
Pending:
- {待办列表}
Per <task_completion_discipline>, advance the next pending todo with the
appropriate tool call NOW. If you have a genuine external blocker (missing
credential, denied permission, network unreachable), state it explicitly
AND mark the affected todos `cancelled` with a reason in the same turn.
```

### 2.2 行为纪律（写入 system prompt）

grok-build `goal_task_discipline.md` 的三条规则，按排查场景改写后加入 system prompt：

1. **先调用，后叙述。** 任何"我正在查/我已检索"式的叙述，同一条回复里必须伴随对应的工具调用。只有叙述没有调用 = 动作没有发生。
2. **明确的下一步直接执行。** 向用户提问只用于会改变排查方向的真歧义（缺时间窗、缺服务名、需要授权）。下一步已由假设板或证据指明时直接做。
3. **收尾前自查。** 结束前检查：假设板上是否还有未验证且可验证的假设？有就继续。合法的停止只有三种——等用户对真歧义做决定、等一个需授权的工具审批、遇到硬性阻塞（数据源未接入 / 数据已滚动 / 缺权限），且必须明说阻塞是什么。

### 2.3 续行指令（每轮回注调查状态）

**是什么**：每次采样前，harness 注入一个固定的状态头，让目标和下一步永远在最新上下文里（grok-build `goal_continuation_directive.md` 模式）。

```
<investigation-state>
事故: {incident.md 的一行摘要：时间窗 / 服务 / 症状}
假设板: {当前假设 + 各自状态 verified/refuted/pending}
预算: 第 {n}/{max} 轮 | tokens {used}/{budget}
</investigation-state>
```

与现有 Notebook（钉 system、压缩不掉）合并实现：Notebook 已承载 findings，本设计将其扩展为 findings + 契约摘要 + 假设板状态三段。

### 2.4 停滞分类器（进阶，Phase 3）

**是什么**：turn 结束后会话空闲时，用一次便宜的 LLM 调用判定 agent 是否 STALLED（grok-build `laziness_classifier.rs`）。

关键设计——给分类器喂**模型无法伪造的 harness 事实**：

```
[runtime_state] pending_approvals=0 unverified_hypotheses=2 turn_elapsed_seconds=45
```

分类器读最近 30 条消息 + runtime_state，输出 `{stalled: bool, confidence: float}`。判定为 stalled 的情形：叙述了动作但无对应工具调用、向用户请示显然的下一步、声称完成但断言无工具调用证据支撑。`confidence >= 0.7` 才触发 nudge。输出上限 150 token，成本可忽略。

**grok-build 原实现**（`laziness_classifier.rs:291`，真值行由 harness 生成，前置在转录之前）：

```rust
pub(crate) fn format_runtime_state_line(...) -> String {
    format!(
        "[runtime_state] outstanding_background_tasks_and_subagents={backing_task_count} \
         turn_elapsed_seconds={secs}\n"
    )
}
```

分类器 prompt 教它用真值行做交叉核验（`laziness_classifier.rs:180`）：

```
- Claim "ran 8+ hours overnight" → cross-check against
  `[runtime_state] turn_elapsed_seconds=N` when present;
  minutes-of-work cannot have produced multi-hour deliverables.
- If outstanding_background_tasks_and_subagents = 0 AND the last assistant
  message claims to have launched a subagent or backgrounded work, that is
  strong evidence for `stalled_narration` regardless of the prose.
```

---

## 3. 事故契约与上下文管理

### 3.1 事故契约（incident.md）

**是什么**：调查启动时，把用户输入（工单原文 / 告警 / 口头描述）蒸馏成结构化契约，作为整个调查的单一事实来源——实现者、验证者、续行指令都以它为准（grok-build goal planner 的 plan.md 模式，与现有 STEP 0 DISTILL 衔接）。

```markdown
# Incident: <一句话>
## 事实（必填）
- 时间窗: <绝对时间，含时区>
- 服务/系统: <名称，多个则列出调用关系>
- 症状: <观测到的现象原文>
## 事实（尽力）
- 影响面 / 关联变更(部署·配置) / 相关 id(trace/order/user)
## 验收标准
- 结论中每条因果断言有证据引用（日志行 / file:line / SQL+结果）
- 时间窗内的关键日志已检索（含"无匹配"这一结果本身）
- 若涉及数据，已用只读 SQL 验证
## 排查计划
- [ ] <首个具体步骤>
```

**入口 gate**：必填字段缺失时，一次性把缺的问齐再开工。排查中途出现会改变方向的新信息缺口（如"需确认当时是否发版"），允许中断询问——这属于 §2.2 定义的真歧义。

**验收标准供 §5 的验证者引用**：验证者按契约里的标准审，与实现者对齐同一把尺子（grok-build "shared verification plan" 设计——两个角色遵循同一可观测标准，判定才能收敛）。

### 3.2 分层裁剪

上下文超过窗口 50% 开始修剪（grok-build `request_builder.rs`），按保护级别：

| 层 | 规则 | 参数 |
|----|------|------|
| 1 | 用户消息永不裁剪 | — |
| 2 | 最近 N 轮完整保留 | N=3 |
| 3 | 老的大工具结果软裁：保头 1500 字符 + 尾 1500 字符，中间标 `[…trimmed…]` | 触发阈值 4000 字符 |
| 4 | 超龄工具结果硬清为占位符 `[Tool result omitted — too old]` | 10 轮 |

日志检索这类大结果的头尾恰好信息量最大（首条命中 + 错误栈收尾），软裁天然适配。已进 Notebook 的 findings 不受影响（钉 system）。

**grok-build 原实现**（`xai-chat-state/src/actor/request_builder.rs:157`）——从后向前数 User 消息定轮龄，只动 ToolResult：

```rust
pub(crate) fn should_prune(total_tokens: u64, context_window: NonZeroU64) -> bool {
    total_tokens > context_window.get() / 2
}

pub(crate) fn prune_conversation(conversation: &mut [ConversationItem], config: &PruningConfig) {
    let mut turn_from_end: usize = 0;
    let mut seen_first_user = false;
    for i in (0..conversation.len()).rev() {
        if matches!(&conversation[i], ConversationItem::User(_)) {
            if seen_first_user { turn_from_end += 1; }
            seen_first_user = true;
            continue;                                       // User 消息永不修剪
        }
        let ConversationItem::ToolResult(tool_result) = &mut conversation[i] else {
            continue;                                       // Assistant/System 不修剪
        };
        if turn_from_end < config.keep_last_n_turns {       // 最近 N 轮完整保留
            continue;
        }
        if turn_from_end >= config.hard_clear_age_turns {   // 超龄硬清为占位符
            tool_result.content = Arc::<str>::from(HARD_CLEAR_PLACEHOLDER);
            continue;
        }
        let content_len = tool_result.content.chars().count();
        if content_len > config.soft_trim_threshold {       // 大结果软裁：保头+尾
            let head = safe_char_slice(&tool_result.content, 0, config.soft_trim_head);
            let tail = safe_char_slice_tail(&tool_result.content, config.soft_trim_tail);
            tool_result.content =
                Arc::<str>::from(format!("{head}{SOFT_TRIM_SEPARATOR}{tail}"));
        }
    }
}
```

### 3.3 压缩摘要（升级现有 LlmCompactor）

**摘要 prompt 采用分节强制结构**（grok-build 9 段式，按排查场景裁成 7 段；每段必须出现，空则写 None）：

```
把到目前为止的调查压缩成摘要，后继者只能看到 incident.md 和这份摘要。
在 <summary> 块内输出以下编号小节：
1. 事故与用户诉求：用户的原始描述与所有后续补充，保留约束和范围
2. 已确认事实：每条附证据引用（日志行号 / file:line / SQL+结果）
3. 假设板现状：每个假设的状态（已证实/已排除/待验证）及依据
4. 已排除方向：排除了什么、凭什么证据排除
5. 错误与修正：查错方向的经过、如何纠正
6. 用户全部消息：按序列出（不含本压缩指令）
7. 当前工作与下一步：压缩前正在做什么，下一步是什么（引用最近消息原句防漂移）
经济性优先：紧凑散文 + 短引用，禁止长篇原文转录。禁止调用任何工具。
```

关键条款（grok-build 原设计）：**如果早前轮次里已有上一次的压缩摘要，视其为早期历史的权威版本，把仍然相关的信息结转进新摘要**——防止多次压缩逐层丢失。原文（`session_compact.rs`，9 段式标准 prompt 内）：

```
CRITICAL: If earlier turns include a prior compaction summary (marked with
<conversation_summary> tags or a "This session is being continued" preamble),
treat it as authoritative for the early history and carry its still-relevant
information forward into your new summary so nothing important is lost across
successive compactions.
```

另两条同样照抄的原文条款：摘要经济性（"A focused summary that fits is far more
useful than an exhaustive one that gets cut off, so aim for at most a few
thousand words"）和末段防漂移锚（第 9 段 Optional Next Step 要求 "include a
direct verbatim quote from the most recent messages showing exactly what you
were doing and where you left off, so the task is interpreted without drift"）。

**两级预压缩**（吞吐优化，Phase 3）：在阈值前 10 个百分点（如 80% 触发则 70% 时）后台先对前 95% 历史做 pass1 摘要缓存；到阈值时只需增量合并最后 5%，避免压缩阻塞当轮响应。

**压缩失败降级梯**（grok-build 三级）：完整压缩 → lossy 预处理（先删冗长工具输出再压）→ 直接截断到 70% 窗口。每级失败自动落下一级，调查不因压缩失败中断。

**压缩后恢复**：压缩完成后把 incident.md 与 Notebook 重新注入（grok-build 压缩后重载 AGENTS.md / 重建任务列表的对应物）。

---

## 4. 工具层

### 4.1 五层设计

| 层 | 做法 | incident-locator 落地 |
|----|------|----------------------|
| 工具描述 | 用法 + 避坑 + 输出格式三件套 | log_search 描述里写明："输出 `{ts}→{level}→{msg}`；命中超限时报 at least N；时间参数用 ISO8601 含时区" |
| 参数设计 | 边界值、默认值、"何时才需要提供" | "limit 默认 100（上限 500）。仅当首查结果超限时才提供 time_range 收窄" |
| 强制说理由 | 关键工具加必填 `reason` 参数 | db_query、log_search 加"一句话说明这次查询要验证哪个假设"——逼模型先想后查，且 reason 直接进 timeline 供人审 |
| 类型化错误 | 错误是枚举不是笼统 string | `QuerySyntaxError`（附正确示例）/ `SourceUnavailable` / `QueryTimeout`（附"可能全表扫描，加时间条件"）/ `PermissionDenied` |
| 结果内引导 | 结果尾部注入 `<system-reminder>` 指路 | 见 4.3 |

行号/字段分隔符用 `→`（不出现在日志和代码里，无解析歧义）。

### 4.2 查空三分类

工具返回空必须区分三种情况，每种都是信息：

| 结果 | 含义 | 返回格式 |
|------|------|---------|
| `NoData` | 查询成功但范围内确实无数据 | "该时间窗 [t1,t2] 内无匹配 ERROR 日志" + reminder："这是排除性证据，记入证据链；如怀疑范围不对，扩窗或换服务名" |
| `QueryError` | 查询本身失败 | 类型化错误 + 修正建议，明确"此结果非证据" |
| `Truncated` | 结果超限 | 头部预览 + "at least N 条命中" + 收窄建议 |

`NoData` 是排查场景的一等证据："时间窗内该服务无异常日志"直接支撑"问题不在这一层"的推理。工具层把这个语义显式说出来，模型才会用。

### 4.3 结果内引导

工具结果尾部按情境注入 `<system-reminder>`（grok-build `reminders/` 模块模式）：

- 结果超限 → "命中 at least 32000 行（已截断至前 100）。用 time_range 收窄或加 level=ERROR 过滤后重查。"
- 检索到异常栈 → "栈中出现 `{类名}`，可用 find_definition / find_callers 定位对应代码。"
- db_schema 之后 → "查询数据请用只读 SELECT，带 LIMIT。"
- 后台/长任务完成 → 完成通知直接出现在下一轮上下文，"无需轮询等待"写进工具描述。

**grok-build 原实现**（`xai-grok-tools/src/reminders/mod.rs:40`）：

```rust
pub fn wrap_reminder_with_tag(text: &str, tag: &str) -> String {
    format!("<{tag}>\n{text}\n</{tag}>")
}

pub fn format_with_reminders(output: String, reminders: Vec<String>, tag: &str) -> String {
    if reminders.is_empty() {
        return output;
    }
    let wrapped: Vec<String> = reminders.iter()
        .map(|r| wrap_reminder_with_tag(r, tag))
        .collect();
    let joined = wrapped.join("\n\n");
    if output.is_empty() { joined } else { format!("{output}\n\n{joined}") }
}
```

### 4.4 截断策略三式

| 方式 | 行为 | 适用 |
|------|------|------|
| 硬截断 | 每行超限丢弃 + `[truncated (N chars total)]` | 日志超长单行（模型无需看完整行） |
| 软换行 | 保留全部内容按宽度重排 | 命令输出（需要完整但要结构化） |
| 预览+尾注 | 头部预览 + `[Output truncated — N bytes total. {hint}]` | 大日志批次（重要但篇幅大） |

**grok-build 原实现**（`xai-grok-tools/src/util/truncate.rs:100`，`footer_hint` 就是 §4.3 结果内引导的注入点）：

```rust
pub fn truncate_with_preview(
    output: &str,
    max_bytes: usize,
    preview_bytes: usize,
    footer_hint: Option<&str>,
) -> (String, bool) {
    if output.len() <= max_bytes {
        return (output.to_string(), false);
    }
    let preview = truncate_str(output, preview_bytes.min(output.len()));
    let footer = match footer_hint {
        Some(hint) => format!("[Output truncated - {} bytes total. {hint}]", output.len()),
        None => format!("[Output truncated - {} bytes total]", output.len()),
    };
    (format!("{preview}\n\n{footer}"), true)
}
```

---

## 5. 结论质量：证据纪律与对抗验证

### 5.1 证据纪律（写入 system prompt + ConclusionGate 硬校验）

- 结论中每条因果断言必须附证据引用：日志行（source+行号或时间戳）、代码位置（file:line）、SQL+结果摘要。
- 引用必须可溯源到本次调查的 timeline 中真实发生过的工具调用——**声称查过但 timeline 无此调用 = 编造，验收门打回**（grok-build honesty check：声称改了 CHANGED_FILES 里没有的文件即 refute 的对应物；与现有引用溯源校验一致，本设计补上"断言→工具调用"的反向核对）。
- 叙述不是证据：模型自己的总结文字只用于定位它做了哪些 claim。

### 5.2 对抗验证 panel（升级现有 ConclusionGate 为双阶段）

**阶段一（现有，保留）**：ConclusionGate 结构校验——引用溯源、运行时覆盖、假设必须声明、修订声明（REVISED/REAFFIRMED）。零 LLM 成本，先挡格式与诚实性问题。

**阶段二（新增）**：结构校验通过后，spawn 对抗验证者复核实质内容。验证者 prompt 要点（改编自 `goal_verifier_prompt.md`）：

```
你是对抗验证者，不是产出这份结论的 agent。你的工作是反驳这份排查结论成立。
不确定时默认 refuted——错误的根因放行会误导止损动作，比多一轮复查糟糕得多。

输入：incident.md（含验收标准）/ 结论 JSON / 调查 timeline（工具调用与结果）/ PRIOR_GAPS（上轮指出的缺口）

审计而非重查：以 timeline 里已有的证据为主要材料判断每条断言是否被支撑；
只做廉价抽查（复读关键日志行/代码位置）。证据缺失时 refute 并明确要求
实现者补什么，而不是自己去补。

反棘轮：复验轮的首要工作是核对上轮缺口是否真正修复。标准不随轮次上升；
新的反对意见只在"结论存在可论证的实质缺陷"时成立，风格偏好不构成 refute。
所有上轮缺口已修复且验收标准成立时，返回 Not Refuted。

排查特化审查镜头：
- 因果链完整性：从症状到根因每一跳有证据，跳步即 refute
- 时间线一致性：因发生在果之前；部署/变更时间与异常起点对得上
- 反例检查：结论能解释全部症状；有反例日志未解释即 refute
- 替代解释：存在未排除的同等合理替代根因时，要求降级为"假设排序"输出
- 置信度诚实：LOG_CONFIRMED/DB_VERIFIED/CODE_INFERRED 标签与实际证据强度一致

输出 JSON：{ refuted, findings[{kind, location, detail}], evidence, confidence,
blocking: none|contradiction|unverifiable }
```

**panel 与投票（Phase 3 可选）**：单验证者起步；升级为 3 员 panel 时采用 grok-build 冷面板规则——skeptic 0 跨轮复用（带着上轮记忆核对缺口），skeptic 1..2 每轮新起；**通过需冷面板（skeptic 1..2）绝对多数投 Not Refuted**，skeptic 0 只有否决权没有通过权。平票判 refuted。

**grok-build 原实现**（`goal_classifier.rs:1009`）。两个容错细节值得照抄：单个 skeptic 崩溃/超时/输出畸形，在上游降级为一张合成的 `refuted: true` 票（fail-closed）；空面板返回不通过：

```rust
pub(crate) fn aggregate_skeptic_verdicts(results: &[SkepticResult]) -> (u32, u32, bool) {
    let total = results.len() as u32;
    if total == 0 {
        return (0, 0, false);                       // 空面板 = 不通过
    }
    let refuted_count = results.iter().filter(|r| r.refuted).count() as u32;
    let (needed, not_refuted) = if total <= 1 {
        (1, total - refuted_count)                  // 独任裁判：一票定
    } else {
        // Variant-C：冷面板(skeptic_idx >= 1)严格多数；skeptic 0 不计通过票
        let cold_count = results.iter().filter(|r| r.skeptic_idx >= 1).count() as u32;
        let cold_not_refuted = results.iter()
            .filter(|r| r.skeptic_idx >= 1 && !r.refuted)
            .count() as u32;
        (cold_count / 2 + 1, cold_not_refuted)
    };
    (refuted_count, total, not_refuted >= needed)
}
```

**验证轮预算**：refute 后带 findings 打回实现者，最多 2 轮（沿用 ConclusionGate 打回封顶 2 次）；仍不过则按 §6 降级输出。

### 5.3 blocking 分类联动失败路由

验证者的 `blocking` 字段决定打回后的路径：`none` → 正常打回迭代；`contradiction`（事故描述自相矛盾）/ `unverifiable`（证据在本环境不可得）→ 直接走 §6.4 升级用户，节省无效重试轮。

---

## 6. 失败状态机与降级模型

### 6.1 分级结论出口

调查的合法产出分四级，每级都是完整交付物（对应现有结论三轨制的扩展）：

| 级 | 产出 | 判据 |
|----|------|------|
| CONFIRMED | 根因确认 | 因果链每跳有证据，验证者通过 |
| HYPOTHESES | 假设排序 | Top N 假设，各附支持证据/反对证据/区分所需数据 |
| ELIMINATION | 排除报告 | 已排除方向+证据，剩余空间指向，当前数据不足 |
| BLOCKED | 阻塞报告 | 明确缺什么：数据源未接入/数据已滚动/缺权限 |

及格线：**每句话有据可查，说清知道什么、不知道什么、还差什么。** 每一级对一线排障都有直接价值（排除报告本身就能省人的时间）。

### 6.2 放弃门槛（3 次规则）

模型声明"查不到/卡住"时，harness 前两次拒绝接受（grok-build `goal.rs:215`）：

```java
int streak = blockedStreak.incrementAndGet();
if (streak < 3) {
    return toolResult("""
        Blocked attempt %d/3 recorded. 连续 3 次才转入 BLOCKED——
        换个角度继续：扩大时间窗 / 换关键词或服务名 / 查上游调用方 /
        对照近期部署与配置变更。""".formatted(streak));
}
transitionTo(State.BLOCKED, reason);
```

**任何一次实质进展（新证据入链、假设状态变化）把计数清零。** 双向作用：挡轻易放弃；同时给模型一个合法的"查不出来"出口——有体面出口，编造率下降。

**grok-build 原实现**（`acp_session_impl/goal.rs:213`）——注意前两次的回执是"已记录"而不是"拒绝"，模型的声明被承认但不生效：

```rust
if let Some(reason) = cmd.blocked_reason {
    let streak = self.goal_blocked_streak
        .fetch_add(1, Ordering::Relaxed) + 1;
    if streak < 3 {
        try_send_ack(ack_tx, UpdateGoalAck::Accepted {
            summary: format!(
                "Blocked attempt {streak}/3 recorded. The harness needs \
                 3 consecutive blocked attempts before pausing the goal — \
                 continue retrying or refining the approach."),
        });
        continue;
    }
    // 第 3 次：真正转入 Blocked，暂停并通知用户
    self.auto_pause_goal_if_active_with_message(
        GoalPauseReason::Verification, pause_msg).await;
}
// ……completed:true 的处理路径上（同文件下方）：实质进展清零计数
self.goal_blocked_streak.store(0, Ordering::Relaxed);
```

### 6.3 停滞检测（gap fingerprint）

每轮结束把"未验证假设 + 待补证据"列表归一化成指纹：提取其中的 `path:line`、日志源+时间窗、假设 id 等 token，临时路径归一化为 `<scratch>`，排序去重后连接。**连续 3 轮指纹完全相同 → 自动转 NO_PROGRESS，输出当前最优的分级结论**（§6.1 的第 2/3 级）。区分"在推进"和"在空转"，空转立即止损。

**grok-build 原实现**（`goal_classifier.rs:1203`）。归一化临时路径是关键——scratch 路径带每轮随机 id，不归一化则同一缺口每轮指纹都不同，停滞检测永远不触发：

```rust
pub(crate) fn gap_fingerprint(raw_evidence: &[&str]) -> String {
    let normalized: Vec<Cow<'_, str>> = raw_evidence
        .iter()
        .map(|e| normalize_scratch_paths(e))     // /tmp/... → <scratch>
        .collect();
    let mut tokens: Vec<String> = normalized
        .iter()
        .flat_map(|e| extract_path_line_tokens(e))  // 提取 path:line 引用
        .collect();
    if tokens.is_empty() {                       // 无引用则退回整行归一化
        tokens = normalized
            .iter()
            .map(|e| e.trim().to_ascii_lowercase())
            .filter(|e| !e.is_empty())
            .collect();
    }
    tokens.sort();                               // 排序去重：与 skeptic 顺序无关
    tokens.dedup();
    tokens.join("\n")
}
```

### 6.4 状态机

```
ACTIVE ──结论过验证──▶ CONCLUDED(4级之一)
  │
  ├─ blocked×3 ─────▶ BLOCKED（输出阻塞报告，升级用户）
  ├─ fingerprint×3 ─▶ NO_PROGRESS（输出降级结论）
  ├─ LLM/基础设施故障 ▶ INFRA_PAUSED（可恢复，静默重试后继续）
  ├─ 轮次/token 预算尽 ▶ BUDGET_LIMITED（强制出降级结论，沿用现有撞顶强制出结论）
  └─ 用户暂停/引导 ──▶ USER_PAUSED（会话可恢复）
```

每次状态迁移记 EventStream，人审 UI 显示当前状态与迁移原因。

### 6.5 诚实降级

- 工具/数据源不可用时，把"不可用的证据"（错误原文、时间戳）本身存入 timeline，作为降级依据。
- 降级后的结论显式声明降级原因与降级后的验收口径。
- 验证者对降级输出同样审：伪造的证据比诚实的降级更糟，发现即 refute。

---

## 7. 采样健壮性

### 7.1 重试分类表（升级现有 4 次指数退避）

grok-build `retry.rs` 的完整分类，按内网场景裁剪：

| 错误 | 决策 | 参数 |
|------|------|------|
| 5xx / 连接错误 / 流中断 / 空响应 | 重试 | 退避 2s·2^(n-1)，封顶 30s，±20% jitter，最多 15 次（≈6 分钟） |
| 429 | 重试但单独封顶 | 最多 2 次，尊重 Retry-After 头 |
| 401 | 刷新凭证后重试一次 | 并发去重（多工具同时 401 只刷新一次） |
| 400/403/404/422 | 立即失败 | 转 INFRA_PAUSED |
| 上下文超长 | 立即触发压缩后重提 | 不计入重试预算 |
| 响应解析失败 | 立即失败 | 确定性错误，重试无意义 |
| 首个 5xx | 重建 HTTP client 后重试 | 逃逸损坏的连接池 |

jitter 公式：`base ± base/5`，防内网多实例同时重试的 thundering herd。

**grok-build 原实现**（`xai-grok-sampler/src/retry.rs`）。`classify_error` 是纯函数，判定顺序即优先级；两处反直觉设计：服务端 `x-should-retry: true` 故意不强制重试（只信 false，防放大故障）；上下文超长是确定性错误立即失败（重发同样超长）：

```rust
// retry.rs:86 — 指数退避 2s·2^(n-1) 封顶 30s，±20% jitter
pub fn retry_backoff_with_jitter(retry_count: u32) -> Duration {
    let shift = retry_count.saturating_sub(1);
    let base_ms = 2000u64.checked_shl(shift).unwrap_or(u64::MAX).min(30_000);
    let jitter_range = base_ms / 5;
    let jitter = hash(jitter_seq, thread_id) % (jitter_range * 2 + 1);
    Duration::from_millis(base_ms - jitter_range + jitter)
}

// retry.rs:144 — 分类决策链（节选，按真实判定顺序）
pub fn classify_error(err, retry_count, max_retries, rate_limit_threshold) -> RetryDecision {
    if err.is_auth_error()             { return EmitToSession(..); }   // 交会话层刷凭证
    if err.is_payload_too_large()      { return RetryWithImageStrip; } // 剥大负载重试，不计预算
    if let Some(false) = err.should_retry_header() { return Fatal(..); } // 只信"别重试"
    if err.is_context_length_error()   { return Fatal(..); }           // 确定性错误
    if matches!(err, DoomLoopDetected) {                                // 复读循环：近乎立即重采
        return Retry { backoff: doom_loop_backoff(retry_count + 1) };  // < 251ms
    }
    if err.is_rate_limited() {                                          // 429 单独封顶
        let effective_cap = max_retries.min(rate_limit_threshold);      // = 2
        if retry_count + 1 >= effective_cap { return Fatal(..); }
        let backoff = err.retry_after().map(Duration::from_secs)        // 尊重 Retry-After
            .unwrap_or_else(|| retry_backoff_with_jitter(retry_count + 1));
        return RetryWithBackoff { backoff, is_rate_limited: true };
    }
    if err.is_retryable() {                                             // 5xx / 传输层
        if retry_count + 1 >= max_retries { return Fatal(..); }
        if retry_count + 1 == 1 {
            return RetryWithClientRebuild { backoff };                  // 首次重建 HTTP client
        }                                                               // 逃逸损坏的 HTTP/2 连接池
        return Retry { backoff };
    }
    Fatal(..)                                                           // 其余一律失败
}
```

### 7.2 doom loop 检测（进阶，Phase 3）

流式输出中检测尾部 token 序列重复（阈值 8）；检测到即中止本次生成、**近乎立即重采样**（<250ms 退避——循环是采样温度下的随机现象，新采样即解药），单 turn 最多重采 2 次，耗尽后放行当前输出交给验收门处理。

**grok-build 原实现**（`xai-grok-sampling-types/src/doom_loop.rs:110`）——只对 thinking 通道的紧重复有信心，其余信号（response 通道、low_logprob）仅记警告不中止：

```rust
pub fn is_confident(&self, signal: &DoomLoopSignal) -> bool {
    let tight = |t: u32| t <= self.max_threshold;        // 默认 8，范围 2..=64
    signal.channel == THINKING_CHANNEL
        && matches!(signal.kind, DoomLoopSignalKind::TailRepetition(t) if tight(t))
}
```

---

## 8. 交互与运行时

### 8.1 中途引导（升级现有雏形）

用户插话在三个安全点消费（grok-build `interjection.rs`），保证既不打断进行中的工具调用、也绝不丢失：

1. 每轮采样前（turn 开始）
2. 每批工具调用执行完之后
3. turn 收尾前（最后再 drain 一次）

注入格式：**独立的合成用户消息**，标记 `[interjection]`，与工具结果严格隔离：

```
The user sent a message while you were working:
<user_query>{插话内容}</user_query>
```

turn 已结束才到达的插话，转为队列头部的独立 prompt（不丢失）。超长插话按 25000 字符截断并标记。

**grok-build 原实现**（`xai-interjection-core/src/format.rs`）——注意注释里的设计决策：不加"推迟处理"的指令，让模型自己权衡插话与在途工作的优先级：

```rust
pub const LARGE_PROMPT_THRESHOLD: usize = 25_000;

/// Wrap interjection text as a synthetic user message with a mid-turn note.
/// No deferral instruction: the model decides how to weigh it against
/// in-flight work.
pub fn format_interjection(text: String) -> String {
    let truncated = if text.len() > LARGE_PROMPT_THRESHOLD {
        // 按 UTF-8 字符边界截断
        format!("{}... [truncated]", &text[..end])
    } else {
        text
    };
    format!(
        "The user sent a message while you were working:\n{}",
        user_query(&truncated)               // <user_query>...</user_query> 包装
    )
}
```

### 8.2 prompt 队列优先级

排队规则（grok-build `prompt_queue.rs`）：用户消息优先于合成消息（自动续跑、唤醒），可清除排队中的合成消息，但**从不移除正在执行的队头**——保证运行中的调查轮完整落盘后再切换。

### 8.3 后台/长任务

长查询（大范围日志检索、慢 SQL）后台化执行：

- **双轨存储**：内存留截断预览（单行 500 字符、批次 3000 字符），完整输出落盘 `tasks/{task_id}.log`，模型按需分页读取。
- **完成即通知**：完成事件作为下一轮上下文的通知注入；工具描述写明"提交后无需轮询"。
- **输出限速**：token bucket（容量 10，2 秒回填）防日志流刷爆上下文；持续超速 30 秒自动终止任务并如实报告。

### 8.4 会话持久化（对齐增强现有 messages_json）

- 落盘格式统一 JSONL append-only：`chat_history.jsonl`（对话）、`events.jsonl`（EventStream）、`signals.json`（token/轮次统计）、`incident.md`（契约）、假设板快照。每轮增量追加，崩溃丢失面最小。
- 恢复时重建：对话 + Notebook + 假设板 + 契约 + 未完成后台任务清单（附 reminder 提示模型"这些任务在你离开时仍在运行"）。
- 调查归档后 FTS 索引化（沿用 past_investigations 的 SQLite），检索按词拆分打分。

---

## 9. 子代理（Phase 4，规模化选项）

单一调查涉及多服务时，主 agent 派**只读 explore 型子代理**并行查证：

- 子代理裁剪：只挂 log_search / code 系工具 / db_schema（只读集），无审批类工具——派活即安全，无需在子代理里重复审批逻辑。
- 派活工具描述写明适用场景："跨多个服务并行收集证据、验证相互独立的假设时使用；单点深挖由主 agent 自己做。"
- **结果只回传摘要**（结构化：结论 + 关键证据引用 + 涉及文件/日志源），完整transcript落盘可溯源，主上下文只承接结论。
- `resume_from` 支持多阶段：第一阶段广搜证据的子代理，可在第二阶段带上下文继续深挖。
- 嵌套深度 1：子代理内部禁止再派子代理。

---

## 10. 推理档位：按角色分配算力

**是什么**：grok-build 的"标准/深度模式"是同一模型的 reasoning effort 档位（六档：None/Minimal/Low/Medium/High/Xhigh），随采样请求的 `reasoning_effort` 字段透传给模型服务端；产品层的"Balanced/Deep"是服务端下发的展示映射（medium/xhigh）。harness 循环与工具在所有档位下行为一致。

**核心纪律——深度推理只给正主，辅助角色一律零推理**：

| 角色 | 档位 | 理由 |
|------|------|------|
| 主 agent（实现者） | 高档 | 因果推理是核心任务 |
| 对抗验证者（§5.2） | 高档 | 反驳论证需要深推理 |
| 压缩摘要调用（§3.3） | None/最低 | 机械转写任务，要快要便宜（grok-build compaction 即 `reasoning_effort: None`） |
| 停滞分类器（§2.4） | None/最低 | 小 prompt 二分类（grok-build 同） |
| explore 子代理（§9） | 中档 | 广搜证据，深度需求有限 |

档位解析优先级（子代理场景）：spawn 时显式指定 > 角色默认 > persona 默认 > 继承父会话。grok-build 的验证 skeptic 继承会话模型与档位；上表中"验证者用高档"是本设计的分配建议。

档位枚举与产品映射（`xai-grok-sampling-types/src/types.rs:765`、`model_state.rs` 协议测试）：

```rust
pub enum ReasoningEffort { None, Minimal, Low, Medium, High, Xhigh }
```

```json
// 服务端下发的展示映射——"标准/深度"是皮，wire 值是档位
"reasoningEfforts": [
  { "id": "balanced", "value": "medium", "label": "Balanced" },
  { "id": "deep",     "value": "xhigh",  "label": "Deep" }
]
```

**incident-locator 落地**：内部 API 支持 reasoning 参数（或存在推理/非推理双模型）时，在 LlmClient 接口上加 per-call effort 参数，各调用点按上表固定；档位对用户不可见，属 harness 内部资源分配。深浅切换由配置固定，模型不自行决定推理预算。

---

## 11. 实施路线

| 阶段 | 内容 | 验收方式 |
|------|------|---------|
| **P1 主动性**（~2天） | §2.1 门禁 + §2.2 纪律 + §2.3 续行指令并入 Notebook | fixture 实测：叙述无调用被拦截重采；无"要继续吗"式请示 |
| **P2 信息与降级**（~3天） | §3.1 契约 + §3.2 分层裁剪 + §6.1 分级出口 + §6.2 三次规则 + §4.2 查空三分类 | 缺字段工单一次问齐；长调查中后段仍能引用首轮事实；无根因场景输出排除报告而非编造 |
| **P3 准确性**（~3天） | §5.2 对抗验证者（单员起步）+ §5.1 断言→工具调用反向核对 + §7.1 重试分类表 + §3.3 压缩摘要升级 | 埋"伪根因" fixture：验证者能 refute；压缩后事故事实与假设板完整存活 |
| **P4 规模化**（按需） | §5.2 三员 panel + §2.4 停滞分类器 + §6.3 fingerprint + §7.2 doom loop + §8.3 后台任务 + §9 子代理 | 真实事故回放（内网后） |

每阶段沿用现有验证方式：JUnit 回归 + DeepSeek + fixture 实测人工评估。

---

## 附录 A：关键参数速查

| 参数 | 值 | 来源 |
|------|----|------|
| 门禁单 prompt 上限 | 2 次 | TodoGate `max_fires_per_prompt` |
| 裁剪触发 | 窗口 50% | `should_prune` |
| 软裁阈值 / 保留 | 4000 字符 / 头尾各 1500 | `PruningConfig` |
| 硬清年龄 | 10 轮 | `hard_clear_age_turns` |
| 完整保留 | 最近 3 轮 | `keep_last_n_turns` |
| 压缩触发 / 预压缩提前量 | 80% / 10 个百分点 | compaction 默认 |
| 放弃门槛 | 连续 3 次 | `goal_blocked_streak` |
| 停滞指纹 | 连续 3 轮相同 | stall threshold |
| 验证 panel | 3 员，冷面板过半数 | `aggregate_skeptic_verdicts` |
| 验证打回封顶 | 2 轮 | 沿用 ConclusionGate |
| 重试上限 / 退避 | 15 次 / 2s·2^n 封顶 30s ±20% | `retry.rs` |
| 429 重试 | 2 次 | `RATE_LIMIT_RETRY_THRESHOLD` |
| doom loop | 阈值 8，重采 ≤2 次，退避 <250ms | `DoomLoopRecoveryPolicy` |
| 插话截断 | 25000 字符 | `LARGE_PROMPT_THRESHOLD` |
| 停滞分类器 | 空闲 10s 触发，置信 ≥0.7，输出 ≤150 token | laziness 常量 |
| token 估算 | bytes/4，图像 765/张 | `xai-token-estimation` |

## 附录 B：grok-build 源码索引

| 机制 | 路径 |
|------|------|
| 主循环 / 门禁 | `crates/codegen/xai-grok-shell/src/session/acp_session_impl/turn.rs` |
| 行为纪律 / 续行指令 | `crates/codegen/xai-grok-shell/src/session/templates/goal_task_discipline.md`、`goal_continuation_directive.md` |
| 对抗验证者 / 计划器 / 战略师 | 同目录 `goal_verifier_prompt.md`、`goal_planner_prompt.md`、`goal_strategist_prompt.md` |
| 完成判定 / panel 投票 / fingerprint | `crates/codegen/xai-grok-shell/src/session/goal_classifier.rs` |
| 失败状态机 / 三次规则 | `goal_tracker.rs`、`acp_session_impl/goal.rs` |
| 停滞分类器 | `acp_session_impl/laziness_classifier.rs` |
| 上下文裁剪 | `crates/codegen/xai-chat-state/src/actor/request_builder.rs` |
| 压缩实现 / 摘要 prompt | `session/compaction.rs`、`session/helpers/session_compact.rs` |
| 工具层 / 截断 / reminders | `crates/codegen/xai-grok-tools/src/` |
| 重试分类 / doom loop | `crates/codegen/xai-grok-sampler/src/retry.rs`、`doom_loop.rs` |
| 中途引导 / prompt 队列 | `acp_session_impl/interjection.rs`、`prompt_queue.rs` |
| 后台任务 / monitor | `terminal/background_task.rs`、`xai-grok-tools/.../monitor/` |
| 会话持久化 / checkpoint | `docs/user-guide/17-sessions.md`、`xai-grok-workspace/src/session/checkpoint.rs` |
| 子代理 | `crates/common/xai-tool-types/src/task.rs`、`docs/user-guide/16-subagents.md` |
| 基座 prompt | `crates/codegen/xai-grok-agent/templates/prompt.md` |
