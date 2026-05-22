# Turn 006 — GitHub auth setup + initial push

**Date:** 2026-05-21
**User intent:** Resolve the push blocker from turn 005 so the initial commit reaches the GitHub remote. Ryan picked the gh-CLI path. The actual setup required several pivots because Homebrew wasn't installed and then Command Line Tools were too old.

## Decisions
- **Auth path:** gh CLI over HTTPS (Ryan's pick from the original two-option list).
- **CLT blocker workaround:** download the gh binary directly from GitHub releases instead of `brew install gh`. The brew compile failed because the system's Xcode Command Line Tools are out of date and Monterey is now a "Tier 3" brew configuration without prebuilt bottles for gh's `go` dependency. Direct binary install sidesteps the whole CLT chain.
- **Install location:** `~/.local/bin/gh`. PATH updated in `~/.bash_profile` so Ryan's interactive shell picks it up.
- **Default branch flip and branch protection on GitHub:** still deferred — surfaced again at the end of this turn.

## Actions
- `brew` itself was installed by Ryan in his terminal (initial run command from the runbook).
- Attempted `brew install gh` — failed with "Your Command Line Tools are too outdated."
- Pivoted: downloaded `gh_2.92.0_macOS_amd64.zip` from GitHub releases via `curl`. First attempt stalled at ~10.5 MB; killed and retried with `--connect-timeout 30 --max-time 600 --retry 3 -C -` (resume). Second attempt completed at 14.9 MB.
- Extracted; moved `gh` to `~/.local/bin/gh`, chmod +x.
- Edited `~/.bash_profile` to add `export PATH="$HOME/.local/bin:$PATH"` so future shells see it.
- Ryan ran `gh auth login --hostname github.com --git-protocol https --web` in his terminal and completed the device-code flow in his browser. Confirmed: "gh authed". `gh auth status` reports `Logged in to github.com account Rearles (keyring)`, token scopes `gist, read:org, repo, workflow`.
- Ran `gh auth setup-git` to install the gh credential helper for git.
- `git push -u origin master` → success, branch tracking set.
- `git push -u origin develop` → success, branch tracking set. Both branches now visible at https://github.com/Rearles/TheLivingLand.

## Open follow-ups
- **GitHub default branch** still points at `master`. Recommended one-liner: `~/.local/bin/gh api -X PATCH repos/Rearles/TheLivingLand -f default_branch=develop`. Offered to Ryan but awaiting his OK.
- **Branch protection** on `master` and `develop`: not yet enabled. Doable via `gh api` or the GitHub web UI; the API call is fiddly (requires multiple fields) and easier to get right via the web UI.
- **Command Line Tools** are still outdated. Not a blocker for current work, but `brew install` of compile-from-source formulas will keep failing until CLT is updated. Worth fixing eventually with `sudo rm -rf /Library/Developer/CommandLineTools && sudo xcode-select --install`.
- **Next planned artifact:** `docs/3d-engine-design.md`. Per the new branching workflow, this lives on `story/3d-engine-design-doc` (branched from `develop`) and will PR into `develop` when done.
