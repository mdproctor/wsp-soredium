# Session Handover

**Date:** 2026-11-24
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`).
**Branch:** `issue-599-work-end-step-promotion` — landed, published, closed

## What was done (this session)

**#632 — Stale workspace branch alignment repair:**
- `project_behind` is now repairable when the workspace branch is stale (not in queue)
- Queue gate in `apply_alignment()`: only repairs when branch is not the queue's active branch
- 4 tests in `test_alignment_repair.py`, 2 tests in `test_work_alignment_gate.py`

**#599 — work end runs promotion and squash review as steps:**
- `promote` and `squash_review` mechanical steps added to `_CLOSE_STEPS` before `land`
- New `scope.py promote` and `scope.py squash-review` commands
- Steps use `done_items()` and `batch_entries()` for correct branch/covers resolution after `mark_done`
- `sequence_preflight()` extended with `PREFLIGHT_WARN=` reporting all blockers before judgment steps
- `item_finished` skip_fn guard: reuses saved answer on retry, prints `ITEM_FINISHED_REUSED=<answer>`
- `land()` exit_1 fix: DETAIL is never empty
- Lifecycle maintainer guide updated
- Design spec and diary entry promoted

**#634 filed:** Investigate how workspace and project branches diverge (the root cause behind #632's safety net)

**Deferred:** Worklog audit of two specific runs (data aged out of trace; root cause fixed by the guard)

## What is next

Next open queue item. The pipeline restructure is done — #599 and #632 are landed.

## Notes for next session

- The pipeline now stops at `promote` or `squash_review` before reaching `land` — no more double-retry pattern
- `sync-local` was run mid-session to install the new code for the ceremony to use
- Engine and eidos forks still need a one-time manual force-push (from #627)
