# Claude Code Practitioner Playbook: Thinking Budgets, Effort Levels, and Runtime Compute

## 0. Operating principle

Claude Code is an **agentic coding tool**: it can read your codebase, edit files, run commands, and integrate with development tools. That makes effort control operationally important: you are not merely choosing “how much Claude thinks,” you are choosing **how much runtime compute to allocate to planning, tool calls, code inspection, edits, tests, and final explanation**.

The session’s core claim: Claude improves when it can spend more test-time compute, but the gains trade off against token cost and latency. The speaker breaks runtime compute into three sinks: **thinking space, tool calling, and text output**. Effort and budgets are the control surfaces for that runtime compute.

Use this playbook as a decision system:

```text
Hard / ambiguous / high-risk task     → higher effort, plan first, strong verification
Narrow / mechanical / low-risk task   → lower effort, constrained prompt, focused test
Unfamiliar repo / architecture        → xhigh plan mode before edits
Production/security/concurrency       → max or xhigh, but with strict stop conditions
```

---

## 1. Effort routing table

Claude Code currently exposes effort levels such as `low`, `medium`, `high`, `xhigh`, and `max`, depending on the model. The docs describe effort as controlling adaptive reasoning: lower effort is faster and cheaper for straightforward work, while higher effort gives deeper reasoning for complex work. Claude Code supports setting effort through `/effort`, `/model`, the `--effort` flag, or the `CLAUDE_CODE_EFFORT_LEVEL` environment variable.

| Task type | Default effort | What | How | Why |
|---|---:|---|---|---|
| One-line fix | `low` | Small local edit | Constrain scope; run one focused test | Avoid overthinking and tool sprawl |
| Rename / formatting / lint cleanup | `low` | Mechanical transformation | “Smallest safe diff; no refactor” | Cheap, fast, easily verified |
| Summarize logs / classify failures | `low` | Extraction task | Pipe logs; ask for top causes only | Intelligence demand is low |
| Small bugfix | `medium` or `high` | Local diagnosis + patch | Inspect local code path; patch; test | Some reasoning, but bounded |
| Feature across 2–5 files | `high` | Implementation task | Read relevant files; plan briefly; implement | Balanced speed and intelligence |
| Unfamiliar repo exploration | `xhigh` | Codebase mapping | Plan/read-only first | Prevent wrong edits from shallow understanding |
| Architecture refactor | `xhigh` | Design-heavy change | Compare alternatives; stage migration | Requires abstraction and invariants |
| Auth/payment/security | `xhigh` or `max` | High blast-radius work | Plan, threat model, tests, rollback | Failure is expensive |
| Race condition / concurrency bug | `max` initially | Causal diagnosis | Hypothesis tree; discriminating tests | Subtle failures need deep reasoning |
| PR review | `high` or `xhigh` | Review/judgment task | Review diff, tests, risks | Judgment matters more than speed |
| Emergency production issue | `max`, tightly scoped | Incident reasoning | Root cause, mitigation, rollback | Cost of wrong answer dominates token cost |

Rule of thumb from the session: **extra high / xhigh is the Pareto default for serious Claude Code work; max helps on the hardest tasks but can show diminishing returns.**

---

## 2. Workflow: Plan high, execute lower

### What

Use high or `xhigh` effort for **planning**, then lower the effort for **implementation** once the plan is constrained.

### How

```bash
claude --effort xhigh
```

Prompt:

```text
Use xhigh effort. Plan only. Do not edit files.

Goal:
[describe the task]

Produce:
1. relevant code paths,
2. architectural constraints,
3. implementation options,
4. recommended plan,
5. tests to run,
6. stop conditions.
```

After reviewing the plan:

```text
/effort high
```

Then:

```text
Implement exactly the approved plan.
Do not revisit architecture unless a test failure contradicts the plan.
Keep the diff minimal.
Run the listed tests.
```

### Why

Planning has high information value. Implementation often has lower marginal reasoning value once the correct path is known. This separates **judgment** from **execution**.

Claude Code’s own workflow docs recommend planning before editing for safe analysis, and describe plan mode as useful for exploring codebases, planning complex changes, or reviewing safely with read-only operations.

---

## 3. Workflow: Escalation ladder

### What

Start with the cheapest effort level likely to solve the task, then escalate only when evidence demands it.

### How

Use this ladder:

```text
low → medium → high → xhigh → max
```

Prompt:

```text
Use the minimum effort needed to solve this safely.

Start with a narrow diagnosis.
Escalate effort only if:
1. the root cause is not local,
2. the patch touches more than 3 files,
3. tests fail in a non-obvious way,
4. security/auth/payment/data-loss risk appears,
5. there are multiple plausible architectures.
```

### Why

Most Claude Code waste comes from using deep reasoning on shallow tasks. Escalation preserves quality while avoiding automatic max-effort runs.

The session explicitly frames effort as a speed/intelligence/token tradeoff and says low effort is suitable when work is not intelligence-bound, while max is reserved for genuinely hard tasks.

---

## 4. Workflow: Low-effort mechanical patch

### What

Use `low` effort when the task is constrained, local, and externally verifiable.

### How

```text
/effort low
```

Prompt:

```text
Use low effort.

Make the smallest safe change to fix the issue.
Do not refactor.
Do not inspect unrelated files.
Do not improve style unless required.
Run only the most relevant test.
Return:
1. files changed,
2. exact fix,
3. test result.
```

Good uses:

```text
Fix this TypeScript error.
Update this import path.
Add the missing null check.
Rename this variable consistently in this file.
Summarize this failing test output.
```

### Why

Low effort is not “bad mode.” It is **latency-sensitive, scope-constrained mode**. The session notes that low effort can sometimes find direct, efficient strategies because it is constrained from over-deliberating.

---

## 5. Workflow: Max-effort scalpel

### What

Use `max` only for genuinely hard, high-value, high-risk tasks.

### How

```text
/effort max
```

Prompt:

```text
Use max effort.

This is high-risk. Do not edit yet.

First produce:
1. causal model,
2. competing hypotheses,
3. evidence needed to distinguish them,
4. minimal files to inspect,
5. tests or probes to run,
6. safest patch strategy,
7. rollback plan.

Stop before editing if the root cause is still uncertain.
```

Use for:

- production-only bugs,
- race conditions,
- subtle state corruption,
- auth/session bugs,
- data migrations,
- security-sensitive logic,
- financial/payment flows,
- large architectural rewrites.

### Why

Max can improve hard-task performance, but both the transcript and current Claude Code docs warn about diminishing returns and overthinking. The docs explicitly say `max` can help on demanding tasks but should be tested before broad adoption.

---

## 6. Workflow: `ultrathink` one-turn override

### What

Use `ultrathink` for a single deep-reasoning turn without permanently changing session effort.

### How

```text
ultrathink

We have a production-only cache invalidation bug.
Do not edit.
Build a hypothesis tree and identify the minimum discriminating tests.
```

### Why

Claude Code docs say `ultrathink` is recognized as a keyword requesting deeper reasoning on that turn, without changing the session effort setting. Other phrases like “think hard” are ordinary prompt text, not the same control.

Use it when you need one burst of reasoning, not a full session at max effort.

---

## 7. Workflow: Budget contract

### What

Tell Claude not just the effort level, but the **budget envelope**: time, files, tests, scope, and stop conditions.

### How

Prompt template:

```text
Effort: high

Budget:
- Max files to edit: 3
- Max commands to run before reporting back: 5
- Max scope: auth refresh path only
- No public API changes
- No dependency changes
- No schema changes

Stop if:
- more than 3 files appear necessary,
- test failures are unrelated,
- root cause is ambiguous,
- changing public behavior seems required.
```

### Why

Effort is a soft control. Budgets impose operational discipline. The transcript distinguishes effort from stricter budgets such as token or task budgets.

This is especially important in Claude Code because Claude can read files, edit files, and run commands. Constraints prevent “helpful” overreach.

---

## 8. Workflow: Read-only reconnaissance

### What

Before a serious edit, run a read-only mapping pass.

### How

```text
Use xhigh effort. Read-only reconnaissance only.

Map:
1. entry points,
2. relevant modules,
3. data flow,
4. invariants,
5. existing tests,
6. likely change surface.

Do not edit.
Do not run broad test suites yet.
Return a concise implementation map.
```

### Why

Most bad agentic coding failures are not syntax failures. They are **codebase-understanding failures**. Reconnaissance spends reasoning on orientation before touching disk.

Claude Code’s workflow docs specifically list codebase exploration, finding relevant code, fixing bugs, refactoring, testing, PRs, plan-before-editing, subagent research, and script piping as common development workflows.

---

## 9. Workflow: Hypothesis-driven debugging

### What

Force Claude to debug like a scientist: hypotheses first, tests second, patch last.

### How

```text
Use high effort.

Debug this failure using a hypothesis-driven loop.

Steps:
1. State the observed failure.
2. List 3 plausible root causes.
3. Rank them by probability.
4. Identify the cheapest evidence for each.
5. Inspect only the files needed.
6. Run one discriminating test or command.
7. Patch only after the leading hypothesis is supported.
```

For harder bugs:

```text
/effort xhigh
```

For race conditions:

```text
/effort max
```

### Why

Without this structure, Claude may jump from symptom to patch. Hypothesis-driven debugging uses thinking budget where it matters: choosing the correct causal model.

---

## 10. Workflow: Test-gated implementation

### What

Make tests the arbiter of whether Claude continues.

### How

```text
Use high effort.

Implement the feature in small steps.
After each meaningful change:
1. run the narrowest relevant test,
2. summarize pass/fail,
3. only continue if the result supports the plan.

Do not run the full suite unless the focused tests pass.
```

### Why

Tests reduce the need for unlimited reasoning. A good verification loop lets you use **less effort safely**.

This is the compute-efficient pattern:

```text
less speculative reasoning + more external verification
```

---

## 11. Workflow: Effort-partitioned subagents

### What

Assign different effort levels to different roles.

### How

Use a role map like this:

| Role | Effort | Job |
|---|---:|---|
| `grep-scout` | `low` | Find candidate files |
| `log-summarizer` | `low` | Extract failure signatures |
| `test-runner` | `low` | Run commands and report failures |
| `doc-reader` | `medium` | Summarize relevant docs |
| `bug-diagnoser` | `high` | Compare root causes |
| `implementation-agent` | `high` | Patch scoped code |
| `architecture-reviewer` | `xhigh` | Review design choices |
| `security-reviewer` | `xhigh` or `max` | Threat model and blast radius |
| `migration-planner` | `xhigh` | Stage schema or API migration |

Prompt example:

```text
Spawn a low-effort research pass:
Find all files related to refresh-token handling.
Return only paths and one-line relevance notes.
Do not analyze architecture yet.
```

Then:

```text
Use xhigh effort.
Given the candidate files, produce an architecture-level diagnosis.
```

### Why

Do not waste `xhigh` reasoning on grep. Do not use `low` effort for architecture. Partitioning keeps the main context cleaner and the cost curve flatter.

Claude Code’s cost docs also warn that agent teams spawn separate Claude Code instances with separate context windows, so token usage scales with active teammates and duration. Keep spawned work focused.

---

## 12. Workflow: Context hygiene before effort escalation

### What

Before increasing effort, reduce irrelevant context.

### How

Use:

```text
Before escalating effort, identify which context is irrelevant.
Summarize only the necessary facts into a compact working brief.
Then continue from that brief.
```

Or:

```text
Create a compact task brief:
1. goal,
2. files involved,
3. known facts,
4. failed attempts,
5. current hypothesis,
6. next action.
Then clear irrelevant discussion and proceed.
```

### Why

High effort over bad context is expensive garbage collection. Context size drives token cost, and Claude Code docs explicitly recommend managing context proactively, choosing the right model, reducing MCP overhead, offloading work to hooks/skills, delegating verbose operations, and writing specific prompts to reduce token use.

---

## 13. Workflow: CLAUDE.md minimalism + skills for procedures

### What

Keep `CLAUDE.md` for stable project invariants. Move long procedures into skills.

### How

Put this in `CLAUDE.md`:

```markdown
# Project invariants

- Use pnpm, not npm.
- Run `pnpm test:unit` before final response.
- Do not alter public API without asking.
- Prefer existing repository patterns over new abstractions.
- For migrations, always produce rollback steps.
```

Move this kind of material into skills:

```text
release checklist
security review procedure
migration planning protocol
React component QA protocol
Playwright debugging protocol
mechanism-space generator
```

### Why

Claude Code memory docs say each session starts with a fresh context window and that CLAUDE.md files and auto memory carry knowledge across sessions. They also advise putting repeated project facts in CLAUDE.md, while moving multi-step procedures or path-specific material into skills or path-scoped rules.

This directly improves effort efficiency: fewer always-loaded instructions means more budget for useful reasoning.

---

## 14. Workflow: Effort eval harness

### What

Empirically determine which effort levels work for your repo.

### How

Create a small benchmark folder:

```text
.claude/evals/
  bugfix-auth-refresh.md
  refactor-api-client.md
  add-cache-expiry-test.md
  review-payment-pr.md
  migrate-config-loader.md
```

Each eval file should contain:

```markdown
# Eval: bugfix-auth-refresh

## Goal
Fix duplicate refresh-token calls under concurrent requests.

## Constraints
- No public API changes.
- No schema changes.
- Minimal diff.

## Acceptance
- Existing auth tests pass.
- Add one regression test.
- No change to login/logout semantics.

## Score
- Correctness: 40
- Minimality: 15
- Test quality: 20
- Safety: 15
- Explanation: 10
```

Run the same task at:

```bash
claude --effort medium
claude --effort high
claude --effort xhigh
claude --effort max
```

Track:

| Metric | Why it matters |
|---|---|
| Time to first good patch | Latency |
| Total token/cost | Economics |
| Tests passed | Correctness |
| Files touched | Blast radius |
| Human corrections | Real productivity |
| Regression quality | Long-term value |
| Overengineering score | Max-effort failure mode |

### Why

The transcript says evals are the best way to find the right balance and recommends using hard representative tasks to determine effort.

Do not choose effort philosophically. Measure it.

---

## 15. Workflow: Model-effort routing

### What

Choose both model and effort based on intelligence sensitivity.

### How

Use this policy:

| Task | Model/effort posture |
|---|---|
| Simple extraction | Smaller/faster model, low effort |
| Mechanical edit | Lower effort |
| Normal coding | Strong model, high effort |
| Architecture | Strongest available model, xhigh |
| Subtle bug | Strongest available model, xhigh/max |
| Security/payment/auth | Strongest available model, xhigh/max |
| Batch chores | Cheaper model/low effort/subagent |

### Why

The transcript argues that if a task needs real intelligence, a larger model at low effort may beat a smaller model at high effort. Smaller models are best for low-intelligence tasks where the outcome is simple and easy to verify.

Practical rule:

```text
Do not buy more thinking for a weak model when the task requires deep abstraction.
Use the stronger model, then control cost with scope and verification.
```

---

## 16. Workflow: PR review effort profile

### What

Use effort based on PR risk, not PR size alone.

### How

Prompt:

```text
Use high effort for this PR review.

Review for:
1. correctness,
2. test coverage,
3. regressions,
4. security/privacy risk,
5. unnecessary complexity,
6. mismatch with project conventions.

Return:
- blocking issues,
- non-blocking suggestions,
- tests that should be added,
- files requiring human review.
```

Escalate:

```text
Use xhigh effort.
This PR touches auth/payment/security/concurrency.
Perform a risk-centered review and threat model.
```

### Why

A small PR in auth can be riskier than a large documentation PR. Effort should track **blast radius**, not line count.

---

## 17. Workflow: Architecture decision record pass

### What

For nontrivial design choices, make Claude produce an ADR before coding.

### How

```text
Use xhigh effort. Produce an ADR before implementation.

ADR format:
1. Context
2. Decision
3. Alternatives considered
4. Consequences
5. Migration plan
6. Test strategy
7. Rollback plan

No edits yet.
```

### Why

ADR mode forces Claude to expose design tradeoffs before turning them into code. It is especially useful for solo projects, where Claude otherwise becomes both architect and implementer without review.

---

## 18. Workflow: “No broad refactor” guardrail

### What

Explicitly prevent Claude from expanding scope.

### How

```text
Use high effort.

Fix the bug without broad refactoring.
Allowed:
- small local helper,
- one regression test,
- minimal condition change.

Forbidden:
- new framework,
- new dependency,
- public API change,
- unrelated cleanup,
- style-only edits outside touched code.
```

### Why

High-effort Claude may become ambitious. Guardrails preserve the economic value of higher reasoning without allowing architectural drift.

---

## 19. Workflow: Cost telemetry loop

### What

Track cost and token usage as part of your Claude Code practice.

### How

Use `/usage` during or after sessions. Claude Code docs say `/usage` provides token usage statistics for the current session and an estimated local dollar figure, while authoritative billing lives in the Console.

Session review template:

```text
After completing this task, report:
1. effort level used,
2. number of files changed,
3. tests run,
4. approximate token/cost from /usage if available,
5. whether this task could have used lower effort next time.
```

Maintain a simple log:

```text
date | task | effort | model | cost | time | outcome | lower-effort next time?
```

### Why

Without telemetry, “use xhigh” becomes a habit rather than a policy. You want an empirical cost-quality frontier.

---

## 20. Workflow: Long-horizon task budget

### What

For tasks that could run for hours or multiple sessions, define task-level spending rules.

### How

```text
Use xhigh effort.

Long-horizon budget:
- First 30 minutes: reconnaissance and plan only.
- No edits until plan accepted.
- Maximum first-pass diff: 5 files.
- Run focused tests before full suite.
- Produce checkpoint summary after each phase.
- Stop if architecture uncertainty remains after reconnaissance.
```

### Why

The transcript’s ideal future state is that you set a time/cost/task budget and Claude allocates compute appropriately. You can approximate that today with explicit phase budgets and stop conditions.

---

## 21. Ready-to-use Claude Code operating modes

### Mode 1: Fast mechanical mode

```text
/effort low

Make the smallest safe change.
No refactor.
No unrelated file inspection.
Run only the relevant test.
Return a terse summary.
```

### Mode 2: Balanced coding mode

```text
/effort high

Inspect the relevant code path.
Make a short plan.
Implement incrementally.
Run focused tests.
Keep the diff minimal.
Report changed files and test results.
```

### Mode 3: Architecture planning mode

```text
/effort xhigh

Plan only. No edits.

Map the codebase area, invariants, risks, alternatives, recommended approach, test plan, and rollback plan.
```

### Mode 4: High-risk incident mode

```text
/effort max

Do not edit yet.

Build a causal model, hypothesis tree, discriminating tests, minimal inspection set, mitigation plan, patch plan, and rollback plan.
Stop if evidence is insufficient.
```

### Mode 5: PR review mode

```text
/effort high

Review this diff for correctness, regressions, tests, security/privacy, complexity, and project convention violations.
Separate blocking issues from suggestions.
```

### Mode 6: Eval mode

```text
/effort xhigh

Run this task as an eval.
Track:
- final correctness,
- tests,
- files changed,
- time,
- token/cost estimate,
- whether lower effort would likely have sufficed.
```

---

## 22. Minimal `CLAUDE.md` snippet for effort discipline

Add something like this to your project-level `CLAUDE.md`:

```markdown
# Claude Code operating policy

## Effort policy

- Use low effort for mechanical, local, easily verified edits.
- Use medium/high effort for normal bugfixes and feature work.
- Use xhigh effort for architecture, unfamiliar code, migrations, and multi-file changes.
- Use max effort only for high-risk production, security, auth, payment, concurrency, or data-loss tasks.

## Planning policy

- For complex or risky work, plan before editing.
- No edits during reconnaissance.
- State assumptions, risks, implementation plan, and tests before changing files.

## Scope policy

- Prefer minimal diffs.
- Do not refactor unrelated code.
- Do not change public APIs, schemas, dependencies, or auth/payment behavior without explicit approval.

## Verification policy

- Run the narrowest relevant tests first.
- Add regression tests for bugfixes when practical.
- Report commands run and results.

## Stop conditions

Stop and ask before continuing if:
- the root cause is uncertain,
- more than 3 files need unexpected edits,
- a public API or schema change seems required,
- tests fail for unrelated reasons,
- security or data-loss risk appears.
```

---

## 23. The practitioner’s decision tree

```text
Is the task mechanical and local?
  → low effort

Is the task normal coding with clear tests?
  → high effort

Is the repo area unfamiliar?
  → xhigh, read-only plan first

Does the change alter architecture, schema, auth, payment, security, concurrency, or data integrity?
  → xhigh or max, plan first, stop conditions

Did high/xhigh fail because the root cause is subtle?
  → max for one diagnostic pass

Is Claude spending too much?
  → lower effort, shrink context, use subagents, improve tests, add budget contract

Is Claude making shallow mistakes?
  → raise effort or use stronger model

Is Claude overengineering?
  → lower effort or add “minimal diff / no refactor” guardrails
```

---

## 24. Bottom line

Your Claude Code workflow should not be:

```text
Use the best model and maximum thinking for everything.
```

It should be:

```text
Route each task to the cheapest reasoning regime that can solve it safely.
Use high effort for judgment.
Use low effort for mechanics.
Use xhigh for architecture.
Use max only when the cost of being wrong dominates the cost of tokens.
Constrain everything with scope, tests, and stop conditions.
```

That is the practical meaning of thinking budgets for Claude Code.
