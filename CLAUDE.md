# Upstream Sync Policy

This repo is a fork of https://github.com/obra/superpowers.

The upstream remote is configured as `upstream`. **Never merge or pull from upstream automatically.**

To bring in upstream changes:
1. `git fetch upstream` — safe, never modifies working tree
2. `git log upstream/main --oneline --not master` — review what's new
3. `git cherry-pick <sha>` — explicitly pick what you want

Nothing from upstream enters this repo without explicit, per-commit approval.
