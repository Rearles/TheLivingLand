# Turn 009 — Kicking off the doc review

**Date:** 2026-05-22
**User intent:** Ryan opted to review the 3D engine design doc locally before opening the PR. Subsequent turns will be the review itself — each review prompt is its own turn file.

## Decisions
- **Review mode:** local, conversational. The PR opens only after the doc is in a "clean" state Ryan is comfortable with.
- **Ryan drives the review.** Claude proposes a framework (section-by-section walkthrough vs. flag-specific-items vs. start-with-§24-open-questions) and waits for direction.

## Actions
- Created this turn file.
- Sent Ryan a kickoff message offering review framings and pointing at the §24 open questions as the highest-leverage starting point.
- No doc edits yet — those happen as Ryan flags items.

## Open follow-ups
- **Doc review** — ongoing. Each subsequent review prompt gets its own turn file documenting what changed.
- **PR open** — held until Ryan signals the doc is ready.
- **Engine GitHub repo creation** — still deferred to implementation time, after the doc is reviewed and merged into `develop`.
