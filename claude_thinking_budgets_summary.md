# Thinking Budgets, Effort Levels, and Runtime Compute Tradeoffs

## 1. Core thesis

The session’s central point: **thinking is now a runtime compute allocation problem.** Claude’s quality is not determined only by model size or training compute. It is also shaped by **test-time compute**: how many tokens Claude spends while solving the task, including reasoning, tool use, and final output. The speaker frames this as a “thinking lever”: more runtime compute usually improves performance, but costs more tokens and latency, with diminishing returns at the top end.

For Claude Code workflows, this means you should stop thinking in binary terms:

> “Should I turn thinking on or off?”

and instead think:

> “How much runtime compute should this task deserve, and where should it be spent: planning, tools, code edits, tests, review, or final explanation?”

That is the practical shift.

---

## 2. Test-time compute: the underlying concept

### 2.1 What test-time compute means

The speaker defines test-time compute as **compute spent at inference/runtime**, not during model training. The model spends additional tokens to reason, use tools, inspect context, and compose the final answer. The claim is that, across domains such as agentic coding, difficult QA, computer-use tasks, and PhD-level benchmarks, Claude generally performs better when allowed to spend more tokens before answering.

Conceptually:

```text
Task Quality ≈ f(model capability, context quality, test-time compute, tool access, verification loop)
```

For Claude Code:

```text
Good output ≠ big model only
```

A stronger formula is:

```text
Good output =
right model
+ right effort
+ right context
+ right tools
+ right tests
+ right stop condition
```

---

## 3. The three runtime token sinks

The transcript divides test-time compute into three broad categories:

| Runtime token sink | What it is | Claude Code analogue |
|---|---|---|
| **Thinking space** | Scratchpad-like reasoning before or between actions | Planning architecture, diagnosing bug causes, comparing implementation strategies |
| **Tool calling** | External actions: search, MCP, file reads/writes, shell commands, test execution | Reading files, editing code, running tests, using Git, querying docs |
| **Text output** | Final response to the user | Summary, implementation report, PR description, change explanation |

The key implication: **effort controls more than hidden reasoning.** It changes how Claude uses tools and how much explanatory output it emits. Anthropic’s current API docs say effort can affect text responses, function arguments, tool calls, and extended thinking; lower effort tends to reduce tool calls and terse outputs, while higher effort may produce more planning, more tool use, richer summaries, and more comprehensive comments.

For Claude Code, that means `/effort low` is not simply “same agent, less private thought.” It can become **a different operating regime**: fewer reads, fewer probes, less exploration, fewer comments, less handholding.

---

## 4. Effort levels: what they mean

The transcript describes effort as a user-facing lever with levels such as low, medium, high, extra high, and max. Current Anthropic docs use `low`, `medium`, `high`, `xhigh`, and `max`, with model-specific availability. For Claude Code, current docs say Opus 4.7 supports `low`, `medium`, `high`, `xhigh`, and `max`; Opus 4.6 and Sonnet 4.6 support `low`, `medium`, `high`, and `max`.

### Practical effort ladder

| Effort | Use when | Avoid when | Claude Code examples |
|---|---|---|---|
| `low` | Task is short, scoped, repetitive, latency-sensitive, or low blast-radius | Task requires architecture judgment, security reasoning, or broad codebase search | Rename variable, summarize one file, classify logs, generate a small regex |
| `medium` | You want cost reduction but still need competent execution | You need deep reasoning or multi-file design | Small refactor, simple tests, update docs, apply known pattern |
| `high` | Intelligence matters; you need balanced reasoning and token control | Task is truly frontier or long-horizon | Debug nontrivial bug, implement feature across 2–5 files, review PR |
| `xhigh` | Coding/agentic default for Opus 4.7; long exploration, repeated tools, detailed search | Budget-sensitive runs, trivial tasks | Architecture planning, unfamiliar repo exploration, complex migration |
| `max` | Highest capability, no serious token constraint, task is genuinely hard | Routine work; structured extraction; cases prone to overthinking | Deep design review, nasty concurrency bug, security-critical refactor, high-stakes production incident |

The transcript’s guidance: **max can help on the hardest tasks, but usually has diminishing marginal returns.** Extra high / `xhigh` is the practical Pareto point for many coding and agentic tasks, while high is a strong minimum when intelligence matters.

---

## 5. The quality/latency/cost tradeoff

The transcript’s live example used a traffic simulation prompt at low, high, and max effort.

### Observed pattern from the demo

| Effort | Approximate behavior in demo | Quality effect |
|---|---|---|
| Low | ~50 seconds, ~4,600 output tokens | Functional but simplistic simulation |
| High | Roughly double time and tokens | More realistic vehicles, better traffic light placement, more intelligent vehicle behavior |
| Max | Roughly 10× time/tokens | Most visually detailed and physically plausible output |

The lesson is not “always use max.” The lesson is:

```text
Δ Quality / Δ Tokens
```

matters.

At low effort, you may get 70–85% of the answer quickly. At high or `xhigh`, you may get the useful jump: better architecture, fewer mistakes, more robust tool use. At max, you may pay a lot for the last few percent.

### Practical Claude Code decision rule

Use this triage:

| Question | If yes | Effort implication |
|---|---|---|
| Is failure cheap and easy to detect? | Yes | `low` or `medium` |
| Is the task multi-file or stateful? | Yes | `high` minimum |
| Is the repo unfamiliar? | Yes | `xhigh` for exploration/planning |
| Does it require security, data loss, auth, payments, migrations, concurrency? | Yes | `xhigh` or `max` |
| Is there a reliable test suite? | Yes | You can often use lower effort plus strong verification |
| Is the task ambiguous and no tests exist? | Yes | Raise effort; demand plan + acceptance criteria |
| Is latency critical? | Yes | Lower effort, narrower prompt, explicit stop condition |
| Is token budget critical? | Yes | Use skills, subagents, scoped prompts, filtered test output |

---

## 6. Budgets vs effort

The session distinguishes two control families.

### 6.1 Effort

Effort is a **soft behavioral signal**: it tells Claude how much reasoning depth and runtime exploration to apply. Effort is not a strict token budget; at low effort Claude may still think on hard tasks, but less than it would at higher effort.

In Claude Code, current controls include:

```bash
/effort low
/effort medium
/effort high
/effort xhigh
/effort max
/effort auto
```

You can also set effort via `/model`, `--effort`, `CLAUDE_CODE_EFFORT_LEVEL`, settings, and skill/subagent frontmatter. Current docs state that `max` is session-only in settings, while `low`, `medium`, `high`, and `xhigh` persist across sessions.

### 6.2 Budgets

Budgets are **harder constraints**: max tokens, thinking-token budgets, or higher-level task budgets. The transcript frames budgets as a way to say, effectively: “Spend only this much money/time/tokens on this task.”

Current docs add an important update: for Opus 4.7, manual `budget_tokens` extended thinking is no longer supported; Anthropic recommends adaptive thinking with the effort parameter instead. For Opus 4.6 and Sonnet 4.6, manual thinking budgets still function but are deprecated.

So the modern mental model is:

| Control | Type | Best use |
|---|---|---|
| `effort` | Soft allocation policy | Choose reasoning/tool depth |
| `max_tokens` | Hard output ceiling | Prevent runaway output or reserve room for reasoning/actions |
| `MAX_THINKING_TOKENS` | Legacy/fixed-budget control for some modes | Cost control or disabling thinking, model-dependent |
| Task budget | Higher-level spending/time constraint | Long-horizon autonomous workflows |

---

## 7. Adaptive thinking vs interleaved thinking vs old extended thinking

This is one of the most important conceptual parts.

### 7.1 Old extended thinking

Earlier reasoning models often behaved like:

1. Think for a while.
2. Execute tools.
3. Produce final answer.

That is powerful, but unnatural for agentic workflows.

### 7.2 Interleaved thinking

Interleaved thinking improved this by allowing Claude to think **after tool calls**:

1. Think.
2. Use tool.
3. Think again.
4. Use another tool.
5. Answer.

That matches software work better. You inspect a file, reason, inspect another file, update hypothesis, run tests, reason again.

### 7.3 Adaptive thinking

Adaptive thinking generalizes further. Claude can decide whether to:

- think,
- call a tool,
- output text,
- ask a question,
- skip thinking entirely,

in whatever order is appropriate. The transcript emphasizes that adaptive thinking is not merely a classifier deciding task difficulty; it gives Claude a thinking tool and lets it decide when the tool is useful.

Current Claude Code docs say adaptive reasoning makes thinking optional on each step, allowing Claude to answer routine prompts faster and reserve deeper thinking for steps that benefit from it. Opus 4.7 always uses adaptive reasoning; for Opus 4.6 and Sonnet 4.6, fixed thinking budget behavior can still be forced with `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1`, but that does not apply to Opus 4.7.

### Practical implication

Do **not** treat thinking as a global on/off switch. Treat it like web search or test execution:

> Make the capability available. Then constrain the operating regime.

Bad prompt:

```text
Do not think. Just implement.
```

Better prompt:

```text
Use low effort. This is a narrow mechanical change. Edit only the target file, run the relevant test, and report only failures or success.
```

For hard tasks:

```text
Use xhigh effort. First map the relevant code path, identify competing implementation strategies, choose one, implement incrementally, run focused tests after each change, and stop if the architecture assumption is contradicted.
```

---

## 8. Why “thinking toggle” is the wrong abstraction

The speaker explicitly argues that a thinking toggle is a poor proxy for effort. Turning thinking off is like telling a teammate: “Do not use your inner monologue.” That does not express how hard the task is; it removes a core capability.

The better abstraction:

| Poor control | Better control |
|---|---|
| Thinking on/off | Effort level |
| Always search / never search | Give tool access and specify when to use it |
| Always max reasoning | Escalate only when quality gain justifies cost |
| Always cheap model | Route by intelligence sensitivity |
| Huge CLAUDE.md | Small base context + on-demand skills |

Current Claude Code docs also warn that thinking tokens are billed as output tokens and that simpler tasks can reduce cost by lowering effort via `/effort` or `/model`, disabling thinking in `/config`, or lowering `MAX_THINKING_TOKENS` where applicable.

---

## 9. Larger model low effort vs smaller model high effort

The transcript directly addresses a common optimization question:

> Should I use a smaller model with high effort, or a larger model with low effort?

The speaker’s conclusion: if the task requires meaningful intelligence, you are often better off using the larger model, even at lower effort. Smaller models make sense for low-intelligence, routine, or simple tasks where the desired output is easy and the cost of imperfection is low.

### Claude Code routing rule

| Task type | Recommended routing |
|---|---|
| Mechanical edit | Smaller/cheaper model or lower effort |
| Boilerplate generation | Sonnet/medium or high |
| Deep architecture | Opus/`xhigh` |
| Bug requiring hypothesis search | Opus or Sonnet high/`xhigh`, depending on blast radius |
| Security/auth/payment/data migration | Opus `xhigh` or `max` |
| Batch lint cleanup | Lower effort + tests |
| PR review | High/`xhigh`; max only for critical code |
| Subagent reconnaissance | Low/medium if narrow; high/`xhigh` if exploratory |

The trap is using a weak model with high effort for tasks that require conceptual abstraction, codebase-level reasoning, or taste. That can become “expensive mediocrity.”

---

## 10. Low effort is not merely worse; it can be strategically useful

The transcript gives a Pokémon example: at low effort, Claude allegedly found a speedrun-like strategy using repels, potions, escape ropes, and running from battles. The interpretation was that constraining reasoning can push the model into different attractor states: more direct, more opportunistic, less deliberative.

For coding, low effort can be useful when you want:

- minimal ceremony,
- fewer exploratory tool calls,
- less architecture wandering,
- direct patching,
- cheap batch work,
- lower latency,
- small scoped edits.

Example:

```text
Use low effort. Make only the smallest edit needed to fix this TypeScript type error. Do not refactor. Do not inspect unrelated files. Run only the failing test if obvious.
```

This is often superior to:

```text
Think deeply about the codebase and improve the implementation.
```

because the latter invites broad scanning and expensive overreach.

---

## 11. Evals are the correct way to choose effort

The speaker’s strongest best-practice recommendation: evaluate your workload at multiple effort levels. Do not guess. Build a test set containing the hard cases you actually care about, then run the same tasks across models and efforts.

### Minimal Claude Code effort eval harness

Create a small benchmark suite for your repo:

```text
evals/
  bugfix-auth-refresh.md
  refactor-payment-adapter.md
  add-test-for-cache-expiry.md
  migrate-api-client.md
  review-security-sensitive-pr.md
```

Each eval should include:

| Field | Example |
|---|---|
| Goal | “Fix token refresh race condition.” |
| Context | Relevant files or issue description |
| Constraints | “No public API changes.” |
| Acceptance tests | “npm test auth-refresh.spec.ts passes.” |
| Human rubric | Correctness, minimality, safety, maintainability |
| Token/latency budget | e.g. under 20 min, under N dollars |
| Expected failure modes | Over-refactor, misses concurrency bug, changes auth semantics |

Run:

```bash
claude --effort low
claude --effort medium
claude --effort high
claude --effort xhigh
claude --effort max
```

Then score:

| Metric | Weight |
|---|---:|
| Tests pass | 30% |
| Patch minimality | 15% |
| Correct root cause | 20% |
| No collateral damage | 15% |
| Explanation quality | 10% |
| Cost/latency | 10% |

Your optimal setting is not the highest score alone. It is:

```text
utility =
quality score
- λ1 · cost
- λ2 · latency
- λ3 · operational risk
```

For solo Claude Code work, I would start with a simpler rule:

- Use `xhigh` for planning/evals.
- Use `high` or `medium` for implementation if the plan is already good.
- Use `low` for narrow mechanical subagents.
- Use `max` only when `xhigh` fails on your hardest evals.

---

## 12. Claude Code application patterns

### Pattern A — Two-phase “think high, execute cheaper”

Use high or `xhigh` for planning, then lower effort for implementation.

```text
Phase 1: Use xhigh effort. Analyze the codebase and produce a plan only. No edits.
Focus on architecture, risk, test strategy, and rollback plan.

Phase 2: Use medium or high effort. Implement exactly the approved plan.
Do not revisit architecture unless a test failure contradicts the plan.
```

This matches Claude Code’s plan-before-editing workflow.

Use:

```bash
claude --permission-mode plan --effort xhigh
```

Then after approving the plan:

```bash
/effort high
```

or for known mechanical implementation:

```bash
/effort medium
```

### Pattern B — Escalation ladder

Start cheap, escalate only on evidence.

```text
1. low: locate obvious issue
2. medium: patch narrow issue
3. high: if tests fail or root cause unclear
4. xhigh: if multi-file architecture or hidden assumptions emerge
5. max: if xhigh fails and task is high value/high risk
```

Prompt:

```text
Use the minimum effort that can solve this safely.
Start with a narrow diagnosis. Escalate reasoning only if:
1. the failing test does not identify the cause,
2. the fix touches more than three files,
3. there is security/auth/payment/data-loss risk,
4. two implementation strategies appear plausible.
```

### Pattern C — Effort by subagent role

Current Claude Code docs say skill and subagent frontmatter can set `effort`, overriding the session level when active.

Example subagent taxonomy:

| Subagent | Effort | Reason |
|---|---|---|
| `grep-scout` | `low` | Find files, summarize hits |
| `test-runner` | `low` | Run focused commands, return failures only |
| `doc-reader` | `medium` | Extract relevant docs, avoid over-analysis |
| `bug-hypothesis-agent` | `high` | Compare root causes |
| `architecture-reviewer` | `xhigh` | Deep design reasoning |
| `security-reviewer` | `xhigh` or `max` | High blast radius |
| `migration-planner` | `xhigh` | State transitions and rollback |
| `pr-summarizer` | `medium` | Structured output, not deep invention |

Example frontmatter pattern:

```markdown
---
name: architecture-reviewer
description: Reviews multi-file design, coupling, extensibility, and failure modes.
effort: xhigh
tools: Read, Grep, Glob
---

You are a codebase architecture reviewer. Do not edit files.
Return:
1. architectural map,
2. risk register,
3. competing design options,
4. recommended implementation plan,
5. verification strategy.
```

### Pattern D — “Ultrathink” as one-off override

Current Claude Code docs say including `ultrathink` anywhere in a prompt requests deeper reasoning on that turn without changing the session effort setting; ordinary phrases like “think hard” are not recognized as the same keyword.

Use it sparingly:

```text
ultrathink

We have a production-only race condition in token refresh. Build a causal hypothesis tree, identify discriminating tests, and propose the minimal safe patch. Do not edit yet.
```

Do **not** use `ultrathink` for every prompt. That destroys the cost/latency advantage of adaptive reasoning.

### Pattern E — Token hygiene via skills

Current Claude Code cost guidance says large `CLAUDE.md` files are loaded at session start, while skills load on demand; Anthropic recommends moving specialized workflow instructions into skills and keeping `CLAUDE.md` under 200 lines.

This is directly related to thinking budgets: bloated context steals budget from useful reasoning.

Better architecture:

```text
CLAUDE.md
  - project invariants
  - coding style
  - commands
  - safety constraints

.claude/skills/
  pr-review/
  migration-planner/
  playwright-debug/
  security-audit/
  release-checklist/
  mechanism-space-generator/
```

Keep the base context small. Load specialized reasoning only when needed.

---

## 13. Concrete effort policy for your Claude Code workflows

Use this as a default operating system.

### Default session

```bash
claude --effort xhigh
```

For Opus 4.7 coding/agentic work, current docs recommend `xhigh` as a starting point; Claude Code docs also say the default is `xhigh` on Opus 4.7 as of v2.1.117, and `high` on Opus 4.6/Sonnet 4.6.

### Mechanical task

```bash
claude --effort low
```

Prompt:

```text
Use low effort. Make only the smallest safe change. Do not refactor. Do not inspect unrelated files. Run only the relevant test. Return a terse summary.
```

### Normal feature implementation

```bash
claude --effort high
```

Prompt:

```text
Use high effort. First inspect the relevant code path, then implement incrementally. Prefer existing project conventions. Run focused tests. Stop and ask if the change requires altering public APIs.
```

### Architecture / unfamiliar repo / migration

```bash
claude --permission-mode plan --effort xhigh
```

Prompt:

```text
Use xhigh effort. Plan only. Build a codebase map, identify invariants, risks, and competing strategies. Propose a staged implementation with verification checkpoints. No edits.
```

### Critical production / security / concurrency issue

```bash
claude --effort max
```

Prompt:

```text
Use max effort. This is high-risk. Build a causal model, enumerate failure modes, inspect the minimal relevant code paths, propose tests that distinguish hypotheses, and only then suggest a patch. No broad refactor.
```

---

## 14. The “thinking lever” workflow template

Use this before starting any nontrivial Claude Code session.

```text
Task:
- What outcome do I want?

Risk:
- What breaks if Claude is wrong?

Scope:
- One file, few files, or whole repo?

Intelligence sensitivity:
- Is this mechanical, tactical, architectural, or research-like?

Verification:
- What tests, screenshots, logs, or acceptance criteria prove success?

Effort:
- low / medium / high / xhigh / max

Budget:
- Max time:
- Max files touched:
- Max tool calls, if relevant:
- Max cost, if relevant:

Stop conditions:
- Stop if public API changes are needed.
- Stop if tests reveal unrelated failures.
- Stop if more than N files need edits.
- Stop if assumptions conflict.
```

Example:

```text
Use xhigh effort in plan mode.

Task: migrate our auth refresh logic to avoid duplicate refresh calls.
Risk: high; auth regressions can log users out or create security bugs.
Scope: probably 3–6 files.
Verification: focused auth tests must pass; add a regression test for concurrent refresh.
Budget: plan only first; no edits.
Stop conditions: stop if public API or DB schema changes seem necessary.
```

This is how you convert “thinking budget” into an engineering control surface.

---

## 15. My distilled rules

### Rule 1 — Use effort, not vibes

Do not say:

```text
Be smart. Think carefully.
```

Say:

```text
Use xhigh effort. Plan first. No edits. Compare at least two implementation strategies and choose based on testability, blast radius, and project conventions.
```

### Rule 2 — Match effort to irreversibility

The more irreversible the change, the higher the effort:

| Change type | Effort |
|---|---|
| Comment/doc tweak | `low` |
| Local bugfix | `medium`/`high` |
| Multi-file feature | `high`/`xhigh` |
| Schema migration | `xhigh` |
| Auth/security/payment | `xhigh`/`max` |
| Production incident | `max`, but tightly scoped |

### Rule 3 — Use max as a scalpel, not a default

Max is expensive and can overthink. Use it when:

- `xhigh` failed,
- the task is high-value,
- the failure mode is subtle,
- verification is hard,
- architecture/security/concurrency is involved.

### Rule 4 — Put cheap work in cheap contexts

Use low/medium-effort subagents for:

- grep/search,
- log summarization,
- test execution,
- docs extraction,
- changelog synthesis,
- mechanical code edits.

Preserve your main session’s expensive context for judgment.

### Rule 5 — Evaluate your own repo

Your repo has a characteristic “effort response curve.” Find it empirically.

Run the same task at:

```text
medium → high → xhigh → max
```

Measure:

- correctness,
- files touched,
- tests passed,
- cost,
- latency,
- number of tool calls,
- human cleanup required.

Then set project defaults.

---

## 16. The deepest takeaway

The “thinking lever” is not about making Claude “smarter” in the abstract. It is about **allocating scarce inference-time resources**.

In Claude Code, your job becomes closer to an operating-system scheduler:

- which task deserves Opus?
- which deserves Sonnet?
- which deserves `low`?
- which deserves `xhigh`?
- which deserves `max`?
- which should be delegated to a subagent?
- which should be stopped early?
- which should be forced through plan mode?
- which should be verified by tests before more tokens are spent?

The winning workflow is not “always think more.”

The winning workflow is:

> **Spend intelligence where marginal reasoning reduces expensive human correction, and spend almost nothing where tests, constraints, and narrow scope already control the risk.**
