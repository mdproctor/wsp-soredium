# Session Handover

**Date:** 2026-11-22
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`). The canonical repo `~/claude/hortora/soredium` is the sync point only.
**Branch:** `issue-596-move-on-gate` — done on branch, awaiting `work end`

## What was done

**#596 — work cannot move on until the current item is released** (5 commits)

Separated "carry on" (`work`, `work continue`) from "move on" (`work start`, `work next`). Previously all four aliased to `work.py continue`. Now:

- `work_route.py`: `start`/`next` with an active item return `DECISION=current_item_not_released` with four options (done, deferred, move_down, continue_current)
- `work.py`: `apply_move_on_gate()` handles the user's choice; dirty-tree check blocks the gate when uncommitted changes exist
- `plan_manager.py`: new `defer-active` (removes → deferred section) and `move-down` (repositions in queue) commands
- Handler: `work/handlers/move-on-gate.md`
- 62 tests across 3 files, all passing
- Docs updated: work/SKILL.md, maintainer guide, CLAUDE.md

## Priority

User requested Hortora/soredium#615 be placed at head of queue.

## Queue state

- #596 is done on `issue-596-move-on-gate`, awaiting close ceremony
- #615 should be prioritized to head of queue

## What's next

1. Run `work end` to land #596
2. Reorder queue to put #615 at head
3. Start #615
