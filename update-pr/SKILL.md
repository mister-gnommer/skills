---
name: update-pr
ver: 1
description: only when directly asked via slash-command
---

# update-pr

## Context

User is working on a PR and has changes to commit (by "changes" we mean both staged and unstaged).

The whole skill turns on one question: **has anyone anchored expectations to my current commit layout?**

- **rewritable** — nobody has. There's no PR, or the PR was only touched by the user (or by CI/agents acting on the user's behalf). Commits are a *private draft* — you may rewrite history freely.
- **append-only** — somebody has. There's a PR and someone else already reviewed or commented on it in a meaningful way. Commits are a *shared contract* — never rewrite what they've already seen.

## Establishing the state

1. Is there a PR? **No** → **rewritable**. **Yes** → continue.
2. Does the PR have any review started, or any comments/suggestions? (ignore tech noise about commits, pushes, etc.) **No** → **rewritable**. **Yes** → continue.
3. Is there a review started? **Yes** → **append-only**. **No** → continue.
4. Are all comments from the user? **Yes** → **rewritable**. **No** → continue.
5. Are the non-user comments all AI/agent-generated on the user's request or by the CI/CD flow? **Yes** → **rewritable**. **No** → continue.
6. Otherwise a human review was started, someone else commented, or someone else kicked off an AI review → **append-only**.
7. If you genuinely can't tell (step 5 is the hard one — you often can't prove an agent comment was made on the user's behalf) → **default to append-only, it's the safe choice**. If the situation is outright bizarre, STOP and tell the user something strange is happening.

## Consequence: no force-writing when append-only

> [!IMPORTANT]
> Because someone is relying on the current history, **never** use `--force` or `--force-with-lease` on an **append-only** branch. NEVER!

## Workflow

Once you've established the state, proceed accordingly.

### 1. append-only

Someone already reviewed or touched the PR — *we don't want to break their flow*. **DO NOT AMEND OR REBASE COMMITS**; the only way is a new commit. Be economical — even if changes span different domains, prefer folding them into a single "post-review commit" — but the user decides. If in doubt, ask.

**Escape hatch:** the user may ask you to do post-review cleanup. If they explicitly confirm nobody else will be disrupted, switch to the "perfect world goal" below and act as if the branch were **rewritable**. This is meant for complex PRs with many after-review commits that need consolidating — not for one or two trivial post-review fixes.

### 2. rewritable

There is one **goal**: rearrange everything on this branch/PR so it looks like it was intended from the beginning — as if the change had been part of the initial scaffolding.

Concretely, avoid separate commits for changes that just touch files an existing commit already changed. For example:
- User creates a branch with a feature that takes number X and adds 5.
- Then the user decides it's better to introduce Y and add that instead, so "5" isn't hardcoded.

You should create a **fixup** for the commit that introduced `X + 5` so it becomes `X + Y` — as if "5" never existed.

Steps:
1. Make an inventory of all uncommitted changes.
2. Compare them against the existing commits on the branch.
3. Anything that naturally belongs in an existing commit → commit as a `fixup` for that commit.
4. Everything left → commit as new changes (apply the most appropriate CONTRIBUTING skill/rule).
5. Then run the rebase / autosquash / whatever is needed to land the final, tidy history.
   - Push with `--force-with-lease` (allowed here — this is a private draft).

Make no mistake: a new commit is the escape hatch. Your first reflex should be to fold changes into the existing commit they belong to (via `fixup`), not to add a new one.

## Stacked PRs (gh-stack)

Before pushing, check whether the current branch is part of a stack: run `gh stack view --json` (see the `gh-stack` skill for the full command set). Exit code `2` / "not in a stack" → it isn't; proceed with the normal flow above and stop. Otherwise it is, and you must propagate your change up the stack.

**The state decision still applies per branch.** How you *land* changes on the current branch is governed exactly by the sections above — decide **rewritable** vs **append-only** for the current branch and act accordingly. The stack only adds a propagation step afterwards.

**Propagate upward.** Once your change is committed on the current branch, every branch above it now sits on a stale base and must be replayed onto the new head:

1. `gh stack rebase --upstack` — replays the upstack branches onto your updated branch.
2. `gh stack push` (or `gh stack sync`) — pushes the repositioned branches. gh-stack does this with per-branch `--force-with-lease`.

**Reconciling this with the "never force when append-only" rule.** These are two different operations, and only one is forbidden:

- **Rebasing an upstack branch onto a moved parent is allowed even if that branch is append-only.** In a stack, an upper branch's base *always* moves when a lower branch changes — reviewers of stacked PRs expect exactly this, and the `--force-with-lease` gh-stack uses here is a normal part of that flow, not the history-destroying rewrite the rule forbids.
- **Rewriting an upstack branch's *own* commits (squash / reorder / fixup) is still forbidden if that branch is append-only.** The "perfect world" rewritable cleanup from section 2 may only be applied to an upstack branch if *that* branch is itself rewritable. So before cleaning up commits on any branch other than the one you started on, re-run the state decision for that branch.

If a `--upstack` rebase hits a conflict, follow the conflict-resolution workflow in the `gh-stack` skill (`gh stack rebase --continue` / `--abort`); don't improvise.

## Notes

- Any rule in this skill may be overridden by the user, but only when they ask explicitly.
- This skill can run without a PR — in that case just do the **rewritable** flow and stop before pushing anything that needs a remote.
