# Context Budget Recommendations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #586 — work watches the context budget and recommends continue or wrap or end
**Issue group:** #586

**Goal:** Give the LLM the data it needs to make informed wrap-vs-continue
decisions by carrying scale/complexity on queue items and printing next-item
metadata alongside OFFER=wrap_or_continue.

**Architecture:** Queue items gain optional `(scale/complexity)` metadata.
The routing layer reads the next open item and prints KEY=VALUE lines
(NEXT_ITEM, NEXT_SCALE, etc.) alongside OFFER. The work skill text is
updated with a three-way recommendation table and mid-task monitoring
guidance.

**Tech Stack:** Python (queue_io, plan_manager, work_route, work.py),
Markdown (work/SKILL.md)

## Global Constraints

- Queue item format must round-trip through both `queue_io.py` and
  `plan_manager.py` parsers without data loss
- Scale values: XS, S, M, L, XL (match GitHub label convention)
- Complexity values: Low, Med, High (match GitHub label convention)
- Both fields are optional; absence means "unknown"
- No external API calls at recommendation time — all data from queue

---

## Batch 1: Queue item metadata

### Task 1: Add scale/complexity to QueueItem and queue_io parsing

**Files:**
- Modify: `project/queue_io.py`
- Test: `tests/test_queue_io.py` (new tests in existing file, or new file)

**Interfaces:**
- Produces: `QueueItem.scale: str`, `QueueItem.complexity: str`
- Produces: `write_queue` renders `(S/Med)` when present
- Produces: `_parse_item` extracts scale/complexity from item line

- [ ] **Step 1: Write failing test for parsing scale/complexity**

```python
# tests/test_queue_item_metadata.py
from pathlib import Path
import sys
sys.path.insert(0, str(Path(__file__).parent.parent / "project"))
from queue_io import Queue, QueueItem, QueueState, read_queue, write_queue

def test_parse_item_with_scale_and_complexity(tmp_path):
    queue_file = tmp_path / ".queue"
    queue_file.write_text(
        "- [ ] o/r#1 — First task (S/Med)        ← active\n"
        "- [ ] o/r#2 — Second task (M/High)\n"
        "- [ ] o/r#3 — No metadata\n\n"
        "## State\nbranch: issue-1-x\nissue-repo: O/r\nstatus: active\n"
    )
    queue = read_queue(tmp_path)
    assert queue is not None
    assert queue.items[0].scale == "S"
    assert queue.items[0].complexity == "Med"
    assert queue.items[1].scale == "M"
    assert queue.items[1].complexity == "High"
    assert queue.items[2].scale == ""
    assert queue.items[2].complexity == ""
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_queue_item_metadata.py::test_parse_item_with_scale_and_complexity -xvs`
Expected: FAIL — QueueItem has no `scale` attribute

- [ ] **Step 3: Add scale/complexity fields to QueueItem**

In `project/queue_io.py`, add fields to QueueItem dataclass (after `from_main`):

```python
@dataclass
class QueueItem:
    ref: str
    title: str
    completed: bool
    active: bool
    done_branch: str = ""
    from_main: bool = False
    scale: str = ""
    complexity: str = ""
```

- [ ] **Step 4: Update _parse_item to extract scale/complexity**

In `project/queue_io.py`, in the `_parse_item` function, after extracting
`from_main`, add parsing for `(S/Med)` from the title. Add a module-level
regex:

```python
_SCALE_COMPLEXITY_RE = re.compile(r'\s*\(([A-Z]+)/([A-Za-z]+)\)')
```

In `_parse_item`, after stripping `(from main)` from the title, strip
the scale/complexity pattern:

```python
scale = ""
complexity = ""
sc_match = _SCALE_COMPLEXITY_RE.search(title)
if sc_match:
    scale = sc_match.group(1)
    complexity = sc_match.group(2)
    title = title[:sc_match.start()] + title[sc_match.end():]
title = title.strip()
```

Pass `scale=scale, complexity=complexity` to the QueueItem constructor.

- [ ] **Step 5: Update write_queue to render scale/complexity**

In `project/queue_io.py`, in `write_queue`, after building the title
line and before the `from_main` marker:

```python
scale_str = f" ({item.scale}/{item.complexity})" if item.scale else ""
base = " (from main)" if item.from_main else ""
```

Insert `scale_str` between the title and `base` in the line format.

- [ ] **Step 6: Run test to verify it passes**

Run: `python3 -m pytest tests/test_queue_item_metadata.py::test_parse_item_with_scale_and_complexity -xvs`
Expected: PASS

- [ ] **Step 7: Write round-trip test**

```python
def test_write_then_read_preserves_scale_complexity(tmp_path):
    original = Queue(
        items=[
            QueueItem(ref="o/r#1", title="First", completed=False, active=True,
                      scale="S", complexity="Med"),
            QueueItem(ref="o/r#2", title="Second", completed=False, active=False,
                      scale="", complexity=""),
        ],
        state=QueueState(branch="issue-1-x", issue_repo="O/r", status="active",
                         covers="o/r#1"))
    write_queue(tmp_path, original)
    reloaded = read_queue(tmp_path)
    assert reloaded.items[0].scale == "S"
    assert reloaded.items[0].complexity == "Med"
    assert reloaded.items[1].scale == ""
    assert reloaded.items[1].complexity == ""
    assert reloaded.items[0].title == "First"
```

- [ ] **Step 8: Run round-trip test**

Run: `python3 -m pytest tests/test_queue_item_metadata.py::test_write_then_read_preserves_scale_complexity -xvs`
Expected: PASS

- [ ] **Step 9: Run existing round-trip tests to verify no regression**

Run: `python3 -m pytest tests/test_queue_file_roundtrip.py -xvs`
Expected: PASS — existing items without scale/complexity still round-trip

- [ ] **Step 10: Commit**

```bash
git add project/queue_io.py tests/test_queue_item_metadata.py
git commit -m "feat: queue items carry scale and complexity metadata (#586) Refs #586"
```

### Task 2: plan_manager scale/complexity parsing and set-scale command

**Files:**
- Modify: `work-slot/plan_manager.py`
- Test: `tests/test_queue_item_metadata.py` (extend)

**Interfaces:**
- Consumes: Queue item format `(S/Med)` from Task 1
- Produces: `plan_manager.py set-scale <path> ref=o/r#1 scale=S complexity=Med`
- Produces: plan_manager's item renderer includes `(S/Med)` when present

- [ ] **Step 1: Write failing test for plan_manager reading scale/complexity**

```python
def test_plan_manager_reads_scale_from_queue_io_format(tmp_path):
    """plan_manager must parse items written by queue_io, including scale."""
    sys.path.insert(0, str(Path(__file__).parent.parent / "work-slot"))
    from plan_manager import parse_plan, flatten_leaves
    queue_file = tmp_path / ".queue"
    queue_file.write_text(
        "- [ ] o/r#1 — First task (S/Med)        ← active\n"
        "- [ ] o/r#2 — Second task\n\n"
        "## State\nbranch: issue-1-x\nissue-repo: o/r\nstatus: active\n"
    )
    tree = parse_plan(queue_file)
    leaves = flatten_leaves(tree)
    assert leaves[0].active
    assert leaves[0].title == "First task"
```

- [ ] **Step 2: Run test to verify it passes (or identify what breaks)**

Run: `python3 -m pytest tests/test_queue_item_metadata.py::test_plan_manager_reads_scale_from_queue_io_format -xvs`

plan_manager's `_ITEM_RE` regex may or may not match the `(S/Med)` suffix.
If it fails, the regex needs updating. If it passes (treating `(S/Med)`
as part of the title), that's acceptable — plan_manager doesn't need to
use the scale/complexity; it only needs to not break on the format.

- [ ] **Step 3: Write failing test for set-scale command**

```python
def test_set_scale_updates_item_metadata(tmp_path):
    from plan_manager import main as pm_main
    queue_file = tmp_path / ".queue"
    queue_file.write_text(
        "- [ ] o/r#1 — First task        ← active\n"
        "- [ ] o/r#2 — Second task\n\n"
        "## State\nbranch: issue-1-x\nissue-repo: o/r\nstatus: active\n"
    )
    # Run set-scale via CLI
    import subprocess
    result = subprocess.run(
        ["python3", str(Path(__file__).parent.parent / "work-slot" / "plan_manager.py"),
         "set-scale", str(queue_file), "ref=o/r#1", "scale=S", "complexity=Med"],
        capture_output=True, text=True)
    assert result.returncode == 0, result.stderr

    reloaded = read_queue(tmp_path)
    assert reloaded.items[0].scale == "S"
    assert reloaded.items[0].complexity == "Med"
    assert reloaded.items[0].title == "First task"
```

- [ ] **Step 4: Run test to verify it fails**

Run: `python3 -m pytest tests/test_queue_item_metadata.py::test_set_scale_updates_item_metadata -xvs`
Expected: FAIL — Unknown command: set-scale

- [ ] **Step 5: Implement set-scale command in plan_manager**

In `work-slot/plan_manager.py`, add a `set_scale` function and wire
it into the CLI dispatch:

```python
def set_scale(plan_path: Path, ref: str, scale: str = "", complexity: str = "") -> dict:
    """Update an item's scale and complexity by ref."""
    from queue_io import read_queue, write_queue, queue_dir
    qdir = plan_path.parent
    queue = read_queue(qdir, exact=True)
    if queue is None:
        return {"ERROR": "no_queue", "ERROR_DETAIL": f"no queue at {qdir}"}
    found = None
    for item in queue.items:
        if item.ref == ref or item.ref.endswith(f"#{ref.lstrip('#')}"):
            found = item
            break
    if found is None:
        return {"ERROR": "not_found", "ERROR_DETAIL": f"{ref} not in queue"}
    found.scale = scale
    found.complexity = complexity
    write_queue(qdir, queue)
    return {"UPDATED": str(found.ref), "SCALE": scale, "COMPLEXITY": complexity}
```

Add CLI dispatch:

```python
elif command == "set-scale":
    ref = opts.get("ref", "")
    scale = opts.get("scale", "")
    complexity = opts.get("complexity", "")
    if not ref:
        print("ERROR=ref is required", file=_sys.stderr)
        return 1
    result = set_scale(plan_path, ref, scale, complexity)
    for k, v in result.items():
        print(f"{k}={v}")
    return 1 if "ERROR" in result else 0
```

- [ ] **Step 6: Run test to verify it passes**

Run: `python3 -m pytest tests/test_queue_item_metadata.py::test_set_scale_updates_item_metadata -xvs`
Expected: PASS

- [ ] **Step 7: Run existing queue round-trip tests**

Run: `python3 -m pytest tests/test_queue_file_roundtrip.py -xvs`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add work-slot/plan_manager.py tests/test_queue_item_metadata.py
git commit -m "feat: plan_manager set-scale command for queue item metadata (#586) Refs #586"
```

## Batch 2: Pipeline data delivery

### Task 3: Extend Route with next-item data and print NEXT_* lines

**Files:**
- Modify: `project/work_route.py`
- Modify: `project/work.py`
- Test: `tests/test_queue_item_metadata.py` (extend) or `tests/test_work_wrap_end.py` (extend)

**Interfaces:**
- Consumes: `QueueItem.scale`, `QueueItem.complexity` from Task 1
- Produces: `NEXT_ITEM=`, `NEXT_TITLE=`, `NEXT_SCALE=`, `NEXT_COMPLEXITY=`, `DONE_WAITING=` in pipeline output

- [ ] **Step 1: Write failing test for NEXT_* lines in OFFER output**

```python
# tests/test_queue_item_metadata.py (extend)

def test_offer_includes_next_item_metadata(tmp_path):
    """When OFFER=wrap_or_continue is printed, NEXT_* lines appear."""
    import subprocess
    ROOT = Path(__file__).parent.parent
    WORK = str(ROOT / "project" / "work.py")
    # Set up a minimal world with two items, first one done
    project = tmp_path / "project"
    project.mkdir()
    subprocess.run(["git", "-C", str(project), "init", "-q", "-b", "main", "--template="],
                   capture_output=True, check=True)
    (project / "seed.txt").write_text("x")
    subprocess.run(["git", "-C", str(project), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(project), "commit", "-q", "-m", "init"], capture_output=True)

    write_queue(project, Queue(
        items=[
            QueueItem(ref="o/r#1", title="Done task", completed=False, active=True,
                      done_branch="issue-1-x"),
            QueueItem(ref="o/r#2", title="Next task", completed=False, active=False,
                      scale="M", complexity="High"),
        ],
        state=QueueState(branch="issue-1-x", issue_repo="O/r", status="active",
                         covers="o/r#1")))
    subprocess.run(["git", "-C", str(project), "add", ".queue"], capture_output=True)
    subprocess.run(["git", "-C", str(project), "commit", "-q", "-m", "queue"], capture_output=True)

    result = subprocess.run(
        ["python3", WORK, "continue",
         f"workspace={project}", f"project={project}",
         "branch=main", "base_branch=main", "on_main=yes",
         f"plan_path={project / '.queue'}", "issue_repo=O/r"],
        capture_output=True, text=True, cwd=str(project))
    out = dict(l.split("=", 1) for l in result.stdout.splitlines() if "=" in l)
    assert "OFFER" in out, result.stdout
    assert out.get("NEXT_SCALE") == "M"
    assert out.get("NEXT_COMPLEXITY") == "High"
    assert "o/r#2" in out.get("NEXT_ITEM", "")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_queue_item_metadata.py::test_offer_includes_next_item_metadata -xvs`
Expected: FAIL — NEXT_SCALE not in output

- [ ] **Step 3: Add next-item fields to Route dataclass**

In `project/work_route.py`, add to the Route dataclass:

```python
next_item: str = ""
next_title: str = ""
next_scale: str = ""
next_complexity: str = ""
done_waiting: int = 0
```

- [ ] **Step 4: Populate next-item fields in route()**

In `project/work_route.py`, in the `route()` function, when `offer=True`,
read the next open item from the queue and populate the fields:

```python
if r.offer and queue:
    next_open = next((i for i in queue.items
                      if not i.completed and not i.active and not i.done_branch), None)
    if next_open:
        r.next_item = next_open.ref
        r.next_title = next_open.title
        r.next_scale = getattr(next_open, 'scale', '') or 'unknown'
        r.next_complexity = getattr(next_open, 'complexity', '') or 'unknown'
    r.done_waiting = len(queue.done_items())
```

- [ ] **Step 5: Print NEXT_* lines in work.py apply_route**

In `project/work.py`, in `apply_route()`, after printing
`OFFER=wrap_or_continue` and `DONE_ITEMS=`, add:

```python
if r.next_item:
    lines.append(f"NEXT_ITEM={r.next_item}")
    lines.append(f"NEXT_TITLE={r.next_title}")
    lines.append(f"NEXT_SCALE={r.next_scale}")
    lines.append(f"NEXT_COMPLEXITY={r.next_complexity}")
lines.append(f"DONE_WAITING={r.done_waiting}")
```

- [ ] **Step 6: Extend sequence_report with NEXT_* lines**

In `project/work.py`, in `sequence_report()`, after the THEN= line,
read the queue and print next-item data:

```python
from queue_io import read_queue as _read_queue
queue = _read_queue(workspace)
if queue:
    next_open = next((i for i in queue.items
                      if not i.completed and not i.active and not i.done_branch), None)
    if next_open:
        lines.append(f"NEXT_ITEM={next_open.ref}")
        lines.append(f"NEXT_TITLE={next_open.title}")
        lines.append(f"NEXT_SCALE={getattr(next_open, 'scale', '') or 'unknown'}")
        lines.append(f"NEXT_COMPLEXITY={getattr(next_open, 'complexity', '') or 'unknown'}")
    lines.append(f"DONE_WAITING={len(queue.done_items())}")
```

- [ ] **Step 7: Run test to verify it passes**

Run: `python3 -m pytest tests/test_queue_item_metadata.py::test_offer_includes_next_item_metadata -xvs`
Expected: PASS

- [ ] **Step 8: Run existing wrap/end tests**

Run: `python3 -m pytest tests/test_work_wrap_end.py -v`
Expected: PASS — existing tests unaffected by new output lines

- [ ] **Step 9: Commit**

```bash
git add project/work_route.py project/work.py tests/test_queue_item_metadata.py
git commit -m "feat: pipeline prints NEXT_ITEM/SCALE/COMPLEXITY with OFFER (#586) Refs #586"
```

## Batch 3: Skill text and worktrace

### Task 4: Update work skill text

**Files:**
- Modify: `work/SKILL.md`

**Interfaces:**
- Consumes: NEXT_ITEM, NEXT_SCALE, NEXT_COMPLEXITY, DONE_WAITING from Task 3

- [ ] **Step 1: Update the wrap-vs-continue section**

In `work/SKILL.md`, replace the `## Wrap-vs-continue decision` section
with the updated version that:

1. References the NEXT_* KEY=VALUE lines from the pipeline
2. Uses a three-way recommendation table:

| Situation | Recommend |
|-----------|-----------|
| Enough room, no done items waiting | **Continue** |
| Enough room, done items piling up (DONE_WAITING >= 2) | **End then continue** |
| Not enough room, done items waiting | **End and stop** |
| Not enough room, nothing done waiting | **Wrap and stop** |

3. Tells the LLM to combine NEXT_SCALE/NEXT_COMPLEXITY with its context
   estimate (counting issues worked) to judge "enough room"
4. Keeps the existing guidance about topic relevance and compressed context
5. Adds the worktrace recording template:

```
worktrace.record("context-budget", repo=workspace, branch=branch,
    recommendation="continue", next_issue="...", next_scale="...", chosen="continue")
```

6. Tells the LLM to sync the queue when it changes a scale/complexity
   label: `plan_manager.py set-scale <queue_path> ref=<ref> scale=<S> complexity=<Med>`

- [ ] **Step 2: Add mid-task monitoring section**

In `work/SKILL.md`, add a new `## Mid-task context monitoring` section
before `## Close / Sync`:

```markdown
## Mid-task context monitoring

While working on an item, watch for signs that the context is running
long relative to what remains on the task. If the item has consumed a
large share of the context window and there is still substantial work
ahead, suggest `work wrap` before context is exhausted — not after.

Signals: you can no longer recall details from the start of the session
without re-reading; the task is growing beyond its original scale
estimate; you are on your third or fourth sub-problem within the same
item.

This is a judgment call, not a hard threshold. A short item that is
almost done should finish. A large item with hours of work ahead should
wrap early enough for the handoff to capture context while it is fresh.
```

- [ ] **Step 3: Commit**

```bash
git add work/SKILL.md
git commit -m "feat: work skill three-way recommendation and mid-task monitoring (#586) Refs #586"
```

- [ ] **Step 4: Sync skills**

```bash
python3 scripts/claude-skill sync-local --all -y
```

## References

- [2026-11-22-context-budget-recommendations-design.md] — design spec
- [project/queue_io.py] — queue item data model and I/O
- [project/work_route.py] — routing logic, OFFER production
- [project/work.py:apply_route] — OFFER=wrap_or_continue output
- [project/work.py:sequence_report] — THEN= output
- [work-slot/plan_manager.py] — queue manipulation, append/activate
- [work/SKILL.md] — wrap-vs-continue decision text
- [project/worktrace.py] — event recording
- [GitHub #586] — focal issue
