# Move-On Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #596 — work cannot move on until the current item is done, deferred or moved down

**Goal:** Separate "carry on" (`work continue`) from "move on" (`work start`, `work next`) so that moving on requires explicitly releasing the current item first.

**Architecture:** Three changes: (1) `work_route.py` stops aliasing `start`/`next` to `continue` when an active item exists and instead returns a gate decision; (2) `work.py` adds a dirty-tree check before the gate and a handler for the four options; (3) `plan_manager.py` gets two new commands (`defer-active`, `move-down`) that release the active item and activate the next. `done` becomes "mark done and stop" — it no longer starts the next item (that's what `start`/`next` are for now).

**Tech Stack:** Python 3.12, pytest, `work_route.py`, `work.py`, `plan_manager.py`

## Global Constraints

- All new `.py` scripts require tests (protocol: externalised-scripts-require-tests)
- `route()` must remain pure — no git operations, no side effects
- Tests use the existing `queue_io.Queue`/`QueueItem` helpers — no git repos needed for route tests
- Plan_manager tests that write files use `tmp_path`

---

## Batch 1: Route separation — start/next no longer alias continue

### Task 1: Make `route()` distinguish start/next from continue

**Files:**
- Modify: `project/work_route.py`
- Modify: `tests/test_work_route.py`

**Interfaces:**
- Produces: `route()` returns `Route(stop="decision", reason="current_item_not_released", ...)` when command is `start` or `next` and active item exists

- [ ] **Step 1: Write failing tests for the gate**

Replace `TestOneCommand.test_every_spelling_gives_the_same_route` and
`TestActiveItem.test_next_is_another_spelling_of_continue` with tests
that assert `start`/`next` return a decision when an active item exists:

```python
class TestMoveOnGate:
    """start and next mean 'move on' — they gate on active work."""

    def test_start_with_active_item_returns_decision(self):
        q = queue(active(1), item(2))
        r = route("start", q, "issue-1-x", "main")
        assert (r.stop, r.reason) == ("decision", "current_item_not_released")
        assert r.issue == "o/r#1"

    def test_next_with_active_item_returns_decision(self):
        q = queue(active(1), item(2))
        r = route("next", q, "issue-1-x", "main")
        assert (r.stop, r.reason) == ("decision", "current_item_not_released")

    def test_continue_with_active_item_still_carries_on(self):
        q = queue(active(1), item(2))
        r = route("continue", q, "issue-1-x", "main")
        assert (r.run, r.reason) == ("continue", "active_work")

    def test_start_no_active_item_starts_next(self):
        q = queue(closed(1), item(2))
        r = route("start", q, "main", "main")
        assert r.run == "start" and r.activate == "o/r#2"

    def test_next_no_active_item_starts_next(self):
        q = queue(closed(1), item(2))
        r = route("next", q, "main", "main")
        assert r.run == "start" and r.activate == "o/r#2"

    def test_start_active_item_on_base_branch_still_gates(self):
        """On main with an active item that has no branch yet — still a gate."""
        q = queue(active(1), item(2))
        r = route("start", q, "main", "main")
        assert r.stop == "decision"

    def test_start_active_item_already_done_starts_next(self):
        """Active item is done on its branch — no gate needed."""
        q = queue(item(1, done_branch="issue-1-x"), item(2))
        r = route("start", q, "issue-1-x", "main")
        assert r.run == "start" and r.activate == "o/r#2"

    def test_decision_includes_options(self):
        q = queue(active(1), item(2))
        r = route("start", q, "issue-1-x", "main")
        assert r.stop == "decision"
        # The options field carries the four choices
```

Also keep the tests that assert `continue` still works as before (carry on).

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_route.py -v -x`
Expected: FAIL — `start`/`next` currently alias to `continue`

- [ ] **Step 3: Implement the route change**

In `work_route.py`, inside the `if active is not None:` block, add a check
before the existing logic:

```python
# "Move on" commands gate on the active item
if command in ("start", "next"):
    return _enrich_offer(Route(
        stop="decision", reason="current_item_not_released",
        issue=active.ref, title=active.title, offer=offer,
        detail=f"{active.ref} is active; release it before moving on"
    ), queue)
```

This goes at the top of the `if active is not None:` block, before the
`if command == "done":` check. The `done` check stays — `done` still marks
done.

- [ ] **Step 4: Update the remaining tests**

Remove `test_next_is_another_spelling_of_continue`. Update
`test_every_spelling_gives_the_same_route` to only assert `continue` behaves
the same regardless of state (or remove it and replace with explicit tests).
Update `TestPassThrough.test_no_queue_runs_the_command_as_given` to include
`start`.

- [ ] **Step 5: Run all route tests**

Run: `python3 -m pytest tests/test_work_route.py -v`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add project/work_route.py tests/test_work_route.py
git commit -m "feat(route): start/next gate on active item instead of aliasing continue (#596)"
```

---

## Batch 2: Gate handler and dirty-tree check in work.py

### Task 2: Add dirty-tree check and decision options in `apply_route`

**Files:**
- Modify: `project/work.py` (in `apply_route`)
- Create: `work/handlers/move-on-gate.md`

**Interfaces:**
- Consumes: `route()` returning `reason="current_item_not_released"`
- Produces: `ACTION=stop` (dirty tree) or `ACTION=decision DECISION=current_item_not_released OPTIONS=done,deferred,move_down,continue_current`

- [ ] **Step 1: Create the handler file**

```markdown
# Handler: move-on-gate (current_item_not_released)

The user typed `start` or `next` while an item is active. Present the
four ways to release it:

1. **Done** (Recommended when the work is complete) — mark it done on
   its branch; `work end` lands it later. The next branch stacks on it.
2. **Deferred** — not doing it now; it moves to the deferred section
   and leaves the open queue.
3. **Move down** — stays in the queue but loses its active slot. Ask
   where: after a named item, or at the end.
4. **Continue** — the "move on" was a mistake; carry on with the
   current item.

Present via AskUserQuestion with a recommendation based on context.
After the user chooses, call the orchestrator:

- Done: `step_done=move_on_gate outcome=completed choice=done`
- Deferred: `step_done=move_on_gate outcome=completed choice=deferred reason="<user's reason>"`
- Move down: `step_done=move_on_gate outcome=completed choice=move_down position=end`
- Continue: `step_done=move_on_gate outcome=completed choice=continue`
```

- [ ] **Step 2: Add the dirty-tree check in `apply_route`**

In `work.py`'s `apply_route`, after `route()` returns, check for the
`current_item_not_released` decision. Before presenting it, check for
uncommitted changes in the project repo:

```python
if r.reason == "current_item_not_released":
    # Dirty tree → hard stop, the tool can prove work isn't finished
    project_path = args.get("project", "")
    if project_path:
        from git_util import git
        status = git(Path(project_path), "status", "--porcelain")
        if status.stdout.strip():
            files = status.stdout.strip()
            lines += ["ACTION=stop",
                      f"DETAIL={r.issue} has uncommitted changes:\n{files}",
                      "NEXT=commit or stash the changes, then try again"]
            return args, lines, True
    # Clean tree → present the decision
    lines += ["ACTION=decision", f"DECISION={r.reason}",
              f"DETAIL={r.detail}",
              "OPTIONS=done,deferred,move_down,continue_current"]
    return args, lines, True
```

- [ ] **Step 3: Run the full test suite to check nothing breaks**

Run: `python3 -m pytest tests/test_work_route.py tests/test_work.py -v` (if test_work.py exists)
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add project/work.py work/handlers/move-on-gate.md
git commit -m "feat(work): dirty-tree check and move-on-gate handler (#596)"
```

---

### Task 3: Handle gate option outcomes in work.py

**Files:**
- Modify: `project/work.py` (add gate step to START_STEPS or handle in apply_route)

**Interfaces:**
- Consumes: `choice=done|deferred|move_down|continue` from the handler
- Produces: runs `plan_manager.py mark-done` / `defer-active` / `move-down` / routes to continue

The gate is not a pipeline step — it's a routing decision. When the user
picks an option, `apply_route` is called again with the choice, executes
it, and then routes to `start` for the next item (or `continue` for
"continue current").

In practice: the handler tells the LLM what `step_done=` to send. The
LLM sends it. `apply_route` sees `choice=done` etc, runs the plan_manager
command, and then re-routes as `start` (no active item now → picks next).

- [ ] **Step 1: Add choice handling to apply_route**

After the `current_item_not_released` decision is returned and the user
responds, the next call comes in with `step_done=move_on_gate` and
`choice=<option>`. Handle it before the normal `step_done` path:

```python
# In main(), before the pipeline runs:
if args.get("step_done") == "move_on_gate":
    choice = args.get("choice", "")
    if choice == "continue":
        # Re-route as continue
        args["command"] = "continue"
        args.pop("step_done", None)
        args.pop("choice", None)
        # Fall through to apply_route which will route as continue
    elif choice == "done":
        # Mark done, then re-route
        _run_mark_done(args)
        args["command"] = "continue"
        args.pop("step_done", None)
        # Fall through — no active item now, starts next
    elif choice == "deferred":
        _run_defer_active(args)
        args["command"] = "continue"
        args.pop("step_done", None)
    elif choice == "move_down":
        _run_move_down(args)
        args["command"] = "continue"
        args.pop("step_done", None)
```

Each `_run_*` function calls the corresponding plan_manager command via
subprocess. After each, the item is no longer active, so re-routing as
`continue` picks up the next open item.

- [ ] **Step 2: Implement `_run_mark_done`, `_run_defer_active`, `_run_move_down`**

```python
def _run_mark_done(args):
    queue_path = _decided_queue_file(args)
    branch = args.get("branch", "")
    subprocess.run(["python3", str(_PLAN_MANAGER), "mark-done",
                    str(queue_path), f"branch={branch}"], check=True)

def _run_defer_active(args):
    queue_path = _decided_queue_file(args)
    reason = args.get("reason", "")
    subprocess.run(["python3", str(_PLAN_MANAGER), "defer-active",
                    str(queue_path), f"reason={reason}"], check=True)

def _run_move_down(args):
    queue_path = _decided_queue_file(args)
    position = args.get("position", "end")
    subprocess.run(["python3", str(_PLAN_MANAGER), "move-down",
                    str(queue_path), f"position={position}"], check=True)
```

- [ ] **Step 3: Commit**

```bash
git add project/work.py
git commit -m "feat(work): handle move-on gate options in apply_route (#596)"
```

---

## Batch 3: Plan manager commands — defer-active and move-down

### Task 4: Add `defer-active` command to plan_manager

**Files:**
- Modify: `work-slot/plan_manager.py`
- Modify: `tests/test_plan_manager.py` (or create `tests/test_defer_active.py`)

**Interfaces:**
- Produces: `defer-active` command that deactivates the active item, moves it to deferred section, activates next

- [ ] **Step 1: Write failing test**

```python
def test_defer_active_moves_item_to_deferred(tmp_path):
    queue_file = tmp_path / ".queue"
    queue_file.write_text(textwrap.dedent("""\
        ## State
        branch: issue-1-x
        covers: o/r#1

        ## Queue
        - [ ] o/r#1 — first item ← active
        - [ ] o/r#2 — second item
    """))
    result = subprocess.run(
        ["python3", str(PLAN_MANAGER), "defer-active", str(queue_file),
         "reason=not ready yet"],
        capture_output=True, text=True)
    assert result.returncode == 0
    assert "DEFERRED=o/r#1" in result.stdout
    # Verify: item 1 is in deferred, item 2 is now active
    content = queue_file.read_text()
    assert "← active" not in content or "o/r#2" in content.split("← active")[0]
    assert "not ready yet" in content  # reason in deferred section
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_defer_active.py -v -x`
Expected: FAIL

- [ ] **Step 3: Implement `defer-active` in plan_manager**

Add a `defer_active` function that:
1. Finds the active item
2. Removes it from the queue
3. Appends it to the deferred section with scale/complexity/reason
4. Activates the next open item
5. Clears the state branch (the deferred item's branch is no longer active)

```python
def defer_active(plan_path: Path, reason: str = "") -> dict:
    tree = parse_plan(plan_path)
    active = _find_active_leaf(tree.queue)
    if not active:
        raise MarkDoneError("no_active_issue", "no active item to defer")
    ref = active.ref
    title = active.title
    scale = active.scale or "M"
    complexity = active.complexity or "Med"
    # Remove from queue
    _remove_item(tree.queue, ref)
    # Add to deferred
    append_deferred(plan_path, title, scale, complexity, [], reason)
    # Activate next
    leaves = flatten_leaves(tree)
    next_leaf = _next_open_leaf(leaves, 0)
    if next_leaf:
        _mark_active(tree.queue, next_leaf.ref)
    tree.state["branch"] = ""
    tree.state["covers"] = str(next_leaf.ref) if next_leaf else ""
    rewrite_plan(plan_path, tree)
    return {"deferred": ref, "next": next_leaf.ref if next_leaf else None}
```

Wire it into the command dispatch.

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_defer_active.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add work-slot/plan_manager.py tests/test_defer_active.py
git commit -m "feat(plan-manager): defer-active command for move-on gate (#596)"
```

---

### Task 5: Add `move-down` command to plan_manager

**Files:**
- Modify: `work-slot/plan_manager.py`
- Create: `tests/test_move_down.py`

**Interfaces:**
- Produces: `move-down` command that deactivates the active item, moves it to end (or after a named item), activates next

- [ ] **Step 1: Write failing test**

```python
def test_move_down_to_end(tmp_path):
    queue_file = tmp_path / ".queue"
    queue_file.write_text(textwrap.dedent("""\
        ## State
        branch: issue-1-x
        covers: o/r#1

        ## Queue
        - [ ] o/r#1 — first item ← active
        - [ ] o/r#2 — second item
        - [ ] o/r#3 — third item
    """))
    result = subprocess.run(
        ["python3", str(PLAN_MANAGER), "move-down", str(queue_file),
         "position=end"],
        capture_output=True, text=True)
    assert result.returncode == 0
    assert "MOVED=o/r#1" in result.stdout
    # item 2 is now active, item 1 is at the end
    content = queue_file.read_text()
    lines = [l for l in content.splitlines() if "- [ ]" in l]
    assert "o/r#2" in lines[0]  # now first
    assert "o/r#1" in lines[-1]  # moved to end
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL

- [ ] **Step 3: Implement `move-down` in plan_manager**

```python
def move_down(plan_path: Path, position: str = "end") -> dict:
    tree = parse_plan(plan_path)
    active = _find_active_leaf(tree.queue)
    if not active:
        raise MarkDoneError("no_active_issue", "no active item to move")
    ref = active.ref
    # Deactivate
    active.active = False
    # Remove from current position
    items = tree.queue  # the flat list
    idx = next(i for i, item in enumerate(items) if item.ref == ref)
    moved = items.pop(idx)
    # Place at end (or after a named item)
    if position == "end":
        items.append(moved)
    else:
        # position is a ref like "o/r#3"
        target = next((i for i, item in enumerate(items) if str(item.ref) == position), None)
        if target is not None:
            items.insert(target + 1, moved)
        else:
            items.append(moved)
    # Activate next open item
    leaves = flatten_leaves(tree)
    next_leaf = next((l for l in leaves if not l.completed and not l.active and not l.done_branch), None)
    if next_leaf:
        _mark_active(tree.queue, next_leaf.ref)
    tree.state["branch"] = ""
    tree.state["covers"] = str(next_leaf.ref) if next_leaf else ""
    rewrite_plan(plan_path, tree)
    return {"moved": ref, "next": next_leaf.ref if next_leaf else None}
```

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_move_down.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add work-slot/plan_manager.py tests/test_move_down.py
git commit -m "feat(plan-manager): move-down command for move-on gate (#596)"
```

---

## Batch 4: Integration and documentation

### Task 6: Integration tests with real queue flows

**Files:**
- Create: `tests/test_move_on_gate.py`

**Interfaces:**
- Consumes: `route()` from `work_route.py`, `defer_active()` and `move_down()` from `plan_manager.py`

- [ ] **Step 1: Write integration tests**

Test the full gate flow: route returns decision → option chosen → plan_manager
executes → re-route picks next item.

```python
class TestMoveOnGateIntegration:
    """Each row of the gate table from #596."""

    def test_no_active_item_starts_next(self):
        """No gate when nothing is active."""
        ...

    def test_active_item_already_done_starts_next(self):
        """Done items don't gate."""
        ...

    def test_active_item_clean_tree_returns_decision(self):
        """Clean working tree → 4-option decision."""
        ...

    def test_done_option_marks_done_then_starts_next(self):
        """User picks 'done' → mark-done → route picks next."""
        ...

    def test_deferred_option_defers_then_starts_next(self):
        """User picks 'deferred' → defer-active → route picks next."""
        ...

    def test_move_down_option_moves_then_starts_next(self):
        """User picks 'move down' → move-down → route picks next."""
        ...

    def test_continue_option_carries_on(self):
        """User picks 'continue' → routes as continue."""
        ...

    def test_start_and_next_produce_identical_gate(self):
        """start and next are still equivalent — both mean 'move on'."""
        q = queue(active(1), item(2))
        r_start = route("start", q, "issue-1-x", "main")
        r_next = route("next", q, "issue-1-x", "main")
        assert r_start == r_next
```

- [ ] **Step 2: Run tests**

Run: `python3 -m pytest tests/test_move_on_gate.py -v`
Expected: ALL PASS

- [ ] **Step 3: Commit**

```bash
git add tests/test_move_on_gate.py
git commit -m "test: integration tests for move-on gate (#596)"
```

---

### Task 7: Documentation updates

**Files:**
- Modify: `work/SKILL.md` (command table, description)
- Modify: `docs/development/lifecycle-maintainer-guide.md` (section 4)
- Modify: `CLAUDE.md` (Key Skills — work entry)

**Interfaces:**
- None — documentation only

- [ ] **Step 1: Update work/SKILL.md**

Command table changes:
```markdown
| `work` | `python3 ~/.claude/skills/project/work.py continue` |
| `work continue` | `python3 ~/.claude/skills/project/work.py continue` |
| `work start` (no issue) | `python3 ~/.claude/skills/project/work.py start` |
| `work next` | `python3 ~/.claude/skills/project/work.py next` |
```

Update the description to reflect the two intents:
- `work` / `work continue` → carry on with current item
- `work start` / `work next` → move on to next item (gates on active work)
- `work done` → mark current item done, start nothing

- [ ] **Step 2: Update maintainer guide**

Line 145: change from "one operation" to explain the two intents.
Add `current_item_not_released` to the decisions list in section 4.2.

- [ ] **Step 3: Update CLAUDE.md Key Skills — work entry**

Change the work entry from "four spellings of one operation" to reflect
the carry-on / move-on distinction.

- [ ] **Step 4: Commit**

```bash
git add work/SKILL.md docs/development/lifecycle-maintainer-guide.md CLAUDE.md
git commit -m "docs: update lifecycle docs for move-on gate (#596)"
```

---

## Design decisions

### Deferred/moved item with an existing branch

When the active item has commits on a branch and is deferred or moved down:
- The branch stays as-is (backed up in canonical via the next `work end`)
- The next item starts from the base branch, not stacked on the
  unfinished branch — `set-base` / `(from main)` applied automatically
- If the user later wants to resume the deferred item, they use
  `work start #N` which finds the existing branch

### Relationship to `work pause`

`pause` saves WIP commits and pushes to the stack. `defer` and `move-down`
do not save WIP — they expect a clean tree (the dirty-tree check ensures
this). The distinction:
- **Pause**: "I'll come back to this soon" — WIP saved, stack restored
  with `work resume`
- **Defer**: "Not doing this now" — leaves the queue entirely
- **Move down**: "Not next" — stays in queue but loses priority

## References

- [work_route.py] — routing logic, `route()` function
- [work.py] — pipeline orchestration, `apply_route()`
- [plan_manager.py] — queue manipulation commands
- [queue_io.py:19-80] — QueueItem/Queue data model
- [tests/test_work_route.py] — existing route tests (22 tests)
- [docs/development/lifecycle-maintainer-guide.md:145] — "one operation" text
- [work/SKILL.md:37-40] — command table
- [GitHub #596] — focal issue
