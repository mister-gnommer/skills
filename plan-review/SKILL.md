---
name: plan-review
description: >-
  Multi-model plan review via 1–3 subagents. Use ONLY when the user explicitly
  asks to start plan review, review the plan, or similar while working on an
  implementation plan — never auto-invoke.
disable-model-invocation: true
---

# Plan Review

**Announce at start:** "I'm using the plan-review skill to review this plan."

## When to Use

**ONLY** when the user explicitly requests plan review (e.g. "start plan review", "review the plan", "run plan review").

Do **not** auto-invoke after writing a plan. Do **not** substitute this for the self-review in `writing-plans`.

## Core Principle

**Under any circumstances, never skip even one subagent finding — including the stupidest ones.**

Every finding must be tracked, addressed in the plan, or explicitly pushed back with reasoning. No silent dismissals. No "that's obvious" shortcuts.

---

## Step 1: Decide Review Panel Size

| Complexity | Agents | When |
|------------|--------|------|
| **1** | 1 reviewer | Single file/area, small scope, low risk |
| **2** | 2 reviewers | Multiple files or subsystems, moderate risk |
| **3** | 3 reviewers | Cross-cutting change, new patterns, high risk, or many rules/skills involved |

When uncertain, prefer **more** reviewers over fewer.

---

## Step 2: Select Models

Use **different but comparable** models across reviewers. Resolve tier → model via the global main rule (`low-tier`, `medium-tier`, `high-tier`, `highest-tier`); do not hardcode model names here. At dispatch, pick an explicit slug from the Task tool's allowed list.

| Reviewer | Tier |
|----------|------|
| Reviewer 1 (always) | Same tier as the agent that wrote the plan |
| Reviewer 2 (if used) | `medium-tier` |
| Reviewer 3 (if used) | `low-tier` |

**Rules:**
- Reviewer 1: match the planner's tier; prefer a different model family when possible
- If reviewer 1 would be `high-tier` or `highest-tier`, ask first (global main rule) — requesting plan review alone does not authorize those tiers
- Reviewers must still differ from each other; never assign the same model to two reviewers
- Without permission, reviewers 2–3 stay on `medium-tier` / `low-tier` only

---

## Step 3: Prepare Context Packet

Build one shared context block. Attach the **full plan** (path or inline). Include:

```markdown
## Plan Review Context

### Business requirements
[What problem this solves, user-facing behavior, success criteria]

### Engineering requirements
[Constraints, non-functional reqs, compatibility, rollout, observability]

### Important input
[Spec excerpts, prior decisions, user preferences, known risks, open questions]

### Plan under review
Path: `[exact/path/to/plan.md]`
(or paste full plan if not saved yet)

### Scope summary
- Files/systems touched: [...]
- Estimated complexity: [low | medium | high]
- Why N reviewers: [1–3 sentence justification]
```

Do **not** dump full session history. Give reviewers what they need to evaluate the plan, not your reasoning process.

---

## Step 4: Dispatch Review Subagents

Launch **1–3 subagents in parallel** (one message, multiple `Subagent` calls). Use `subagent_type: "generalPurpose"` or a domain-specific type when clearly better (e.g. `security-reviewer`, `architect`, `code-reviewer`).

Each subagent prompt must include the context packet plus:

```markdown
You are Plan Reviewer [N] ([model name]). Review this implementation plan — do NOT implement it.

## Your job
1. **Scope & changes** — What will be created, modified, deleted? Missing files? Wrong boundaries?
2. **Rules** — Check ALL applicable rules:
   - User rules (global)
   - Project rules (`.cursor/rules/`, `AGENTS.md`, workspace rules)
   - Directory-scoped rules for every path the plan touches
3. **Skills & agents** — Identify every skill/subagent the plan should reference or invoke during execution (e.g. `preflight-unit-tests-check`, `review-security`, `preview-env-curl-testing`, `security-reviewer`). Flag missing or wrong ones.
4. **Plan quality** — Gaps vs requirements, placeholders/TBDs, task ordering, test strategy, rollback, edge cases.
5. **Risk** — What could go wrong? What's underspecified?

## Output format (required)

### Summary
[2–3 sentences]

### Findings
Number every finding. Use severity:

- 🔴 **Blocker** — Plan must change before execution
- 🟡 **Important** — Should fix; execution risk if ignored
- 🟢 **Minor** — Nice to fix; low risk

For each finding:
- **ID:** R[N]-F[number]  (e.g. R1-F3)
- **Severity:**
- **Category:** scope | rules | skills | testing | architecture | requirements | other
- **Finding:** [specific issue]
- **Evidence:** [quote plan section or rule/skill name]
- **Recommendation:** [concrete fix]

### Strengths
[What the plan does well]

### Verdict
[Approve | Revise | Block] — one sentence why
```

Assign each reviewer a **different focus angle** when using 2–3 agents, e.g.:
- **Reviewer 1:** requirements coverage + architecture
- **Reviewer 2:** rules, skills, conventions
- **Reviewer 3:** testing, edge cases, operational risk

---

## Step 5: Synthesize — Mandatory Finding Ledger

When all reviewers return, create a **Finding Ledger** before changing anything:

```markdown
| ID | Reviewer | Severity | Summary | Disposition |
|----|----------|----------|---------|-------------|
| R1-F1 | Reviewer 1 | 🔴 | ... | pending |
| R2-F1 | Reviewer 2 | 🟡 | ... | pending |
```

**Disposition** must be exactly one of:
- **Applied** — Plan updated; note where
- **Pushback** — Not applied; user must see why (see below)
- **Duplicate** — Same as R1-F3; merged into that row

### Rules for synthesis

1. **Every row must reach a final disposition.** No `pending` when you respond to the user.
2. **Never skip a finding** — including minor, pedantic, or seemingly wrong ones.
3. **Duplicates still count** — merge in the ledger; do not drop.
4. If reviewers disagree, note both views; default to the stricter recommendation unless pushback is justified.

---

## Step 6: Improve Plan or Push Back

### If findings warrant changes

Update the plan inline. For each **Applied** finding, show what changed (brief diff or section reference). Re-run a **focused** re-review only if blockers remain or the plan changed materially.

### If pushing back on a finding

Pushback must be **explicit and visible** to the user:

```markdown
## Pushback: [ID] — [one-line summary]

**Reviewer said:** [quote or paraphrase finding]

**Why not applied:** [technical reasoning — cite rule, codebase fact, or requirement]

**Risk accepted:** [what we accept if we skip this]
```

Never implement pushback silently. Never omit pushback from the user-facing summary.

---

## Step 7: Report to User

Use this structure:

```markdown
## Plan Review Complete

**Panel:** [N] reviewers — [models used]
**Verdict:** [Revised plan ready | Blocked — needs input | Approved with pushbacks]

### Finding ledger
[Full table with dispositions]

### Plan changes made
[Bullet list, or "None"]

### Pushbacks
[All pushbacks in full, or "None"]

### Updated plan
[Path or inline summary of key revisions]

### Open questions (if any)
[Items needing user decision before execution]
```

Then ask whether to proceed with execution (if applicable).

---

## Red Flags — Never Do These

- Skip review because the plan "looks fine"
- Drop findings as "nitpicks" or "obviously wrong" without ledger entry + pushback
- Dispatch fewer reviewers than complexity warrants to save time
- Give subagents session history instead of a curated context packet
- Auto-run this skill without explicit user request
- Present a revised plan without showing the finding ledger

## Integration

- **Before this skill:** `writing-plans` (or equivalent) produces the plan
- **After revised plan approved:** offer `executing-plans` or `subagent-driven-development`
