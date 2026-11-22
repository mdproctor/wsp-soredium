# Session Handover

**Date:** 2026-11-22
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`). The canonical repo `~/claude/hortora/soredium` is the sync point only.
**Branch:** `main` in both clone repos (no work branch open)

## What was done

Three issues completed, landed, published, and closed:

1. **#588 — Self-resolved context tests** (1 commit)
   - 10 tests in `tests/test_work_self_resolved.py`
   - Exercises `fill_from_context()` → `ctx.resolve()` → `topology.resolve(cwd)`
   - Three layouts: plain workspace, permanent clone, numbered slot
   - Each includes failed-landing-then-retry

2. **#605 — Silent landing failure detection** (1 commit)
   - `LandResult.before_sha` captures canonical main before landing
   - `land_targets()` verifies `already_landed` claims
   - 4 tests in `tests/test_canonical_sync.py`

3. **#586 — Context budget recommendations** (5 commits)
   - `QueueItem.scale`/`.complexity` fields; `(S/Med)` parsed and rendered
   - `plan_manager.py set-scale` command
   - Pipeline prints NEXT_ITEM/SCALE/COMPLEXITY with OFFER=wrap_or_continue
   - `work/SKILL.md`: three-way recommendation table, mid-task monitoring
   - 10 tests in `tests/test_queue_item_metadata.py`

## Issues filed this session

| # | Title | Scale |
|---|-------|-------|
| #612 | plan_manager append populates scale from labels | S |
| #613 | plan_manager _write_item renders scale explicitly | S |
| #614 | investigate already_landed false positive | S |

## Queue state

- Active: #596 (not started on a branch)
- 32 open items including the 3 just filed
- #589 was deactivated (never branched) — still in queue as open

## What's next

Run `work start` or `work continue` to begin #596.
