# Session Handover

**Date:** 2026-11-23
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`).
**Branch:** `issue-592-retire-work-chain` (stacked on `issue-627-ci-fork-divergence`, stacked on `issue-590-work-start-create-clone`)

## What was done

**#627 — CI fork divergence fix** (done on branch `issue-627-ci-fork-divergence`)

`mirror()` in `canonical_sync.py` now fetches the fork's main before pushing and returns `"diverged"` when the fork has commits canonical doesn't. The misleading "catches up on next publish" warning replaced with an accurate diagnosis naming the fork's specific divergent commits. CI action on engine/eidos/agency gated to only commit on casehubio repos, not forks.

Cross-repo commits: engine, eidos, agency each got `if: github.repository_owner == 'casehubio'` on `generate-api-docs.yml`. These landed on working branches in those repos — HANDOFF updated in casehub workspace.

1 commit on project branch, 3 cross-repo commits.

**#592 — Retire work_chain.py and lifecycle.py** (done on branch `issue-592-retire-work-chain`)

Deleted both legacy modules (-1,651 lines). Live symbols relocated: `done_fields()` → `ctx.py`, `VALID_STATES` → `plan_io.py`, `is_closed()`/`ClosureState` → `work_health.py`. 37 tests removed, 5 updated. Review caught 4 stale imports outside `project/`/`tests/` (brief.py, two scripts, one handler) — fixed.

4 commits on project branch.

## What is next

**#573 — One structure for slots, clones, and agents** — next in queue. Topic change (structural slot/clone architecture).

## Notes for next session

- Engine and eidos forks still need a one-time manual force-push to clear existing divergence (27 and 1 commits respectively). Commands are in the issue body and design spec.
- The casehub CI fix commits are on working branches in those repos — they need to reach main via work-end in a casehub session to take effect on GitHub.
- Stale `work_chain.py` reference in `work/SKILL.md` line 54 — cosmetic, not filed as an issue.
