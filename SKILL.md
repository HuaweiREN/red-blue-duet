---
name: red-blue-duet
description: "Multi-agent structured debate for problems with no clear answer, where multiple solution paths each have pros/cons. Triggers when user presents a trade-off decision, competing approaches, architecture choice, technology selection, roadmap divergence, or asks to 'compare options', 'which solution is better', 'debate these approaches', '帮我分析哪个方案好', 'A和B怎么选', '有几个思路帮我辩论一下'. NOT for questions with a single verifiable factual answer."
version: 1.5.0
---

# Red-Blue-Duet

Multi-agent structured debate skill. Main agent serves as moderator; sub-agents serve as debaters, each defending one solution path. The moderator orchestrates a 3-round debate, checks for topic drift in every round, requires user confirmation between rounds, and delivers a final judgment with a complete debate record.

## When NOT to Use

- Questions with a single verifiable factual answer → answer directly
- Purely subjective/preference-based questions → ask clarifying questions instead
- User only wants a quick opinion, not a structured debate → confirm intent before engaging

---

## Agent Communication Protocol

All agent-to-agent and agent-to-user communications in debate context MUST follow this protocol.

### Forbidden

| Category | Examples |
|----------|----------|
| Filler words | "我认为", "我感觉", "好像", "I think", "it seems" |
| Transitional padding | "嗯", "那么", "接下来", "well", "so", "next" |
| Encouragement/approval | "很好的观点", "有意思", "good point", "interesting" |
| Unsubstantiated attack | "明显错误", "荒谬", "clearly wrong", "absurd" |
| Verbose restatement | Quoting opponent's full argument before responding |
| Greeting words in debate context | "DAMN!!!" — excluded from debate-related outputs |

### Required

| Rule | Enforcement |
|------|-------------|
| Direct factual assertion | Every claim of fact cites source |
| Specificity in challenge | Each challenge targets a specific claim, with counter-evidence |
| Structured conciseness | **Forbidden**: restating opponent's full argument verbatim; filler transitions between points. **Permitted & encouraged**: extended quantitative analysis, institutional detail enumeration, layered logical deduction — no word limit on substance. |
| Source annotation | Single-source: marked `[单一信源]`; cross-validated: marked `[交叉验证: N源]` |
| Logical chain clarity | Premise → reasoning → conclusion, each step traceable |
| Rebuttal depth | Each response to an opponent's challenge MUST contain: ① fact-check result, ② logical analysis, ③ counter-evidence if applicable |

---

## Phase 0: Problem Definition & Path Enumeration

### 0.1 Trigger Analysis

When the skill triggers, the main agent analyzes the user's problem:

1. **Extract the core dilemma**: One sentence describing the irreducible trade-off.
2. **Enumerate solution paths** (2–3 paths; hard cap at 3):
   - If user provides paths explicitly → use them as-is
   - If user describes a situation without naming paths → the main agent synthesizes 2–3 distinct approaches
3. **Define evaluation dimensions** (default set, modifiable by user):

| Dimension | Weight | What it measures |
|-----------|--------|-----------------|
| Feasibility | 20% | Technical/practical implementability |
| Cost | 20% | Resource, time, opportunity cost |
| Risk | 20% | Failure modes, downside exposure |
| Maintainability | 15% | Long-term sustainability |
| Scalability | 15% | Growth/expansion capacity |
| Team fit | 10% | Alignment with existing skills/stack |

### 0.2 User Confirmation (REQUIRED)

Present to user using `AskUserQuestion`:

```
header: "辩论设定确认"
question: "解决路径和评判维度是否准确？是否需要调整？"
options:
  - label: "确认，开始辩论"
    description: "路径和维度合适，进入辩手准备阶段"
  - label: "需要调整路径"
    description: "补充或修改解决路径"
  - label: "需要调整评判维度"
    description: "修改维度或权重"
```

Do NOT proceed to Phase 1 until user confirms.

### 0.3 Baseline Research

After user confirmation, the main agent conducts **neutral baseline research** before dispatching debaters. This establishes an agreed-upon factual foundation — preventing debaters from arguing past each other using contradictory data.

**Procedure:**

1. **Gather agreed-upon facts** relevant to all paths:
   - Key metrics (e.g., market size, growth rate, cost benchmarks)
   - Domain constants (e.g., regulatory constraints, technical limitations)
   - Uncontested historical data points

2. **Produce baseline fact document** at `/tmp/red-blue-duet/round-0/baseline-facts.md`:

```
# 中立事实基线

## 关键指标
| 指标 | 值 | 信源 | 验证状态 |
|------|-----|------|---------|
| ... | ... | ... | [交叉验证: N源] / [单一信源] |

## 领域约束
- [约束1] — 信源: [...]
- [约束2] — 信源: [...]

## 已知事实分歧点
以下数据在多信源间存在差异，辩手引用时须注明采用的版本：
- [分歧点1]: 版本A ([信源]) vs 版本B ([信源])
```

3. **Estimate time**: If baseline research > 15s, emit one preview sentence — 「正在进行中立事实基线调研，预计 N 秒后完成」— then go silent.

4. **Inject into Phase 1**: The baseline document is appended to ALL debater task dispatches in Phase 1.2. Debaters MUST acknowledge the baseline and build their research on top of it. They MAY supplement with path-specific deep dives and MAY challenge baseline facts only with specific counter-evidence (citing the source of the challenge).

---

## Phase 1: Initial Research & Opening Statements

### 1.1 Create Agent Team

```
Team name: RedBlue-oc_c13a6-duet
Display mode: in-process (forced)
Teammates:
  - debater-A (路径A)
  - debater-B (路径B)
  - debater-C (路径C, if applicable)
```

### 1.2 Task Dispatch Template

For each debater, dispatch a Task with this structure:

```
# 角色
辩手-[X]，立场：路径[X]——「[路径名]」是最优解决路径。

# 中立事实基线（必读）
以下基线由主 agent 中立调研产出，所有辩手共享同一事实基础。
[附: /tmp/red-blue-duet/round-0/baseline-facts.md 完整内容]

# 任务
1. 审阅中立事实基线，确认无异议（如有异议须附具体反证及信源）
2. 在基线之上深挖路径[X]的支撑论据 —— 基线已覆盖的通用事实不再重复调研，聚焦路径[X]的差异化优势
3. 对每条关键数据执行交叉验证；单一信源标注 `[单一信源]`
4. 识别自身路径的已知弱点，预判对手可能攻击面，准备防御论证
5. 输出立论陈词到 opening-statement.md

# 输出格式
## 路径[X]: [路径名]
### 论点
1. [论点1] — 支撑证据: [信源]
2. [论点2] — 支撑证据: [信源]

### 证据清单
| 证据 | 信源 | 交叉验证状态 |
|------|------|-------------|
| ... | ... | [交叉验证: N源] / [单一信源] |

### 已知弱点 & 防御
- [弱点1] — 防御论证: [理由/证据]

# 工作目录
/tmp/red-blue-duet/round-1/debater-[X]/

# 约束
- 遵循 Agent 通信协议（禁止语气词、助词、空洞评价、无端攻击）
- 不确定时明确说明，不编造事实
- 所有中间产物写入指定工作目录
- 基于基线事实之上的增量调研，避免重复基线已覆盖的数据
```

### 1.3 Silent Wait

Main agent waits for ALL sub-agents to complete. If estimated time > 15s, emit ONE preview sentence:

> 辩手正在准备立论陈词，预计 N 秒后完成。

Then go silent until all tasks return. Do NOT emit intermediate commentary.

### 1.4 Collect Opening Statements

Read all `opening-statement.md` files. Verify:
- Each follows the required format
- Cross-validation annotations present
- No communication protocol violations

If violations found → re-dispatch to that debater with correction note.

---

## Phase 2: Debate Rounds (1–3)

### Speaking Order Rotation

To eliminate the systemic advantage of fixed-position speaking (later speakers can respond to earlier speakers' new attacks in the same round, while earlier speakers cannot), the speaking order **rotates across rounds**. Each debater experiences exactly one round as First Speaker, one as Second Speaker, and one as Third Speaker over the three rounds.

| Round | Order | 1st (议程权) | 2nd (过渡) | 3rd (信息权) |
|-------|-------|:----------:|:--------:|:----------:|
| 1 | A → B → C | A | B | C |
| 2 | B → C → A | B | C | A |
| 3 | C → A → B | C | A | B |

**With 2 debaters only (no C):**

| Round | Order | 1st | 2nd |
|-------|-------|:--:|:--:|
| 1 | A → B | A | B |
| 2 | B → A | B | A |
| 3 | A → B | A | B |

### Round Structure (each round identical)

```
Round N:
  Step 1: Main agent assigns speaking order per rotation table
  Step 2: Prepare context packages for all positions (research tasks can be pre-written)
  Step 3: Dispatch debaters STRICTLY SEQUENTIALLY per §2.0 dispatch protocol
  Step 4: Moderator collects, assesses topic drift
  Step 5: Present summary to user → user confirms → next round
```

### 2.0 Dispatch Protocol (CRITICAL — override all default parallelism)

**This section takes precedence over any default agent-dispatch behavior.** The main agent's default inclination to launch independent sub-agents in parallel MUST be suppressed during debate rounds.

Later speakers depend on earlier speakers' same-round rebuttals as input. Parallel dispatch breaks this dependency — the 2nd speaker cannot respond to the 1st speaker's new attacks if they are launched simultaneously.

#### Mandatory Sequential Dispatch

For each round, the main agent MUST dispatch debaters one at a time, in the assigned speaking order:

```
Step A: Dispatch 1st speaker (Agent tool, run_in_background: false or true)
        → Wait for completion
        → Read the rebuttal.md to verify it exists and is complete

Step B: Dispatch 2nd speaker (Agent tool)
        → Prompt MUST embed §2.2.2 "Input" section WITH the 1st speaker's full rebuttal content
        → The 2nd speaker's task template (§2.2 Position 2nd) requires responding to 1st speaker's same-round attacks
        → Wait for completion
        → Read the rebuttal.md

Step C: Dispatch 3rd speaker (Agent tool, if applicable)
        → Prompt MUST embed §2.2.3 "Input" section WITH both 1st and 2nd speakers' full rebuttal content
        → Wait for completion
        → Read the rebuttal.md
```

#### Rules

| Rule | Detail |
|------|--------|
| **Parallel dispatch FORBIDDEN** | Do NOT launch 2+ debaters of the same round simultaneously. Each must see earlier speakers' files. |
| **Inject actual rebuttal** | The prompt for 2nd/3rd speakers must contain the earlier speakers' actual rebuttal text — not a summary, not a prediction |
| **Research may be pre-dispatched** | If desired, research-only tasks can be run in parallel before sequential speaking begins. But the rebuttal-writing Agent call MUST be sequential |
| **Phase 1 is exempt** | Opening statements are independent — parallel dispatch is permitted and recommended for Phase 1 |
| **`run_in_background` is acceptable** | Sequential means one-at-a-time dispatch, not necessarily blocking the main agent. Background dispatch is fine as long as each agent completes before the next is launched |

#### Why This Matters

Without this constraint, the debate degenerates into three parallel monologues. Each debater attacks strawman versions of opponents' positions (based on the moderator's summary) rather than engaging with the opponent's actual same-round arguments. The sequential dependency is what makes it a debate rather than three simultaneous essays.

### 2.1 Context Distribution

Before each round, the main agent prepares a context package for each debater. The package identifies the debater by their **speaking position** in this round (1st / 2nd / 3rd), NOT by a fixed A/B/C role.

```
# 上下文包 (Round N)
## 你的立场
[该辩手的上轮完整论点]

## 你的本轮发言位置: [1st / 2nd / 3rd]
- 1st: 率先发言，可全力攻击，但无法在同一轮内回应后续辩手的质询
- 2nd: 回应 1st 的质询 + 攻击其他辩手
- 3rd: 回应 1st 和 2nd 的质询 + 攻击其他辩手，拥有本轮的完整信息

## 对手观点
### 路径[Y] 核心主张
- [主张1] — [支撑证据摘要]
- [主张2] — [支撑证据摘要]

### 路径[Z] 核心主张
...

## 对手对你的质询 (Round N-1，如适用)
- [具体质询1]
- [具体质询2]

## 用户补充信息 (如适用)
[用户在上轮确认时提供的信息]

## ⚠️ 本轮调研任务（必做）
上一轮辩论暴露了以下事实争议焦点，你需要对每一个焦点做定向调研：

1. **[争议焦点1]** — 搜索并验证双方引用的数据/事实，产出核查结论
2. **[争议焦点2]** — 搜索反向证据，找到支持己方立场或削弱对手立场的新数据
3. 如发现影响辩论走向的新信息，立即补充

调研结果写入独立文件: `/tmp/red-blue-duet/round-[N]/debater-[X]/research-notes.md`

## 本轮答辩任务
1. 重申核心立场
2. 验证对手论据真实性（结合调研结果）
3. 识别对方逻辑链断裂点
4. 回应对手对你的质询
5. 输出答辩词到 /tmp/red-blue-duet/round-[N]/debater-[X]/rebuttal.md
```

### 2.2 Position-Based Sequential Speaking

Debaters speak sequentially per the rotation table (see phase intro). Each later speaker receives the rebuttal files of ALL earlier speakers in the same round. The task template is determined by **speaking position** (1st / 2nd / 3rd), NOT by fixed debater identity.

**Research always runs before speaking for every position.**

---

#### Position 1st: First Speaker

This debater speaks first in this round. They have the **agenda-setting advantage** — their arguments frame the round's discussion. They cannot respond to same-round attacks, so they must preemptively reinforce their position.

```
# ⚠️ 先完成调研，再撰写发言（顺序不可颠倒）

## 第一步: 定向调研（必做）
1. 接收上下文包中列出的「争议焦点」
2. 对每个争议焦点搜索验证，产出事实核查结论
3. 如发现影响辩论走向的新数据，记录并准备引用
4. 输出: /tmp/red-blue-duet/round-[N]/debater-[X]/research-notes.md

## 第二步: 本轮发言 (1st Speaker)
1. 重申核心立场（每点立场可附带 1-2 句支撑说明）
2. 回应上一轮对手对你的质询 —— 逐条，每条回应包含: ① 事实核查结果 ② 逻辑分析 ③ 反向证据（如适用）
3. 对每个对手路径提出质询（每个对手最多3条，每条附带具体数据引用或逻辑推演，不设字数上限）
4. 补充论证 —— 基于本轮调研发现的新证据（必做）
5. 预判对手可能的攻击面并前置防御（必做 —— 你是本轮唯一无法当轮回应的人）

# 输出: /tmp/red-blue-duet/round-[N]/debater-[X]/rebuttal.md
# 约束: 不得以简洁为由省略定量分析、制度引用或关键推理链
# 注意: 你无法在本轮内回应后续发言者的质询，因此务必在前置防御中覆盖你最脆弱的论点
```

---

#### Position 2nd: Second Speaker

This debater speaks second. They must respond to the **1st speaker's new attacks** from this round, plus last round's outstanding challenges. They also attack all opponents.

```
# ⚠️ 先完成调研，再撰写发言（顺序不可颠倒）

## 第一步: 定向调研（必做）
1. 接收上下文包中列出的「争议焦点」
2. 对每个争议焦点搜索验证，产出事实核查结论
3. 特别关注: 验证本轮第1位发言者新提出的数据/事实主张
4. 输出: /tmp/red-blue-duet/round-[N]/debater-[X]/research-notes.md

## 第二步: 本轮发言 (2nd Speaker)
1. 重申核心立场（每点立场可附带 1-2 句支撑说明）
2. 回应上一轮对手对你的质询 —— 逐条，每条回应包含: ① 事实核查结果 ② 逻辑分析 ③ 反向证据（如适用）
3. **回应本轮第1位发言者对你的质询** —— 逐条，同上深度要求
4. 对每个对手路径提出质询（每个对手最多3条，每条附带具体数据引用或逻辑推演）
5. 补充论证 —— 基于本轮调研发现的新证据（必做）

# 输入: 本轮第1位发言者的 rebuttal.md（已附在上下文包中）
# 输出: /tmp/red-blue-duet/round-[N]/debater-[X]/rebuttal.md
# 约束: 不得以简洁为由省略定量分析、制度引用或关键推理链
```

---

#### Position 3rd: Third Speaker

This debater speaks last. They have the **information advantage** — they see both the 1st and 2nd speakers' new attacks. They can deliver the round's definitive rebuttal, but must divide attention between defending against two speakers' simultaneous challenges and mounting their own attacks.

```
# ⚠️ 先完成调研，再撰写发言（顺序不可颠倒）

## 第一步: 定向调研（必做）
1. 接收上下文包中列出的「争议焦点」
2. 对每个争议焦点搜索验证，产出事实核查结论
3. 特别关注: 验证本轮第1位和第2位发言者新提出的数据/事实主张
4. 输出: /tmp/red-blue-duet/round-[N]/debater-[X]/research-notes.md

## 第二步: 本轮发言 (3rd Speaker)
1. 重申核心立场（每点立场可附带 1-2 句支撑说明）
2. 回应上一轮对手对你的质询 —— 逐条，每条回应包含: ① 事实核查结果 ② 逻辑分析 ③ 反向证据（如适用）
3. **回应本轮第1位发言者对你的质询** —— 逐条，同上深度要求
4. **回应本轮第2位发言者对你的质询** —— 逐条，同上深度要求
5. 对每个对手路径提出质询（每个对手最多3条，每条附带具体数据引用或逻辑推演）
6. 补充论证 —— 基于本轮调研发现的新证据（必做）

# 输入: 本轮第1位和第2位发言者的 rebuttal.md（已附在上下文包中）
# 输出: /tmp/red-blue-duet/round-[N]/debater-[X]/rebuttal.md
# 约束: 不得以简洁为由省略定量分析、制度引用或关键推理链
# 注意: 你拥有本轮完整信息，但防御负担最重——合理分配篇幅
```

### 2.3 Topic Drift Assessment

After all debaters in a round have spoken, the main agent evaluates each debater:

| Check | Method |
|-------|--------|
| Core claim consistency | Does the debater still defend the assigned path? |
| Factual accuracy | Spot-check 1–2 key claims against sources |
| Relevance | Are arguments about the problem domain or unrelated tangents? |
| Protocol compliance | No filler words, no unsubstantiated attacks |

Output drift assessment:

```
## Round N 偏离度检查
| 辩手 | 状态 | 说明 |
|------|------|------|
| A | ✅ 未偏离 | — |
| B | ⚠️ 偏离 | 偏离方向: [具体描述]；修正要求: [具体修正] |
| C | ✅ 未偏离 | — |
```

If any debater drifted → re-dispatch that debater with explicit correction before presenting to user.

### 2.4 User Confirmation (REQUIRED after every round)

Present round summary + drift assessment to user using `AskUserQuestion`:

```
header: "Round N 确认"
question: "本轮辩论是否偏离主题？是否需要补充背景信息？"
options:
  - label: "继续下一轮"
    description: "辩论方向正确，进入 Round N+1"
  - label: "补充信息"
    description: "我需要补充背景信息或纠正方向"
  - label: "提前终止"
    description: "信息已足够，直接进入判决"
```

**Critical rule**: User-supplied information from this step MUST be appended to the context package of ALL debaters in the next round. If user names a specific debater as drifted, the correction applies to that debater AND the user's general input applies to all.

### 2.5 Round Progression

| Round | Focus | Research Emphasis |
|-------|-------|-------------------|
| Round 1 | 立论 + 初步质询（确立各自立场和核心分歧） | 主 agent 识别首轮争议焦点，为 Round 2 准备定向调研任务 |
| Round 2 | 深度交锋（验证对手证据、指出逻辑谬误） | 辩手针对 Round 1 暴露的事实分歧做定向调研；每轮输出独立 research-notes.md |
| Round 3 | 最终陈述（回应所有质疑、总结不可动摇的核心优势） | 最终轮调研聚焦于裁决级证据——即能一锤定音的关键数据点 |

After Round 3, proceed to Phase 3. Hard stop at 3 rounds — no exceptions.

---

## Phase 3: Final Judgment

### 3.1 Design Rationale

During Phases 0–2, the main agent (moderator) accumulates private knowledge beyond what debaters had access to: baseline research findings, topic drift assessments, and interim evaluations. If the moderator also acts as judge, this information asymmetry compromises objectivity — the moderator may penalize a debater for failing to cite data that only the moderator discovered, or favor a path that aligns with pre-debate impressions.

To ensure a level playing field, Phase 3 spawns a **fresh judge subagent** that receives ONLY information equally available to all debaters: the shared baseline facts + the complete debate record. The judge is explicitly barred from conducting additional research.

### 3.2 Evaluation Framework

The judge subagent scores each path using these dimensions:

| Dimension | Weight | Scoring Method |
|-----------|--------|---------------|
| Evidence quality | 30% | Cross-validation pass rate, source reliability |
| Logical coherence | 25% | Premise→conclusion chain integrity |
| Response to challenges | 20% | Effectiveness in addressing opponent challenges |
| Refutation of opponents | 15% | Specific, evidence-backed identification of opponent flaws |
| Practical applicability | 10% | Engineering/business implementability |

### 3.3 Assemble Judge Input Package

The main agent compiles the following package. **Critical**: the package must be self-contained — the judge subagent receives no implicit context from the conversation.

```
# 角色
你是本次辩论的独立裁判。你的唯一任务是基于下方提供的辩论记录，做出客观、量化的裁决。

# 输入材料

## 1. 问题定义
[Phase 0 的问题背景、路径枚举、评判维度及权重]

## 2. 中立事实基线
[baseline-facts.md 完整内容]

此基线由主 agent 在辩论前完成中立调研并分发给所有辩手。所有辩手拥有同等机会对基线中的任何数据提出质疑和反驳。你可以将基线事实作为验证辩手证据主张的参照。

## 3. 完整辩论记录

### Phase 1: 立论阶段
#### 路径A 立论陈词
[opening-statement.md 完整内容]

#### 路径B 立论陈词
[opening-statement.md 完整内容]

#### 路径C 立论陈词（如适用）
[opening-statement.md 完整内容]

### Phase 2: 辩论过程
#### Round 1
##### debater-A 发言
[rebuttal.md 完整内容]
##### debater-B 发言
[rebuttal.md 完整内容]
##### debater-C 发言（如适用）
[rebuttal.md 完整内容]

#### Round 2
[同上结构]

#### Round 3
[同上结构]

# 评判维度与权重
| 维度 | 权重 | 评分方法 |
|------|------|---------|
| 证据质量 | 30% | 交叉验证通过率、信源可靠性。结合基线事实核查辩手引用的证据是否真实、是否被对手证伪。 |
| 逻辑一致性 | 25% | 前提→推理→结论 链条完整性。是否存在跳跃、循环论证或自相矛盾。 |
| 回应质询有效性 | 20% | 对对手质询的回应是否逐条、深度、有据。回避或绕开的质询扣分。 |
| 对对手的反驳 | 15% | 对对手路径缺陷的识别是否具体、是否有反向证据支撑。 |
| 实践可行性 | 10% | 工程/业务可实现性。基于辩论记录中的实施细节、资源估算等。 |

# 约束（严格遵循，违反任一条则裁决无效）

1. **禁止额外调研**：你只能基于上述输入材料做出裁决。不得自行搜索、不得引用材料之外的任何数据或事实。你的知识截止日期在此任务中不适用——辩论记录是唯一的信息来源。

2. **信息对等原则**：你拥有的事实基础（基线 + 辩论记录）与所有辩手在辩论结束时拥有的信息完全一致。不得利用任何"外部知识"填补辩论记录的空白或修正辩手的错误。

3. **证据审查标准**：辩手的主张只有在辩论记录中有明确信源引用时才视为"有证据支撑"。基线中的事实可作为验证参照，但基线本身不替代辩手的举证责任。

4. **无法裁决时诚实说明**：如果辩论记录不足以支撑某个维度的判断，在判决书中明确标注"信息不足"，不得强行评分。

5. **判决必须有辩论记录引文支撑**：每条胜出理由和落败分析都必须引用辩论记录中的具体内容，格式: `[路径X, Round N, rebuttal.md]` 或 `[路径X, opening-statement.md]`。

6. **不得推测辩手意图**：只评价辩手实际写出的内容。不得假设"辩手可能是想表达……"或"辩手应该知道……"。

7. **必须产出可执行建议**：判决的最终价值是指导用户行动。基于获胜路径的论证，产出具体的「建议与行动方案」。行动建议必须锚定辩论记录中辩手已论证过的内容，不得凭空建议辩论中未曾讨论的措施。

# 输出
判决书写入 `/tmp/red-blue-duet/judgment.md`，格式严格遵循下方 §3.5 判决输出模板。
```

### 3.4 Dispatch Judge Subagent

| Rule | Detail |
|------|--------|
| **Fresh subagent** | Use Agent tool with a new, independent subagent — NOT the main agent's own reasoning, NOT a debater agent |
| **Prompt** | Use the complete input package from §3.3 verbatim |
| **On failure** | Re-dispatch once with the same prompt. If fails again, note the failure in the debate record and have the moderator deliver a fallback judgment, explicitly marking it as "moderator-scored (judge subagent unavailable)" rather than "judge-scored" |
| **Post-dispatch** | Main agent reads `/tmp/red-blue-duet/judgment.md`, verifies format compliance, and presents to user |

### 3.5 Judgment Output Template

The judge subagent writes to `/tmp/red-blue-duet/judgment.md`:

```
## 判决
**获胜方: 路径[X]——「[路径名]」**

### 评分明细
| 维度 | 权重 | 路径A | 路径B | 路径C |
|------|------|-------|-------|-------|
| 证据质量 | 30% | [score] | [score] | [score] |
| 逻辑一致性 | 25% | [score] | [score] | [score] |
| 回应质询有效性 | 20% | [score] | [score] | [score] |
| 对对手的反驳 | 15% | [score] | [score] | [score] |
| 实践可行性 | 10% | [score] | [score] | [score] |
| **加权总分** | **100%** | **[total]** | **[total]** | **[total]** |

### 胜出依据
1. [核心理由1] — 决定性证据: [引用辩论记录中的具体证据及信源，标注引用位置]
2. [核心理由2]
...

### 对手路径落败分析
**[路径Y]**
- 核心主张「[具体主张]」被 [证据X, 引用位置: 路径Z Round N rebuttal.md] 证伪/削弱
- 逻辑断裂点: [具体环节及分析]
- 未有效回应的质询:
  - [来自路径Z Round N 的质询]: [为何回应不充分]

**[路径Z]** (如适用)
...

### 信息不足项（如有）
| 维度 | 路径 | 说明 |
|------|------|------|
| [维度] | [路径] | [辩论记录中缺失什么信息导致无法评分] |

### 边界条件
- 路径[X] 在 [条件1] 下不适用
- 若 [条件2] 变化，路径[Y] 可能更优
- 上述分析基于当前可用信息，以下假设如变化需重新评估: [列出关键假设]

### 建议与行动方案

> 基于获胜路径「[路径X]」的核心论证，提取可执行的建议与行动步骤。

#### 核心建议
[用 2-3 句话概括推荐方案。必须锚定辩论记录中辩手的核心论据——不得超出辩论中已经论证过的范围。]

#### 行动步骤
| 步骤 | 行动项 | 优先级 | 预计周期 | 依据（辩论记录引用） |
|------|--------|--------|---------|-------------------|
| 1 | [具体行动] | P0/P1/P2 | [时间估算] | [路径X, Round N, 论点Y] |
| 2 | [具体行动] | P0/P1/P2 | [时间估算] | [引用] |
| 3 | ... | ... | ... | ... |

#### 风险缓解
| 风险 | 来源 | 缓解措施 |
|------|------|---------|
| [边界条件/已知弱点的具体体现] | [边界条件 / 路径X已知弱点 / 对手路径Y的质询] | [具体应对] |

#### 关键监控指标
以下假设如发生变化，需重新评估方案:
1. [假设1] — 监控方式: [如何判断假设是否还成立]
2. [假设2] — 监控方式: [...]
```

### 3.6 User Confirmation

Main agent reads `/tmp/red-blue-duet/judgment.md`, presents the judgment to user. User may accept or request reconsideration. If reconsideration requested, note the specific concern and **re-dispatch the judge subagent** with the user's concern appended as additional input (but do NOT re-open debate rounds).

---

## Phase 4: Record Delivery

### 4.1 Generate Debate Record

Compile the complete debate record into a single `.md` file. Follow the template in `references/debate-record-template.md`.

**Compilation rules** (overriding default summarization behavior):

| Rule | Detail |
|------|--------|
| **Preserve quantitative data** | ALL numerical comparisons, percentage calculations, cost estimates, time-window analyses — include in full |
| **Preserve institutional references** | ALL citations of specific regulations, policies, legal provisions, official programs — include with source |
| **Preserve logical chains** | When a debater constructs a multi-step deduction (premise → reasoning → conclusion), preserve the FULL chain — do not collapse to conclusion-only |
| **Preserve fact-check results** | ALL fact-verification outcomes (even when the check "fails" — source not found, data contradicts claim) |
| **Compress only** | Transitional language, repeated headers, re-summaries of already-stated positions — may be compressed |
| **Minimum round record** | Each debater's speech in the compiled record MUST occupy at least **3 substantial paragraphs** (or equivalent in tables/bullets) per round — a 2-line summary is NEVER sufficient |

File naming: `red-blue-duet-{YYYYMMDD}-{命题缩写}.md`

### 4.2 File Content Structure

```
# 辩论记录: [命题]

## 元信息
- 日期: YYYY-MM-DD
- 参与辩手: A (路径A), B (路径B), C (路径C)
- 辩论轮次: N
- 评判维度: [维度列表及权重]

## Phase 0: 问题定义
[命题描述、路径定义、评判维度]

## 中立事实基线
[baseline-facts.md 完整内容]

## Phase 1: 立论阶段
### 路径A 立论陈词
[完整 opening-statement.md 内容]

### 路径B 立论陈词
...

## Phase 2: 辩论过程
### Round 1
#### debater-A 发言
[保留 rebuttal.md 全部实质性内容: 回应质询的完整逻辑链、新提出的数据和信源、补充论证段落]
#### debater-B 发言
[同上]
#### debater-C 发言 (如适用)
[同上]
#### 偏离度检查
#### 用户反馈

### Round 2
...

## Phase 3: 终局判决
[完整判决书]

## 附录
### A. 工作产物目录
[包含 round-0/baseline-facts.md 及各轮 research-notes.md]
### B. 信源清单
[汇总所有引用的信源]
```

### 4.3 Deliver

```bash
cp red-blue-duet-{YYYYMMDD}-{命题缩写}.md /tmp/metabot-outputs-huawei/oc_c13a638bb38d8bd0ea81d19413b12084/
```

Summary message to user:
```
辩论完成。完整记录已生成: red-blue-duet-{YYYYMMDD}-{命题缩写}.md
获胜方: 路径[X]
产物包含: 评分明细 / 胜出依据 / 对手落败分析 / 建议与行动方案
```

### 4.4 Cleanup

```
Clean up the team (RedBlue-oc_c13a6-duet)
```

Remove `/tmp/red-blue-duet/` after successful delivery (optional; keep if user might want raw artifacts).

---

## Workflow Checklist

Copy this checklist and check off items during execution:

```
Red-Blue-Duet Progress:
- [ ] Phase 0: Problem Definition
  - [ ] 0.1 Analyze problem, enumerate paths, define dimensions
  - [ ] 0.2 User confirmation ⚠️ REQUIRED
  - [ ] 0.3 Baseline research → produce baseline-facts.md
- [ ] Phase 1: Opening Statements
  - [ ] 1.1 Create Agent Team (RedBlue-oc_c13a6-duet)
  - [ ] 1.2 Dispatch research tasks to all debaters (include baseline)
  - [ ] 1.3 Collect and validate opening statements
- [ ] Phase 2: Debate Rounds
  - [ ] Round 1 (order: A→B→C / A→B)
    - [ ] 2.1 Assign speaking order + prepare context package (inc. research tasks)
    - [ ] 2.2 Debaters execute research → research-notes.md, then speak in assigned order
    - [ ] 2.3 Topic drift assessment
    - [ ] 2.4 User confirmation ⚠️ REQUIRED
  - [ ] Round 2 (order: B→C→A / B→A)
    - [ ] (same substeps — research tasks updated per newly surfaced disputes)
  - [ ] Round 3 (order: C→A→B / A→B)
    - [ ] (same substeps)
- [ ] Phase 3: Final Judgment
  - [ ] 3.1 Assemble judge input package (problem def + baseline + full debate record)
  - [ ] 3.2 Dispatch fresh judge subagent (NOT moderator-scored)
  - [ ] 3.3 Read judgment.md, verify format compliance
  - [ ] 3.4 Present judgment to user ⚠️ REQUIRED
  - [ ] 3.5 Handle reconsideration if requested (re-dispatch judge, not re-open rounds)
- [ ] Phase 4: Record Delivery
  - [ ] 4.1 Compile debate record .md (preserve quantitative data, institutional refs, logic chains)
  - [ ] 4.2 Copy to output directory
  - [ ] 4.3 Clean up Agent Team
```

---

## Edge Cases

| Scenario | Action |
|----------|--------|
| User provides only 1 path | Ask user: "至少需要2个路径才能辩论。请补充另一个方案，或由我基于问题生成替代路径。" |
| User provides >3 paths | Select top 3 most distinct paths; inform user of selection rationale |
| A debater fails to complete | Re-dispatch once with same prompt; if fails again, note failure in record and continue with remaining debaters |
| User skips confirmation | Do NOT proceed; confirmation is mandatory between rounds |
| User wants to end early | Accept; skip remaining rounds, proceed to Phase 3 with available data |
| Debate reaches no clear winner | Declare the result honestly; present strengths of each path and conditions under which each would be optimal |
| User asks factual question mid-debate | Answer directly (not as a debater); the moderator provides neutral information |
| Debaters discover new path mid-debate | Note it; ask user at next confirmation whether to include as a new debater (capped at 3 total) |

---

## References

| File | Content |
|------|---------|
| `references/debate-record-template.md` | Full .md template for the final debate record |
