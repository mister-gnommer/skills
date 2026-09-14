---
name: parallel-plan
ver: 1
description: Creates implementation plans designed for multiple parallel AI agents using exclusive file ownership, dependency waves, integration barriers, parallel verification, and centralized fixes. Use when the user explicitly asks for a parallel plan, multi-agent plan, fan-out/fan-in workflow, or invokes /parallel-plan.
disable-model-invocation: true
---

# Parallel Implementation Plan

Create an implementation-ready plan that uses meaningful agent parallelism without introducing edit races.

## Core principles

1. Explain the feature before explaining agent orchestration.
2. Parallelize independent files and concerns, not arbitrary subtasks.
3. Give every concurrent writer an exclusive file allowlist.
4. Place a single-writer integration barrier after concurrent edits.
5. Tell lane agents that peer scopes are not their responsibility; the integrator will reconcile all lanes.
6. Run independent verification lanes concurrently and keep them source-read-only.
7. Consolidate fixes through one writer, then rerun affected checks in parallel.
8. Use an Economy-by-default model budget and never inherit the parent's model for fan-out workers.
9. Run only explicitly named unit-test files during the agent workflow; leave broad suites to CI unless the user authorizes escalation.
10. Prefer the fewest agents that capture the natural parallelism. More agents are not automatically better.

## Planning workflow

### 1. Understand the implementation

Read the request, source specification, applicable rules, and relevant code. Identify:

- required behavior and acceptance criteria;
- existing patterns and helpers to reuse;
- files that will be created or modified;
- API/data/privacy/backward-compatibility constraints;
- tests and verification commands;
- dependencies between files and tasks.

Ask only questions whose answers materially change the contract or task graph.

### 2. Write the implementation narrative first

The plan body must begin in this order:

```markdown
# [Feature]

## What is being implemented and why
[User/business problem, intended behavior, scope, compatibility.]

## How: [contract/architecture]
[Response shape, data flow, boundaries, major decisions.]

## How: [implementation details]
[Files, symbols, algorithm, safety, tests.]
```

Do not begin with waves, agents, task tables, or orchestration mechanics.

### 3. Find natural parallel lanes

Build a file-ownership map. A lane is safe to run concurrently when:

- it writes files no other active lane writes;
- its contract is fully specified in the plan;
- it can tolerate peer files temporarily not existing;
- it does not need repository-wide formatting or automatic fixes;
- integration can resolve temporary import/type mismatches afterward.

Typical lanes include:

- DTO/schema/contract;
- application service;
- service tests;
- controller plus controller tests;
- module wiring plus a focused wiring test;
- independent adapters or migrations when they touch separate files.

Keep implementation and tests in separate lanes only when the method signatures, fixtures, and expected outcomes are precise enough for both agents to work independently.

### 4. Define a focused test budget

The plan must enumerate exact unit-test files that cover the changed behavior and compatibility boundary. During implementation and verification:

- run tests by explicit file path using the framework's exact-file option, such as Jest `--runTestsByPath`;
- include directly modified specs and specifically named neighboring regression specs when the change affects their contract;
- never invoke a package-wide, project-wide, workspace-wide, or complete unit-test suite by default;
- never run integration, end-to-end, preview, or environment-backed suites unless they are explicit acceptance criteria or the user authorizes them;
- do not use coverage mode unless requested;
- leave broad suite coverage to CI.

Escalate narrowly: when a targeted test exposes a dependency issue, add only the specific relevant unit-test file. For shared foundational changes where focused tests cannot provide reasonable confidence, explain the risk and ask before running a broader suite. Do not silently broaden test scope.

### 5. Assign an Economy-by-default model budget

Do not hardcode model names here or in the generated plan. Use the tier names from the global main rule (`low-tier`, `medium-tier`, `high-tier`, `highest-tier`) — that rule is the source of truth for which models map to each tier. At execution, pick an explicit model slug from the Task tool's allowed list for every model-backed subagent.

Use this default allocation:

| Work | Model budget |
|---|---|
| Mechanical DTO/schema, controller, module wiring, straightforward tests | `low-tier` |
| Core business logic, ambiguous implementation, difficult test design | `medium-tier` |
| Integration and cross-lane fixes | Current parent agent; do not launch another agent |
| Final contract/code review | At most one `medium-tier` reviewer (ask before `high-tier` / `highest-tier`) |
| Tests, build, typecheck, lint, formatting checks | Direct tools; no model-backed subagent |

Fan-out workers must never use `inherit`. The user's selected planning model may be `high-tier` / `highest-tier`, and inheritance would multiply that cost across every lane. If the requested `low-tier` / `medium-tier` model is unavailable, do not silently escalate workers; choose the nearest lower-cost suitable model or ask the user.

Hard guardrails:

- without explicit user permission, only launch `low-tier` or `medium-tier` subagents (same as the global main rule);
- never launch more than one `high-tier` / `highest-tier` subagent unless the user explicitly opts in;
- keep worker prompts and context packets limited to the plan plus their lane;
- use the parent as integrator and fixer so it can reuse existing context without another expensive run;
- record the tier for every implementation lane in the plan's ownership table.

### 6. Define dependency waves

After the implementation narrative, add:

```markdown
## Parallel agent execution waves
```

Use this default fan-out/fan-in structure:

#### Wave 1 — Parallel implementation

- Launch all independent writers in one parallel dispatch.
- Specify each lane's subagent type, explicit model tier, exclusive files, deliverable, dependencies, and—where applicable—the exact unit-test files it owns or affects.
- Give every agent the complete plan plus its lane contract.
- Agents may read anything but must edit only allowlisted files.
- Agents must not inspect, coordinate, repair, or wait for peer lanes. Their sole responsibility is their own deliverable.
- Agents may briefly report a cross-lane assumption or required change in their handoff, but must not implement it. The integrator owns reconciliation after every lane finishes.
- Agents must not run repository-wide formatters or broad auto-fixes.
- If the user explicitly requests parallel agents, dispatch them in a single message with multiple subagent calls.

Use local agents sharing the working tree by default. Use cloud agents or isolated worktrees only when the user explicitly requests them or overlapping experiments genuinely require isolation.

Include this instruction in every parallel implementation-agent prompt:

```text
Your sole responsibility is this lane and its allowlisted files. Do not inspect,
coordinate, fix, or wait for other agents' scopes. Temporary cross-lane errors
are expected. After all lanes finish, the parent agent will integrate and reconcile
interfaces and fix combined-scope issues. Mention any assumption in your
handoff, but do not edit outside your allowlist.
```

#### Wave 2 — Integration barrier

The current parent agent becomes the sole writer across all changed files. Do not launch a separate integrator subagent. The parent must:

- inspect the combined diff and detect out-of-allowlist changes;
- align imports, types, fixtures, and module wiring;
- resolve temporary cross-lane compilation issues;
- format only the changed files;
- preserve acceptance criteria and avoid weakening tests;
- avoid expanding scope.

#### Wave 3 — Parallel verification

Run deterministic command lanes as direct parallel tool calls, not `shell` subagents:

- explicitly named unit-test files using the framework's exact-file option;
- typecheck;
- build/compile;
- formatting and lint checks.

These commands must not edit source. A build may write ignored build output if it does not race with another build. State exact commands and expected scope.

In parallel with the direct commands, launch at most one read-only contract, architecture, privacy, security, or code reviewer on `medium-tier` (ask before `high-tier` / `highest-tier`). Do not create a separate agent for each review angle; choose one reviewer type and give it a combined checklist.

If build, typecheck, and targeted tests contend heavily for CPU or memory, split command lanes into two parallel batches. Treat resource contention separately from code failures.

#### Wave 4 — Centralized fixes

The current parent agent remains the sole writer and receives all verification results. Do not launch a separate fixer subagent. After fixes:

- rerun affected checks in parallel;
- repeat until all required checks pass;
- leave no unresolved correctness, privacy, or compatibility review findings.

### 7. Make ownership explicit

Use a task table:

```markdown
| Lane | Subagent | Model budget | Exclusive write ownership | Deliverable |
|---|---|---|---|---|
| 1A — Contract | `senior-engineer` | `low-tier` | `path/to/dto.ts` | Typed contract |
| 1B — Service | `senior-engineer` | `medium-tier` | `path/to/service.ts` | Business logic |
| 1C — Tests | `senior-engineer` | `low-tier` | `path/to/service.spec.ts` | Outcome matrix |
```

For every wave, state:

- its start barrier;
- maximum useful concurrency;
- explicit model tier for each model-backed lane;
- files owned by each writer;
- forbidden overlaps;
- completion condition;
- the next barrier.

### 8. Choose agents proportionally

Recommended roles:

- `senior-engineer`: scoped production implementation and test authoring;
- direct Shell/tool calls: build, test, typecheck, lint, and formatting checks without a subagent;
- `code-reviewer` or `pragmatic-code-reviewer`: the single read-only final reviewer;
- `security-reviewer`: use instead of the general reviewer when the scope genuinely requires a security audit, not alongside it unless the user explicitly requests both;
- `explore` or `architect`: research and design while creating the plan, not as an implementation lane.

The current parent agent performs integration and fixes. Do not create multiple agents that all need to edit the same core service. Do not split tiny files solely to increase agent count.

## Required plan sections

Produce sections in this order:

1. What is being implemented and why
2. How the contract/architecture works
3. How implementation and tests will work
4. Parallel agent execution waves
5. Verification and completion criteria
6. Rollback or compatibility notes when relevant

Include wave-aware todos in plan frontmatter when the plan format supports todos.

## Conflict checklist

Before finalizing, verify:

- [ ] Every parallel writer has exclusive file ownership.
- [ ] Every lane prompt says peer scopes are not its responsibility and names the later integrator.
- [ ] Every fan-out worker has an explicit `low-tier` or `medium-tier` and does not inherit the parent model.
- [ ] No `high-tier` / `highest-tier` subagent is planned without explicit user permission.
- [ ] Every test command names exact unit-test files and uses exact-file selection.
- [ ] No full unit, integration, end-to-end, or coverage suite runs without explicit authorization.
- [ ] No formatter or auto-fixer runs during the parallel edit wave.
- [ ] Shared method names and DTO shapes are fully specified.
- [ ] Existing files with high conflict risk have one owner.
- [ ] The parent is the sole integration and fix writer.
- [ ] Deterministic verification uses direct parallel tools, not subagents.
- [ ] The optional final reviewer is the only model-backed verification agent.
- [ ] Heavy commands have a resource-contention fallback.
- [ ] The plan still reads naturally without the orchestration section.
- [ ] Agent count reflects useful parallelism rather than novelty.

## Anti-patterns

Do not:

- start the plan with the agent-wave breakdown;
- assign two active agents to the same file;
- let fan-out workers inherit a potentially `high-tier` / `highest-tier` parent model;
- launch model-backed agents merely to run deterministic shell commands;
- launch multiple `high-tier` / `highest-tier` reviewers for separate review angles;
- escalate workers past `medium-tier` without asking;
- run a broad test target when exact unit-test files are known;
- run integration, end-to-end, preview, or coverage suites as routine verification;
- ask every implementation agent to run global tests or formatting;
- let multiple verification agents apply fixes independently;
- hide dependencies behind vague phrases such as "wire everything together";
- use parallelism where a short prerequisite would remove substantial ambiguity;
- require cloud agents when disjoint local file ownership is sufficient;
- declare success before integration and all required verification complete.

## Updating this skill

When the user identifies an improvement while reviewing or executing a parallel plan:

1. Update this `SKILL.md`, not only the current plan.
2. Preserve broadly useful lessons; do not encode repository-specific filenames or domain rules.
3. Keep instructions concise and under 500 lines.
4. Summarize the changed principle so the user can validate it.
