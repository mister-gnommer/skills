# SKILLS

## description

This repo, even if public, is used by the creator mainly as a way to coordinate
and sync skills among different places (work/priv laptop, different projects
and harnesses).
Default way to use it is to run `npx skills` to install and update chosen
skills.

## versioning

No tags/semver: a new version is any pushed commit changing a skill folder.
`npx skills` detects updates via folder hashes; git history is the changelog.

### `ver` field in header

Each skill has `ver` in its mdc header - as described above it is not needed,
but it is useful for human readability.

Agents should keep `ver` bumped (once per commit, not per edit):

- If the agent makes (or helps make) skill changes, bump `ver` automatically,
  without asking.
- If the user only asks to commit and `ver` looks stale, ask whether to bump
  it — never change it on your own in that case.


