# Turn 007 — Default branch flip, branch protection, scanning recs, story branch

**Date:** 2026-05-21
**User intent:** With auth resolved, Ryan asked me to (a) flip the GitHub default branch to `develop`, (b) apply branch protection on both `master` and `develop` via `gh api`, (c) recommend PR vulnerability scanning tools for this stack, and (d) create the `story/3d-engine-design-doc` branch and start setting up the 3D engine design doc with as many clarifying questions as I need.

## Decisions
- **Default branch on GitHub:** flipped from `master` to `develop`. New clones land on the integration branch.
- **Branch protection on `master` and `develop`** (identical rules):
  - Require pull request before merging.
  - `required_approving_review_count: 0` — Ryan is solo, so self-merge after CI is fine.
  - `dismiss_stale_reviews: false`, `require_code_owner_reviews: false`.
  - `allow_force_pushes: false`, `allow_deletions: false`.
  - `required_linear_history: true` — keeps history readable.
  - `required_conversation_resolution: true` — must resolve PR comments before merge.
  - `enforce_admins: false` — admin (Ryan) can override in emergencies without removing the rule.
- **Story branch:** `story/3d-engine-design-doc` created off `develop`. All 3D engine doc work and trailing context updates land here, PR back to `develop`.

## Actions
- `gh api -X PATCH repos/Rearles/TheLivingLand -f default_branch=develop` — succeeded.
- `gh api -X PUT repos/Rearles/TheLivingLand/branches/master/protection --input -` with the rules above — succeeded.
- Same PUT for `develop/protection` — succeeded.
- `git checkout develop && git checkout -b story/3d-engine-design-doc` — branch created locally; not pushed yet.
- Replied to Ryan with a tiered PR-vulnerability-scanning recommendation; nothing in this list was wired up this turn, just surfaced for later:
  - **Tier 1 (free, native, set-and-forget):** Dependabot alerts + version updates, GitHub secret scanning + push protection, CodeQL (csharp + javascript-typescript).
  - **Tier 2 (free / OSS, CI setup needed):** Snyk, Trivy (for the eventual Docker image), OSV-Scanner.
  - **Tier 3 (broader quality + security):** SonarCloud, Semgrep.
  - Honest first move: **Dependabot + CodeQL + secret scanning push protection** when there's actual code to scan. Add the rest if signal/noise feels lacking.
- Wrote this turn file.
- About to ask Ryan a first round of high-impact clarifying questions on the 3D engine doc.

## Pending uncommitted context from earlier turns
The story-branch PR will also pick up:
- `context/turn-005-...md` — modified after the initial commit to mark the push blocker as resolved.
- `context/turn-006-github-auth-setup.md` — created after the initial commit.

These could not be pushed directly to `develop` because protection now blocks direct pushes; bundling them into this PR is the simplest path.

## Open follow-ups
- **Wire up scanning tools** once API and client code exist. Will start with Tier 1.
- **Add a `.github/dependabot.yml`** and `.github/workflows/codeql.yml` in the same PR that scaffolds the API/client projects.
- **3D engine doc** itself — pending answers to the clarifying questions being asked at the end of this turn.
