# work end step promotion — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #599 — work end runs promotion and squash review as steps
**Issue group:** #599

**Goal:** Move promotion and squash review checks out of `close()` into their own pipeline steps before `land`, add a preflight that reports all blockers at once, and guard `item_finished` against re-answering on retry.

**Architecture:** Extract the unpromoted-artifacts and squash-review checks from `close.py:_close()` into new `scope.py` commands (`promote`, `squash-review`). Wire them as mechanical steps in `_CLOSE_STEPS` before `land`. Extend `sequence_preflight()` to report all blockers upfront. Add a `skip_fn` to `item_finished` that reuses the saved answer on retry.

**Tech Stack:** Python 3.14, pytest, real git repos in tests (no mocks)

## Global Constraints

- All file changes are in `project/` and `work-end/` directories
- Tests use real git repos (fixture pattern from `test_close_integration.py`)
- `close()` is called by `scope.py land` and `audit_work_end.py` — removing checks from `close()` must not break the audit
- `skip_fn` is independent of `skippable` — `skip_fn` is automatic, `skippable` controls LLM opt-out

---

## Batch 1: item_finished guard

### Task 1: Guard item_finished against re-answering on retry

**Files:**
- Modify: `project/work.py:982-988,1042` — add skip_fn to item_finished StepDef
- Test: `tests/test_work_wrap_end.py` (new file)

**Interfaces:**
- Consumes: `Progress.get(ITEM_FINISHED_ANSWER)` from `project/progress.py:115-118`
- Produces: `_item_finished_already_answered(ctx) -> bool` skip function

- [ ] **Step 1: Write failing test — item_finished is skipped on retry when already answered**

Create `tests/test_work_wrap_end.py`:

```python
"""Tests for wrap/end pipeline behavior — item_finished guard (#599)."""
from __future__ import annotations

import os
import subprocess
import sys
from pathlib import Path

import pytest

ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(ROOT / "project"))

from progress import Progress

GIT_ENV = {"GIT_CONFIG_GLOBAL": "/dev/null", "GIT_CONFIG_NOSYSTEM": "1",
           "GIT_AUTHOR_NAME": "T", "GIT_AUTHOR_EMAIL": "t@t",
           "GIT_COMMITTER_NAME": "T", "GIT_COMMITTER_EMAIL": "t@t"}


def g(repo: Path, *args: str) -> str:
    r = subprocess.run(["git", "-C", str(repo), *args], capture_output=True, text=True,
                       env={**os.environ, **GIT_ENV})
    assert r.returncode == 0, f"git {' '.join(args)}: {r.stderr}"
    return r.stdout.strip()


@pytest.fixture
def workspace(tmp_path):
    ws = tmp_path / "workspace"
    ws.mkdir()
    g(ws, "init", "-q", "-b", "main")
    (ws / "base.txt").write_text("base")
    g(ws, "add", "base.txt")
    g(ws, "commit", "-q", "-m", "init")
    return ws


class TestItemFinishedGuard:
    def test_skip_fn_returns_true_when_answer_recorded(self, workspace):
        from work import _item_finished_already_answered, ITEM_FINISHED_ANSWER
        from shared_steps import StepDef

        progress = Progress.load(workspace)
        progress.set(ITEM_FINISHED_ANSWER, "yes")
        progress.save(workspace)

        class FakeCtx:
            def __init__(self, ws):
                self.workspace = ws
                self.progress = Progress.load(ws)

        ctx = FakeCtx(workspace)
        assert _item_finished_already_answered(ctx) is True

    def test_skip_fn_returns_false_when_no_answer(self, workspace):
        from work import _item_finished_already_answered, ITEM_FINISHED_ANSWER

        class FakeCtx:
            def __init__(self, ws):
                self.workspace = ws
                self.progress = Progress.load(ws)

        ctx = FakeCtx(workspace)
        assert _item_finished_already_answered(ctx) is False

    def test_skip_fn_returns_false_for_empty_answer(self, workspace):
        from work import _item_finished_already_answered, ITEM_FINISHED_ANSWER

        progress = Progress.load(workspace)
        progress.set(ITEM_FINISHED_ANSWER, "")
        progress.save(workspace)

        class FakeCtx:
            def __init__(self, ws):
                self.workspace = ws
                self.progress = Progress.load(ws)

        ctx = FakeCtx(workspace)
        assert _item_finished_already_answered(ctx) is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_wrap_end.py::TestItemFinishedGuard -v`
Expected: ImportError — `_item_finished_already_answered` does not exist yet

- [ ] **Step 3: Implement the skip function and wire it to the step**

In `project/work.py`, add the skip function near line 988 (after `_item_not_finished`):

```python
def _item_finished_already_answered(ctx) -> bool:
    answer = ctx.progress.get(ITEM_FINISHED_ANSWER, "")
    if answer in ("yes", "no"):
        print(f"ITEM_FINISHED_REUSED={answer}")
        return True
    return False
```

Change the item_finished StepDef at line 1042 from:

```python
_wrap_step(ITEM_FINISHED, "wrap-item-finished.md", skippable=False),
```

to:

```python
StepDef(ITEM_FINISHED, "wrap", StepType.JUDGMENT,
        action_context_fn=lambda ctx: {"CONTEXT": ITEM_FINISHED},
        handler="handlers/wrap-item-finished.md", skippable=False,
        skip_fn=_item_finished_already_answered),
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_wrap_end.py::TestItemFinishedGuard -v`
Expected: 3 PASS

- [ ] **Step 5: Commit**

```bash
git add project/work.py tests/test_work_wrap_end.py
git commit -m "fix(pipeline): guard item_finished against re-answer on retry

Refs Hortora/soredium#599"
```

---

## Batch 2: Promote and squash_review steps

### Task 2: Add promote command to scope.py

**Files:**
- Modify: `work-end/scope.py:308-313` — add `promote` to command dispatch
- Modify: `work-end/close.py:1118-1124` — remove promotion check from `_close()`
- Test: `tests/test_close_integration.py` — add test for standalone promote check

**Interfaces:**
- Consumes: `close.unpromoted_artifacts(workspace, covers)` from `work-end/close.py:672`
- Produces: `scope.promote(workspace) -> tuple[int, list[str]]` — exit 0 (clean) or exit 2 (ERROR=promotion_needed)

- [ ] **Step 1: Write failing test — scope promote returns promotion_needed**

Add to `tests/test_close_integration.py` (or a new `tests/test_scope_commands.py`):

```python
class TestScopePromote:
    def test_unpromoted_artifacts_returns_exit_2(self, slot_fixture):
        """A workspace with an unpromoted document blocks the promote step."""
        # Create a workspace document that is not in the project
        ws = slot_fixture.workspace
        (ws / "blog").mkdir(exist_ok=True)
        (ws / "blog" / "2026-11-24-entry.md").write_text("---\ntitle: test\n---\nbody\n")
        g(ws, "add", "blog")
        g(ws, "commit", "-q", "-m", "add blog entry")

        code, lines = scope.promote(ws)

        assert code == 2
        assert any("ERROR=promotion_needed" in l for l in lines)
        assert any("PROMOTE_COMMANDS=" in l for l in lines)

    def test_no_unpromoted_returns_exit_0(self, slot_fixture):
        code, lines = scope.promote(slot_fixture.workspace)
        assert code == 0
        assert any("PROMOTED=nothing_needed" in l for l in lines)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_scope_commands.py::TestScopePromote -v`
Expected: AttributeError — `scope` has no `promote` function

- [ ] **Step 3: Implement scope.promote()**

In `work-end/scope.py`, add the function:

```python
def promote(workspace: Path) -> tuple[int, list[str]]:
    """Check for unpromoted artifacts. Returns (exit code, lines).

    Exit 0: nothing to promote. Exit 2: promotion needed with commands.
    """
    queue = ceremony.read_queue(workspace)
    covers = queue.state.covers.split(",")[0].strip() if queue and queue.state.covers else ""
    unpromoted, commands = ceremony.unpromoted_artifacts(workspace, covers)
    if unpromoted:
        return 2, ["ERROR=promotion_needed",
                   "UNPROMOTED=" + " ;; ".join(unpromoted),
                   "PROMOTE_COMMANDS=" + " ;; ".join(commands)]
    return 0, ["PROMOTED=nothing_needed"]
```

Add `"promote"` to the command list at line 310 and add the dispatch block:

```python
if command == "promote":
    code, lines = promote(workspace)
    for line in lines:
        print(line)
    return code
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_scope_commands.py::TestScopePromote -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/scope.py tests/test_scope_commands.py
git commit -m "feat(scope): add promote command for pre-land check

Refs Hortora/soredium#599"
```

### Task 3: Add squash-review command to scope.py

**Files:**
- Modify: `work-end/scope.py` — add `squash-review` command
- Modify: `work-end/close.py:1126-1136` — remove squash check from `_close()`
- Test: `tests/test_scope_commands.py`

**Interfaces:**
- Consumes: `close.resolve(workspace)` for targets, `squash_marker.review_status()` for check
- Produces: `scope.squash_review(workspace) -> tuple[int, list[str]]` — exit 0 or exit 2

- [ ] **Step 1: Write failing test — squash_review returns exit 2 for unreviewed branches**

```python
class TestScopeSquashReview:
    def test_multi_commit_branch_without_review_returns_exit_2(self, slot_fixture):
        """A project branch with multiple commits and no squash review blocks."""
        project = slot_fixture.project
        # Add multiple commits to the working branch
        for i in range(3):
            (project / f"file{i}.txt").write_text(f"content {i}")
            g(project, "add", f"file{i}.txt")
            g(project, "commit", "-q", "-m", f"commit {i}")

        code, lines = scope.squash_review(slot_fixture.workspace)

        assert code == 2
        assert any("ERROR=squash_review_needed" in l for l in lines)

    def test_single_commit_branch_returns_exit_0(self, slot_fixture):
        code, lines = scope.squash_review(slot_fixture.workspace)
        assert code == 0
        assert any("SQUASH_REVIEW=all_clear" in l for l in lines)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_scope_commands.py::TestScopeSquashReview -v`
Expected: AttributeError — `scope` has no `squash_review` function

- [ ] **Step 3: Implement scope.squash_review()**

```python
def squash_review(workspace: Path) -> tuple[int, list[str]]:
    """Check squash review status for all project targets. Returns (exit code, lines).

    Exit 0: all reviewed or single-commit. Exit 2: review needed.
    """
    targets = ceremony.resolve(workspace)
    review_needed = []
    for target in targets:
        if target.is_workspace:
            continue
        reason = squash_marker.review_status(target.repo, target.branch, target.base_branch)
        if reason:
            review_needed.append(f"{target.repo.name}:{target.branch}: {reason}")
    if review_needed:
        return 2, ["ERROR=squash_review_needed",
                   "SQUASH_REVIEW=" + " ;; ".join(review_needed)]
    return 0, ["SQUASH_REVIEW=all_clear"]
```

Add `"squash-review"` to the command list and dispatch.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_scope_commands.py::TestScopeSquashReview -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/scope.py tests/test_scope_commands.py
git commit -m "feat(scope): add squash-review command for pre-land check

Refs Hortora/soredium#599"
```

### Task 4: Wire promote and squash_review into the pipeline and remove from close()

**Files:**
- Modify: `project/work.py:1051-1060` — add steps to `_CLOSE_STEPS`
- Modify: `work-end/close.py:1118-1136` — remove promotion and squash checks from `_close()`
- Modify: `work-end/scope.py:171-177` — remove promotion/squash handling from `land()`
- Test: `tests/test_close_integration.py` — verify close() no longer refuses for unpromoted/squash

**Interfaces:**
- Consumes: `scope.promote`, `scope.squash_review` from Tasks 2-3
- Produces: Updated `_CLOSE_STEPS` with promote → squash_review → land order

- [ ] **Step 1: Write failing test — close() succeeds even with unpromoted artifacts**

The existing test `test_slot_close_refuses_until_the_workspace_document_is_promoted` in `test_close_integration.py` should now PASS (close no longer checks). Write a new test that verifies this:

```python
def test_close_no_longer_refuses_for_unpromoted_artifacts(self, tmp_path):
    """After #599, close() does not check promotion — the pipeline step does."""
    # Setup: workspace with unpromoted document, done item
    # ... (slot fixture with blog entry)
    result = ceremony.close(workspace, land_active=False)
    # close() should not return unpromoted — it was removed
    assert not result.unpromoted
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — close() still checks and returns unpromoted

- [ ] **Step 3: Remove promotion and squash checks from close()**

In `work-end/close.py`, remove lines 1118-1136 (the promotion check block and the squash review block). Both are now handled by upstream pipeline steps.

In `work-end/scope.py`, remove the promotion/squash handling from `land()` (lines 171-177) since those error codes will never be returned by `close()` anymore.

In `project/work.py`, update `_CLOSE_STEPS`:

```python
_CLOSE_STEPS: list[StepDef] = [
    StepDef("promote", "close", StepType.MECHANICAL, script_fn=_scope("promote"),
            requires=(ITEM_FINISHED,)),
    StepDef("squash_review", "close", StepType.MECHANICAL, script_fn=_scope("squash-review"),
            requires=("promote",)),
    StepDef("land", "close", StepType.MECHANICAL, script_fn=_scope("land"),
            requires=("squash_review",)),
    StepDef("promote_backlog", "close", StepType.MECHANICAL,
            script_fn=_scope("promote-backlog", with_covers=True), requires=("land",)),
    StepDef("publish", "close", StepType.MECHANICAL, script_fn=_scope("publish"),
            requires=("land",)),
    StepDef("publish_blog", "close", StepType.MECHANICAL, script_fn=_scope("publish-blog"),
            requires=("promote_backlog",)),
]
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_close_integration.py tests/test_scope_commands.py -v`
Expected: ALL PASS. The existing test that expected close() to refuse for unpromoted artifacts needs updating — it should now expect close() to succeed.

- [ ] **Step 5: Run the full close test suite**

Run: `python3 -m pytest tests/test_close.py tests/test_close_integration.py tests/test_close_batch.py -v`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add project/work.py work-end/close.py work-end/scope.py tests/test_close_integration.py
git commit -m "refactor(pipeline): promote and squash_review are pipeline steps before land

Refs Hortora/soredium#599"
```

---

## Batch 3: Preflight, error codes, audit, and docs

### Task 5: Extend sequence_preflight with PREFLIGHT_WARN

**Files:**
- Modify: `project/work.py:1075-1143` — add blocker scanning to sequence_preflight()
- Test: `tests/test_work_wrap_end.py`

**Interfaces:**
- Consumes: `ceremony.unpromoted_artifacts()`, `squash_marker.review_status()`, `ceremony.resolve()`
- Produces: `PREFLIGHT_WARN=` lines in sequence_preflight output

- [ ] **Step 1: Write failing test — preflight reports blockers for end**

```python
class TestSequencePreflight:
    def test_end_reports_unpromoted_in_preflight_warn(self, workspace_with_unpromoted):
        from work import sequence_preflight
        args = make_end_args(workspace_with_unpromoted)
        lines, stop = sequence_preflight(args)
        assert not stop
        warn_lines = [l for l in lines if l.startswith("PREFLIGHT_WARN=")]
        assert len(warn_lines) == 1
        assert "promotion" in warn_lines[0]

    def test_wrap_does_not_report_preflight_warn(self, workspace_with_unpromoted):
        from work import sequence_preflight
        args = make_wrap_args(workspace_with_unpromoted)
        lines, stop = sequence_preflight(args)
        warn_lines = [l for l in lines if l.startswith("PREFLIGHT_WARN=")]
        assert len(warn_lines) == 0
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — no PREFLIGHT_WARN= lines produced

- [ ] **Step 3: Implement preflight blocker scanning**

In `sequence_preflight()`, after the SCOPE= block (around line 1142), add for `end` only:

```python
if command == "end" and workspace.is_dir():
    blockers = []
    try:
        work_end_dir = str(_WORK_END_DIR)
        if work_end_dir not in sys.path:
            sys.path.insert(0, work_end_dir)
        import close as ceremony_mod
        unpromoted, _ = ceremony_mod.unpromoted_artifacts(workspace)
        if unpromoted:
            blockers.append(f"promotion: {len(unpromoted)} artifact(s)")
        targets = ceremony_mod.resolve(workspace)
        review_needed = sum(1 for t in targets if not t.is_workspace
                           and squash_marker.review_status(t.repo, t.branch, t.base_branch))
        if review_needed:
            blockers.append(f"squash: {review_needed} branch(es)")
    except Exception:
        pass
    if blockers:
        lines.append("PREFLIGHT_WARN=" + "; ".join(blockers))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_wrap_end.py::TestSequencePreflight -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add project/work.py tests/test_work_wrap_end.py
git commit -m "feat(preflight): report all close blockers before judgment steps

Refs Hortora/soredium#599"
```

### Task 6: Fix exit_1 error codes at land

**Files:**
- Modify: `work-end/scope.py:154-180` — refine error codes in `land()`
- Modify: `work-end/close.py` — ensure all error paths produce non-empty strings
- Test: `tests/test_scope_commands.py`

**Interfaces:**
- Consumes: `CloseResult.errors` from close.py
- Produces: Refined `ERROR=land_<specific>` codes with non-empty `DETAIL=`

- [ ] **Step 1: Write failing test — land returns specific error codes**

```python
class TestLandErrorCodes:
    def test_land_error_always_includes_detail(self, slot_fixture_with_errors):
        code, lines = scope.land(slot_fixture_with_errors.workspace)
        assert code != 0
        error_lines = [l for l in lines if l.startswith("ERROR=")]
        detail_lines = [l for l in lines if l.startswith("DETAIL=")]
        assert len(error_lines) == 1
        assert len(detail_lines) == 1
        assert detail_lines[0] != "DETAIL="  # not empty
```

- [ ] **Step 2: Run test, verify fail, implement, verify pass**

Ensure every `CloseResult(success=False, errors=...)` path in `close.py` produces a non-empty error string. In `scope.py land()`, always emit `DETAIL=` with content.

- [ ] **Step 3: Commit**

```bash
git add work-end/scope.py work-end/close.py tests/test_scope_commands.py
git commit -m "fix(land): specific error codes with non-empty detail

Refs Hortora/soredium#599"
```

### Task 7: Audit worklog runs and update lifecycle guide

**Files:**
- Modify: `docs/development/lifecycle-maintainer-guide.md` — update step lists, error table

- [ ] **Step 1: Audit the two worklog runs**

```bash
python3 project/worktrace.py show action=step limit=100
```

Filter for the two runs named in the issue:
- `casehub/platform` 2026-11-20 06:43-06:51
- `casehub/pages` 2026-11-19 13:00-13:04

Report: which items had `item_finished finished=yes` answered multiple times, and whether each marked item was actually worked on.

- [ ] **Step 2: Update lifecycle-maintainer-guide.md**

Update:
- Step list at line 143 to include `promote` and `squash_review` between `commit_queue` and `land`
- Error table at lines 184-188 to remove `promotion_needed` and `squash_review_needed` from `land` errors (they're now separate steps)
- Section 4.4 notes about the new preflight PREFLIGHT_WARN= behavior
- Add note about the item_finished guard

- [ ] **Step 3: Commit**

```bash
git add docs/development/lifecycle-maintainer-guide.md
git commit -m "docs: update lifecycle guide for #599 step promotion

Refs Hortora/soredium#599"
```

## References

- [2026-11-24-work-end-step-promotion-design.md] — design spec
- `project/work.py:970-1067` — wrap/end pipeline structure
- `work-end/close.py:1044-1163` — close ceremony
- `work-end/scope.py:154-180` — land command
- `work-end/close.py:672-690` — unpromoted_artifacts()
- `project/shared_steps.py:35-56` — StepDef class
- `project/orchestrator_engine.py:279-280` — skip_fn handling
- `project/progress.py:115-118` — Progress.get/set
- `docs/development/lifecycle-maintainer-guide.md:170-189` — ceremony docs
- Hortora/soredium#599
