# Contributing to The Living Land

## Branching model

We use a long-lived `master` + `develop` split with short-lived story branches.

| Branch     | Purpose                                  | Who merges in?                               |
| ---------- | ---------------------------------------- | -------------------------------------------- |
| `master`   | Production. Always release-ready.        | Only `develop` (or `hotfix/*`) via PR.       |
| `develop`  | Integration. The next release lives here.| Story / feature / fix branches via PR.       |
| `story/*`  | A single unit of work (one PR's worth).  | Branched off `develop`, merged back via PR.  |
| `feature/*`| Synonym for `story/*`; pick whichever reads better for the change. | Same as above. |
| `fix/*`    | A bug fix targeting `develop`.           | Same as above.                               |
| `hotfix/*` | Urgent prod fix branched off `master`.   | Merged into both `master` and `develop`.     |

## Standard workflow

```bash
# Start a new piece of work
git checkout develop
git pull origin develop
git checkout -b story/<short-slug>

# ... commit work ...

git push -u origin story/<short-slug>
# Open a PR on GitHub targeting `develop`.
# After review + merge, delete the story branch.
```

## Release workflow

When `develop` is release-ready:

```bash
git checkout master
git pull origin master
git merge --no-ff develop
git tag -a vX.Y.Z -m "Release vX.Y.Z"
git push origin master --tags
```

## Branch naming

- `story/3d-engine-design-doc` — a unit of design or implementation work.
- `feature/shop-page` — a user-facing feature.
- `fix/run-heartbeat-timeout` — a bug fix.
- `hotfix/auth-token-expiry` — production hotfix.

Use lowercase kebab-case after the prefix. Keep slugs short and descriptive.

## Commit messages

- Imperative mood: "add shop endpoint", not "added" or "adds".
- Reference the design doc or PDF section when relevant.
- Co-author Claude when the work was done with Claude Code:
  ```
  Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
  ```

## Pull requests

- Target `develop` (or `master` only for hotfixes / release merges).
- Title: short, imperative, < 70 chars.
- Body: what changed and why; a brief test plan.
- Link to relevant section of [docs/](./docs/) or the design PDF.
- One reviewer is enough for a passion project; the goal is a clean history, not bureaucracy.

## Recommended GitHub settings

These need to be set in the GitHub web UI (or via `gh api`):

- `master`: protected. No direct pushes. Require PR. Require linear history.
- `develop`: protected. No direct pushes. Require PR.
- Default branch: `develop` (so new clones land there, not on `master`).
