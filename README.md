# The Thinking Lever — Claude Code Effort & Thinking Budget Notes

Notes distilled from the Anthropic Developer Day talk **"The Thinking Lever"** by Alexander Bricken (Applied AI Research, Anthropic).

**Video:** [https://www.youtube.com/watch?v=T7KqH7kYnE4&t=253s](https://www.youtube.com/watch?v=T7KqH7kYnE4&t=253s)

---

## What this repo contains

Two documents distilled from the video transcript:

### [`claude_thinking_budgets_summary.md`](claude_thinking_budgets_summary.md)

Conceptual summary of the talk — covers:

- What test-time compute is and why it matters
- The three runtime token sinks (thinking, tool calling, text output)
- Effort levels (`low` → `medium` → `high` → `xhigh` → `max`) and when to use each
- Adaptive thinking vs interleaved thinking vs legacy extended thinking
- Why "thinking toggle" is the wrong abstraction
- Larger model + low effort vs smaller model + high effort
- How to evaluate your own repo's effort response curve
- Concrete decision rules and the "thinking lever" workflow template

### [`claude_code_thinking_budget_practitioner_playbook.md`](claude_code_thinking_budget_practitioner_playbook.md)

Practical playbook — 24 ready-to-use workflows:

- Effort routing table by task type
- Plan-high / execute-lower pattern
- Escalation ladder
- Low-effort mechanical patch
- Max-effort scalpel
- `ultrathink` one-turn override
- Budget contracts and stop conditions
- Read-only reconnaissance
- Hypothesis-driven debugging
- Effort-partitioned subagents
- Context hygiene before escalation
- CLAUDE.md minimalism + skills
- Effort eval harness
- Model-effort routing
- PR review effort profile
- Architecture decision record pass
- Cost telemetry loop
- Ready-to-use operating modes (fast / balanced / architecture / incident / PR review / eval)

---

## Core idea

The talk's central point: **thinking is a runtime compute allocation problem**, not a binary on/off switch.

> Stop asking "should I turn thinking on or off?"
> Start asking "how much runtime compute does this task deserve, and where should it go?"

The practical shift for Claude Code users: treat effort like an operating system scheduler. Route each task to the cheapest reasoning regime that can solve it safely. Use tests and stop conditions to make lower effort viable.

---

## Quick reference

| Effort | Use when |
|--------|----------|
| `low` | Mechanical, local, easily verified edits |
| `medium` | Small refactors, simple tests, known patterns |
| `high` | Bugfixes, features across a few files, PR review |
| `xhigh` | Architecture, unfamiliar repo, migrations |
| `max` | Production incidents, security/auth/payment, concurrency bugs |

Set effort in Claude Code with `/effort <level>` or `--effort <level>`.

---

## Source

Both documents were distilled from the YouTube transcript of the video linked above using GPT-5.5.
