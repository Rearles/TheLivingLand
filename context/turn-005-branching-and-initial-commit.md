# Turn 005 — Branching setup + initial commit + push

**Date:** 2026-05-21
**User intent:** Ryan asked to commit the work-in-progress (design + context folders) to the GitHub remote. Interrupted partway through to add: set up GitFlow-style branching (`master` = production, `develop` = development, story branches → PR to develop), commit and push the initial state to both `master` and `develop`. Then ask about the 3D engine design doc.

## Decisions
- Branching model is master/develop with short-lived `story/*` (or `feature/*`, `fix/*`) branches PR'd into `develop`. Releases are merge-and-tag from `develop` to `master`. Hotfixes branch off `master` and merge back into both.
- Default branch on GitHub should be `develop` (so new clones land on the integration branch, not production). Setting this and branch-protection rules requires GitHub web UI or `gh api` calls — surfaced as a recommendation, not done automatically.
- This initial commit lands on `master` because the repo had no commits yet; `develop` is created at the same revision immediately afterward so the two branches start identical.

## Actions
- Wrote `CONTRIBUTING.md` at the repo root documenting:
  - branch roles (`master`, `develop`, `story/*`, `feature/*`, `fix/*`, `hotfix/*`)
  - standard workflow (branch from develop → PR back to develop)
  - release workflow (merge develop → master + tag)
  - naming conventions and commit-message expectations
  - recommended GitHub branch-protection settings
- Created this turn file before the commit so the commit captures the full state of the conversation.
- Git operations performed:
  - `git add CONTRIBUTING.md docs context` (specific paths, not `-A`).
  - `git commit -m "initial pass with design folder and context folder"` with Co-Authored-By Claude. Commit hash: `d177d42`.
  - `git branch develop` (creates `develop` at the same commit as `master`).
  - `git push -u origin master` — **failed**: `fatal: could not read Username for 'https://github.com': Device not configured`.
  - `git push -u origin develop` — not attempted because master push failed.

## Push blocker

The remote at `origin` is `https://github.com/Rearles/TheLivingLand.git` (HTTPS). The environment has:
- no `gh` CLI installed
- no SSH keys in `~/.ssh/` (only `known_hosts`)
- no git credential helper configured
- no `GH_TOKEN` / `GITHUB_TOKEN` env var
- no `~/.netrc`

So there's no credential path to GitHub from this machine. Ryan needs to pick one:
1. **`gh` CLI** (recommended): `brew install gh && gh auth login` — also handles HTTPS git pushes after `gh auth setup-git`.
2. **SSH keys**: `ssh-keygen -t ed25519`, add the public key to GitHub, then switch the remote to `git@github.com:Rearles/TheLivingLand.git`.
3. **Personal access token + keychain**: create a PAT on GitHub, then `git config --global credential.helper osxkeychain` and push once to cache.

Once auth is set up the push is a single `git push -u origin master && git push -u origin develop`.

## Open follow-ups
- **Push to GitHub** — resolved in [turn-006](./turn-006-github-auth-setup.md). Local repo is in the correct state and pushed.
- **Branch protection** on GitHub: needs to be enabled via the web UI or `gh api`. Not done this turn.
- **Default branch** on GitHub: should be flipped from `master` (GitHub's repo default) to `develop`. Web UI or `gh api repos/Rearles/TheLivingLand -X PATCH -f default_branch=develop`. Offer to Ryan.
- **The Living Land.pdf** is referenced throughout the docs but is not in the working tree — it was supplied via the chat attachment. Worth checking with Ryan whether to commit a copy so future Claude Code sessions can read it directly.
- Next planned artifact: `docs/3d-engine-design.md`. Will be done on a `story/*` branch off `develop` and PR'd in, per the new workflow.
