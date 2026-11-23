# Clone Guard Decision Flow and Wrap Offer Suppression — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #590 — work start in a canonical repo of a clone-enabled family offers to create the clone
**Issue group:** #590

**Goal:** Make the clone guard a decision flow instead of a stop, and stop offering wrap_or_continue on session start.

**Architecture:** Two independent changes to the work pipeline. The wrap offer suppression changes `work_route.py` to only set `offer=True` on `done` commands, and replaces the `done_items_await_close` stop with an informational route. The clone guard change converts `apply_clone_guard` in `work.py` from `ACTION=stop` to `ACTION=decision` with a handler file.

**Tech Stack:** Python (project/work_route.py, project/work.py), Markdown (handlers/clone-guard.md)

## Global Constraints

- Protocol: a-read-never-writes — the guard detects, it does not create
- Protocol: the-script-that-acts-records-it — clone creation records via worktrace
- Protocol: a-test-must-fail-with-the-rule-removed — test must fail without the guard

---

## Batch 1: Wrap offer suppression

### Task 1: Remove unconditional offer from work_route.py

**Files:**
- Modify: `project/work_route.py:78` — change `offer = bool(done)` to only fire on `done`
- Modify: `project/work_route.py:131-134` — replace stop="end" with informational route
- Test: `tests/test_work_route.py`

**Interfaces:**
- Produces: `Route.offer` is `True` only when `command == "done"` and an active item exists. `Route.done_waiting` is always populated when done items exist. New `Route.stop="nothing"` with `reason="done_items_informational"` replaces the old `stop="end"` path.

- [ ] **Step 1: Write failing tests for offer suppression**

Add to `tests/test_work_route.py`:

```python
class TestOfferOnlyAfterDone:
    def test_continue_with_done_items_does_not_offer(self):
        q = queue(item(1, done_branch="issue-1-x"), active(2))
        r = route("continue", q, "issue-2-y", "main")
        assert not r.offer
        assert r.done_waiting == 1

    def test_start_with_done_items_does_not_offer(self):
        q = queue(item(1, done_branch="issue-1-x"), active(2))
        r = route("start", q, "issue-2-y", "main")
        assert not r.offer

    def test_done_still_offers(self):
        q = queue(active(1), item(2))
        r = route("done", q, "issue-1-x", "main")
        assert r.offer

    def test_only_done_items_remain_does_not_stop(self):
        q = queue(item(1, done_branch="issue-1-x"))
        r = route("continue", q, "main", "main")
        assert r.stop != "end"
        assert r.done_waiting == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_route.py::TestOfferOnlyAfterDone -v`
Expected: FAIL — `offer` is True when it should be False, `stop=="end"` when it shouldn't

- [ ] **Step 3: Implement offer suppression in work_route.py**

Change line 78 from `offer = bool(done)` to remove the unconditional offer. Instead, set `offer=True` only on the `done` command's `mark_done` route (line 88). For all other routes, set `done_waiting` but leave `offer=False`.

In `work_route.py`, replace:

```python
    done = queue.done_items()
    offer = bool(done)
```

with:

```python
    done = queue.done_items()
    n_done = len(done)
```

Then replace every `offer=offer` with `offer=False, done_waiting=n_done` and keep `offer=True` only on the `done`/`mark_done` route at line 88.

For lines 131-134, replace:

```python
        elif done:
            result = Route(stop="end", reason="done_items_await_close",
                           detail=f"{len(done)} finished item(s) are waiting to be closed",
                           offer=True)
```

with:

```python
        elif done:
            result = Route(stop="nothing", reason="done_items_informational",
                           detail=f"{len(done)} finished item(s) waiting; run work end when ready",
                           done_waiting=n_done)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_route.py -v`
Expected: ALL PASS

- [ ] **Step 5: Update existing tests that assert offer=True on non-done routes**

Several existing tests assert `r.offer` is True on continue/start routes with done items (e.g. `test_still_on_a_finished_items_branch_starts_the_next`, `test_skips_done_items_when_choosing`). Update them to assert `not r.offer` and `r.done_waiting > 0`.

Also update `test_on_a_finished_branch_with_nothing_open_goes_to_end` to match the new `stop="nothing"` and `reason="done_items_informational"`.

- [ ] **Step 6: Run full test suite**

Run: `python3 -m pytest tests/test_work_route.py -v`
Expected: ALL PASS

- [ ] **Step 7: Update work.py OFFER emission**

In `project/work.py:468-477`, the `OFFER=` block fires when `r.offer` is True. After the offer suppression, this will only fire on `done` commands. Add a separate `DONE_WAITING=` informational line for non-offer cases:

After the existing `if r.offer:` block, add:

```python
    elif r.done_waiting:
        lines.append(f"DONE_WAITING={r.done_waiting}")
```

This ensures the LLM can mention done items without being gated into the wrap decision.

- [ ] **Step 8: Update work.py stop handling for done_items_informational**

In `project/work.py:500-503`, the stop handler emits `NEXT=work end` for `stop=="end"`. Update to handle the new `stop="nothing"` with `reason="done_items_informational"`:

The existing code at line 501-503:
```python
            lines += ["ACTION=stop", f"DETAIL={r.detail}",
                      "NEXT=work end" if r.stop == "end"
                      else "NEXT=work find, or work start #N"]
```

Since `stop="nothing"` already maps to `"NEXT=work find, or work start #N"` (the else branch), this needs no change — but verify it doesn't break.

- [ ] **Step 9: Run work.py integration tests**

Run: `python3 -m pytest tests/test_work_clone_guard.py tests/test_work_route.py -v`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git add project/work_route.py project/work.py tests/test_work_route.py
git commit -m "fix(route): offer wrap_or_continue only after work done, not on continue/start

OFFER=wrap_or_continue now fires only after `work done` marks an item finished.
For continue/start/resume, done items appear as DONE_WAITING=N (informational)
without gating the user into the wrap-vs-continue decision.

The done_items_await_close stop path becomes done_items_informational with
stop=nothing, so a fresh session is never pushed into work end.

Refs Hortora/soredium#590"
```

---

## Batch 2: Clone guard decision flow

### Task 2: Convert clone guard to ACTION=decision

**Files:**
- Modify: `project/work.py:162-195` — `apply_clone_guard` emits `ACTION=decision` instead of `ACTION=stop`
- Create: `work/handlers/clone-guard.md` — new handler file
- Test: `tests/test_work_clone_guard.py`

**Interfaces:**
- Consumes: `clone_manager.create_clone(family_root, repo)` from `project/clone_manager.py`
- Produces: `ACTION=decision`, `DECISION=clone_guard`, `OPTIONS=create_and_open,stop` or `OPTIONS=open_clone,stop`, `HANDLER=handlers/clone-guard.md`

- [ ] **Step 1: Write failing tests for decision output**

Update `tests/test_work_clone_guard.py`. Change the existing tests that assert `ACTION=stop` to assert `ACTION=decision`:

```python
class TestEnabledFamily:

    @pytest.fixture(autouse=True)
    def _enable(self, family):
        (family / ".clone-enabled").touch()

    @pytest.mark.parametrize("command", ["start", "continue", "next", "done"])
    def test_canonical_on_base_emits_decision_with_open_clone(self, family, command):
        clone = make_clone(family)
        lines, stop = call(family / "P", command)
        assert stop
        assert "ACTION=decision" in lines
        assert "DECISION=clone_guard" in lines
        assert "OPTIONS=open_clone,stop" in lines
        assert any("HANDLER=" in l for l in lines)

    def test_no_clone_yet_emits_decision_with_create(self, family):
        lines, stop = call(family / "P")
        assert stop
        assert "ACTION=decision" in lines
        assert "DECISION=clone_guard" in lines
        assert "OPTIONS=create_and_open,stop" in lines
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_clone_guard.py::TestEnabledFamily::test_canonical_on_base_emits_decision_with_open_clone -v`
Expected: FAIL — still emits `ACTION=stop`

- [ ] **Step 3: Implement decision output in apply_clone_guard**

In `project/work.py:162-195`, replace the `ACTION=stop` output with `ACTION=decision`:

```python
def apply_clone_guard(args: dict[str, str]) -> tuple[list[str], bool]:
    """Stop new work from starting in a canonical repo of a family that uses clones."""
    from work_route import ROUTED
    if args.get("command", "") not in ROUTED or args.get("step_done"):
        return [], False
    if args.get("on_main") != "yes" or not args.get("project"):
        return [], False
    project = Path(args["project"]).resolve()
    family = project.parent
    if (family / ".slot").exists() or not (family / ".clone-enabled").exists():
        return [], False
    clone_dir = family / "clone" / project.name
    handler = str(Path(__file__).resolve().parent.parent / "work" / "handlers" / "clone-guard.md")
    if (clone_dir / ".slot").exists():
        project_clone = clone_dir / project.name
        workspaces = sorted(d for d in clone_dir.iterdir() if (d / ".workspace").exists())
        ws = workspaces[0] if workspaces else None
        lines = [
            "ACTION=decision",
            "DECISION=clone_guard",
            f"DETAIL=work on {project.name} happens in its clone, not in the canonical repo",
            "OPTIONS=open_clone,stop",
            f"HANDLER={handler}",
            f"CLONE={clone_dir}",
            f"PROJECT={project_clone}",
        ]
        if ws:
            lines.append(f"WORKSPACE={ws}")
            lines.append(f"NEXT=open the session in {ws} (workspace) and run work there")
        else:
            lines.append(f"NEXT=open the session in {project_clone} and run work there")
    else:
        script = Path(__file__).resolve().parent / "clone_manager.py"
        lines = [
            "ACTION=decision",
            "DECISION=clone_guard",
            f"DETAIL={family.name} uses clones and {project.name} has none yet",
            "OPTIONS=create_and_open,stop",
            f"HANDLER={handler}",
            f"FAMILY_ROOT={family}",
            f"REPO={project.name}",
            f"CREATE_CMD=python3 {script} create family_root={family} repo={project.name}",
        ]
    return lines, True
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_clone_guard.py -v`
Expected: Most tests pass. Some existing tests may need updating (see Step 5).

- [ ] **Step 5: Update remaining test assertions**

Update tests that assert old output format:
- `test_named_start_is_stopped_too` — verify `ACTION=decision` in output
- `test_clone_without_a_workspace_names_the_project_clone` — verify `NEXT=` still points to project clone
- `test_no_clone_yet_names_the_create_command` → replaced by `test_no_clone_yet_emits_decision_with_create`
- `test_a_directory_without_slot_file_is_not_taken_for_a_clone` — verify `OPTIONS=create_and_open` in output
- `test_through_main_nothing_else_runs` — replace `STOP_REASON` assertion with `DECISION=clone_guard`

- [ ] **Step 6: Run full test suite**

Run: `python3 -m pytest tests/test_work_clone_guard.py -v`
Expected: ALL PASS

- [ ] **Step 7: Create the handler file**

Create `work/handlers/clone-guard.md`:

```markdown
# Handler: clone_guard

The user ran a work command in a canonical repo of a clone-enabled
family. Present the decision based on the OPTIONS line:

## OPTIONS=open_clone,stop

A clone already exists. Present:

1. **Open clone (Recommended)** — the clone is at CLONE= with its
   workspace at WORKSPACE=. Tell the user to open a new session there.
2. **Stop** — do nothing.

After the user chooses:

```
python3 ~/.claude/skills/project/work.py <COMMAND> step_done=clone_guard outcome=completed choice=open_clone
python3 ~/.claude/skills/project/work.py <COMMAND> step_done=clone_guard outcome=completed choice=stop
```

## OPTIONS=create_and_open,stop

No clone exists yet. Present:

1. **Create and open (Recommended)** — creates a permanent clone for
   this repo. The CREATE_CMD= line has the exact command. After
   creation, tell the user to open a new session at the clone workspace.
2. **Stop** — do nothing.

After the user chooses:

```
python3 ~/.claude/skills/project/work.py <COMMAND> step_done=clone_guard outcome=completed choice=create_and_open
python3 ~/.claude/skills/project/work.py <COMMAND> step_done=clone_guard outcome=completed choice=stop
```

Use the COMMAND= value from the route output (the original command the
user typed: start, continue, next, or done).
```

- [ ] **Step 8: Commit**

```bash
git add project/work.py work/handlers/clone-guard.md tests/test_work_clone_guard.py
git commit -m "feat(clone-guard): convert to ACTION=decision with handler

The clone guard now emits ACTION=decision with DECISION=clone_guard and
OPTIONS= instead of ACTION=stop with a command to copy-paste. A new
handler file (clone-guard.md) presents the options with a recommendation.

Two option sets: create_and_open,stop (no clone yet) and open_clone,stop
(clone exists). Both recommend the first option.

Refs Hortora/soredium#590"
```

### Task 3: Update lifecycle maintainer guide and work skill

**Files:**
- Modify: `docs/development/lifecycle-maintainer-guide.md` — update clone guard documentation
- Modify: `work/SKILL.md` — add clone_guard to the ACTION → handler table

**Interfaces:**
- Consumes: Changes from Tasks 1 and 2

- [ ] **Step 1: Update the ACTION → handler table in work/SKILL.md**

Add `clone_guard` to the handler table:

```markdown
| `clone_guard` | Read `handlers/clone-guard.md` |
```

- [ ] **Step 2: Update lifecycle-maintainer-guide.md**

Find the clone guard section and update it to reflect the decision flow instead of the stop behavior.

- [ ] **Step 3: Update CLAUDE.md if needed**

Check if CLAUDE.md references the old `STOP_REASON=canonical_repo_of_clone_family` behavior. If so, update to describe the decision flow.

- [ ] **Step 4: Commit**

```bash
git add work/SKILL.md docs/development/lifecycle-maintainer-guide.md
git commit -m "docs: update clone guard and wrap offer documentation

Reflects the new clone guard decision flow (ACTION=decision instead of
ACTION=stop) and the wrap offer suppression (only after work done).

Refs Hortora/soredium#590"
```

## References

- `specs/issue-590-work-start-create-clone/2026-11-23-clone-guard-and-wrap-offer-design.md` — design spec
- `project/work_route.py:78` — offer = bool(done)
- `project/work_route.py:131-134` — done_items_await_close stop path
- `project/work.py:162-195` — apply_clone_guard
- `project/work.py:468-477` — OFFER emission
- `project/clone_manager.py:51` — create_clone()
- `work/handlers/move-on-gate.md` — model handler
- `docs/protocols/a-read-never-writes.md`
- `docs/protocols/the-script-that-acts-records-it.md`
- `docs/protocols/a-test-must-fail-with-the-rule-removed.md`
- Hortora/soredium#590
