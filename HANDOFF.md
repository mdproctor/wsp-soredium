# Session Handover

**Date:** 2026-11-22
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`). The canonical repo `~/claude/hortora/soredium` is the sync point only.
**Branch:** main — #615 landed

## What was done

**#615 — CRITICAL: create-slot can overwrite an existing slot** (1 commit after squash)

Slot 227 was destroyed when `allocate_slot_number` reused its number based on DB state without checking disk. `shutil.rmtree` with `ignore_errors=True` silently deleted the directory containing 9 commits of active work.

Three layered guards added to `work-slot/slot_lifecycle.py`:
- `_slot_dir_has_content()` — distinguishes real work (git repos, files) from creation debris (`.m2`, `.mvn`, Maven wrapper)
- `_safe_remove_debris()` — replaces bare `shutil.rmtree`, refuses to delete dirs with content
- `create_slot()` guard — rejects if allocated slot dir already exists on disk after allocation

7 tests in `tests/test_slot_overwrite_guard.py`, all existing 143 slot lifecycle tests pass.

Diary entry: "The Line That Deletes Your Work" in `docs/blog/2026-11-22-mdp02-the-line-that-deletes-your-work.md`.

## What's next

Next queue item: #583 (Worklog trace gaps found by review) — different domain, fresh session recommended.
