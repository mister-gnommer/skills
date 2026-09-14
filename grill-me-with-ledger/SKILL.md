---
name: grill-me-with-ledger
description: A relentless interview (grilling) that offers lettered-choice bulk replies and keeps a running decision ledger. Use when you want a grilling session whose decisions are recorded.
disable-model-invocation: true
---

Run the `grilling` skill first (the skill whose `name` is `grilling`) and follow its instructions in full. Then layer these additions on top of that session. If no `grilling` skill is available, stop and ask me to clarify instead of proceeding.

## Lettered choices for bulk replies

When a question has a closed set of answers (finite, mutually exclusive options), list them as lettered choices (**A** / **B** / **C** / **D** …) so I can reply in bulk like `1 A, 2 A, 3 C`. Prefer this whenever the options are naturally closed; skip lettering for open-ended questions.

## Grilling ledger

Do **not** create a ledger by default. Only start one when I explicitly ask for it **and** point to where it should live (path/file). Once started, keep that file up to date after every round — append new settled questions; mark corrections instead of silently rewriting history.

Use this shape:

```md
## Grilling ledger

Decision record from the planning interview.

- ✅ = final answer explicitly chosen by the user.
- ◐ = final outcome derived from a conditional answer or accepted follow-up.
- ↪ = an earlier answer that was later corrected or superseded.

### Round N — <round theme>

#### Q1 — <question title>

- A — <option>
- ✅ **B — <chosen option>.**
- C — <option>
```

For open-ended answers (no lettered choices), record the chosen wording under ✅ / ◐ the same way. When an earlier answer is superseded, leave it in place marked ↪ and record the replacement under the same question.
