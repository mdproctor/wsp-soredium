# Retire work_chain.py and lifecycle.py — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development). Steps use checkbox (`- [ ]`) syntax
> for tracking.

**Focal issue:** #592 — Retire work_chain.py and lifecycle.py

**Goal:** Remove two legacy modules that no skill runs any more, relocating the small amount of live functionality they still export.

**Architecture:** `done_fields()` moves inline to ctx.py (its only consumer). `VALID_STATES` moves to plan_io.py (its only consumer). `is_closed()` and `ClosureState` move to work_health.py (their only consumer). Tests that only exercise the legacy modules are deleted. Tests that exercise live wiring are kept and updated to import from the new locations.

**Tech Stack:** Python, pytest

## Global Constraints

- Each deletion is preceded by a full importer search and followed by the full test suite
- Maintainer guide must be updated in the same change

---

## Batch 1: Retire work_chain.py

### Task 1: Move done_fields to ctx.py, delete work_chain.py and legacy tests

**Files:**
- Modify: `project/ctx.py` — inline `done_fields()`, remove work_chain import
- Delete: `project/work_chain.py`
- Delete: `tests/test_work_chain.py` — 21 tests, all exercise `evaluate()` only
- Delete: `tests/test_sync_chain.py` — 6 tests, all exercise `evaluate()` only
- Modify: `tests/test_done_wiring.py` — keep `TestWorkNextMarksDone` and `TestCtxReportsPendingItems`, delete classes that test `evaluate()`

**Interfaces:**
- `done_fields(queue)` signature unchanged, now lives in ctx.py

## Batch 2: Retire lifecycle.py

### Task 2: Move live symbols out of lifecycle.py, delete it and legacy tests

**Files:**
- Modify: `project/plan_io.py` — inline `VALID_STATES` constant, remove lifecycle import
- Modify: `project/work_health.py` — inline `is_closed()` and `ClosureState`, remove lifecycle import
- Modify: `tests/test_plan_io_validation.py` — update import to plan_io
- Delete: `project/lifecycle.py`
- Delete: `tests/test_state_durability.py` — 10 tests, all exercise the lifecycle state machine

## Batch 3: Documentation

### Task 3: Update CLAUDE.md, maintainer guide, and test commands

**Files:**
- Modify: `CLAUDE.md` — Key Skills section (work_chain.py and lifecycle.py entries), developer workflow test commands
- Modify: `docs/development/lifecycle-maintainer-guide.md` — legacy reference line

## References

- [GitHub #592] — focal issue
- [project/work_chain.py] — legacy chaining engine (227 lines)
- [project/lifecycle.py] — legacy state machine (776 lines)
- [project/ctx.py:29] — imports done_fields from work_chain
- [project/work_health.py:19] — imports is_closed, ClosureState from lifecycle
- [project/plan_io.py:144] — imports VALID_STATES from lifecycle
- [docs/development/lifecycle-maintainer-guide.md:272] — legacy module reference
