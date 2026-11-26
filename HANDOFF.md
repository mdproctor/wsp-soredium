# Session Handover

**Date:** 2026-11-26
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`).
**Branch:** `issue-630-fix-work-done` (stacked on `issue-625-post-landing-verify` on `issue-624-landing-gates` on `issue-618-structured-closure-refs`)

## What was done (this session)

**#623 — plan_manager scale/complexity from GitHub labels (on main as df5f6ec):**
- `QueueItem` gains `scale`/`complexity` fields, parsed via `_SCALE_MARKER_RE`
- `_write_item` renders `(S/Med)` inline; `_fetch_issue_scale` queries labels on append
- 12 new tests

**#624 — Landing gates: divergence check, stage invariant, settlement error:**
- Preflight now checks `canonical_sync.follow()` per target — diverged clone blocks ceremony
- Stage invariant: every target must have a non-empty SHA before stamping
- `SettlementError` replaces silent warnings in `settle_permanent_clone`
- 8 new tests; existing diverged test updated to expect failure

**#625 — Post-landing verification and stamp reconciliation:**
- `verify_landing_advanced`: confirms main advanced, checks linear history
- `reconcile_stamps`: verifies closure ref SHAs still reachable from main
- 9 new tests

**#630 — Fix mark_done double-call bug:**
- Guard: refuses when active item has no branch in state (never started)
- Prevents pipeline retry from marking the next item done on the previous branch
- 1 regression test

## What is next

4 done items (#623, #624, #625, #630) are about to be landed by work end.
Next queue item: #634.

## Notes for next session

- Engine and eidos forks still need a one-time manual force-push (from #627)
- Pre-existing test failures in `test_close_integration.py` (TestSlotPromotionGate, TestSquashReviewGate) and `test_close_batch.py` (TestNothingLandsWhenTheBatchIsNotReady) — not caused by this session's changes
