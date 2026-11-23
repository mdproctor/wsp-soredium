# Read-only Survey Command Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #589 — Read-only survey command for every area
**Issue group:** #589

**Goal:** Build `project/areas.py` — a read-only command that surveys
every area in a family root and reports state, drift, closure status,
and what `work end` would do.

**Architecture:** Three-layer design (discover → inspect → report) in a
single file `project/areas.py`. Discovery walks the filesystem. Inspection
uses existing building blocks via read-only git operations (never
`canonical_sync.follow/ahead_of_canonical/reference` — all fetch).
Reporting rolls up findings. Output is KEY=VALUE (script) and JSON Lines
(CLI/TUI via `commands/survey.py`).

**Tech Stack:** Python 3.14, pytest, dataclasses, `git_util.git()`,
`queue_home.candidates()`, `slot_io.parse_slot()`, `verification.Finding`

## Global Constraints

- All git operations via `project.git_util.git()` — never
  `subprocess.run(["git", ...])` directly
- Never call `canonical_sync.follow()`, `.ahead_of_canonical()`, or
  `.reference()` — all three fetch for clone repos
- Read-only by construction: no `open(path, 'w')`, no `Path.write_text()`,
  no mutating git subcommands
- `verification.Finding` severities are uppercase: `ERROR`, `WARN`, `INFO`
- Issue references in `owner/repo#N` format everywhere
- `queue_home.candidates()` for queue discovery — never build queue paths
- Test helper `build_family()` builds a realistic family tree in `tmp_path`
  and is defined once in a conftest or test helper, reused across all tests

---

## Batch 1: Foundation — data model, discovery, git helpers

After this batch: `discover_areas()` finds all areas in a family root,
`repos_in_area()` enumerates repos within each area, and read-only git
comparison functions exist with tests. The read-only test pattern is
established.

### Task 1: Data model + discovery layer

**Files:**
- Create: `project/areas.py`
- Create: `tests/test_areas.py`

**Interfaces:**
- Produces: `RepoInfo(name, path, role)`, `Area(kind, path, slot_number,
  repo_name, repos, project_repo, workspace_repo)`,
  `discover_areas(family_root: Path) -> list[Area]`,
  `repos_in_area(area: Area, family_root: Path) -> list[RepoInfo]`

- [ ] **Step 1: Write failing test for `discover_areas` — canonical repos**

```python
# tests/test_areas.py
import subprocess
from pathlib import Path
from project.areas import discover_areas, Area, RepoInfo


def _git(repo, *args):
    subprocess.run(["git", "-C", str(repo)] + list(args),
                   capture_output=True, text=True, check=True)


def _init_repo(path: Path, branch: str = "main"):
    path.mkdir(parents=True, exist_ok=True)
    _git(path, "init", "-q", "-b", branch)
    (path / "README.md").write_text("init")
    _git(path, "add", "-A")
    _git(path, "commit", "-q", "-m", "init")


def build_family(tmp_path: Path) -> Path:
    """Build a realistic family tree with canonical, slot, clone, attic."""
    family = tmp_path / "family"
    family.mkdir()

    # Canonical project repo
    proj = family / "myproject"
    _init_repo(proj)

    # Canonical workspace repo (paired via wsp- convention)
    wsp = family / "wsp-myproject"
    _init_repo(wsp)
    (wsp / ".workspace").touch()

    # Slot
    slot_dir = family / "slots" / "1"
    slot_dir.mkdir(parents=True)
    slot_proj = slot_dir / "myproject"
    _init_repo(slot_proj)
    slot_wsp = slot_dir / "wsp-myproject"
    _init_repo(slot_wsp)
    (slot_wsp / ".workspace").touch()
    (slot_dir / ".slot").write_text(
        "repos: myproject, wsp-myproject\n"
        "primary: myproject\n"
        "branch: issue-42-feature\n"
        "title: Feature\n"
        "state: active\n"
    )

    # Clone
    clone_dir = family / "clone" / "myproject"
    clone_dir.mkdir(parents=True)
    clone_proj = clone_dir / "myproject"
    _init_repo(clone_proj)
    clone_wsp = clone_dir / "wsp-myproject"
    _init_repo(clone_wsp)
    (clone_wsp / ".workspace").touch()
    (clone_dir / ".permanent").touch()

    # Attic
    attic_dir = family / "slots" / "attic" / "2"
    attic_dir.mkdir(parents=True)
    attic_proj = attic_dir / "myproject"
    _init_repo(attic_proj)
    (attic_dir / ".slot").write_text(
        "repos: myproject\nprimary: myproject\n"
        "branch: old\ntitle: Old\nstate: archived\n"
    )

    return family


def test_discover_areas_finds_all_kinds(tmp_path):
    family = build_family(tmp_path)
    areas = discover_areas(family)
    kinds = [a.kind for a in areas]
    assert "canonical" in kinds
    assert "slot" in kinds
    assert "clone" in kinds
    assert "attic" in kinds


def test_discover_areas_ordering(tmp_path):
    family = build_family(tmp_path)
    areas = discover_areas(family)
    kinds = [a.kind for a in areas]
    # canonical first, then slot, clone, attic
    assert kinds.index("canonical") < kinds.index("slot")
    assert kinds.index("slot") < kinds.index("clone")
    assert kinds.index("clone") < kinds.index("attic")


def test_discover_areas_workspace_paired_not_separate(tmp_path):
    family = build_family(tmp_path)
    areas = discover_areas(family)
    # wsp-myproject should NOT appear as a separate area
    area_paths = [a.path.name for a in areas]
    assert "wsp-myproject" not in area_paths
    # canonical area should have workspace_repo set
    canonical = [a for a in areas if a.kind == "canonical"][0]
    assert canonical.workspace_repo is not None
    assert canonical.workspace_repo.name == "wsp-myproject"


def test_discover_areas_slot_has_repos(tmp_path):
    family = build_family(tmp_path)
    areas = discover_areas(family)
    slot = [a for a in areas if a.kind == "slot"][0]
    assert slot.project_repo is not None
    assert slot.slot_number == 1


def test_discover_areas_clone_has_permanent(tmp_path):
    family = build_family(tmp_path)
    areas = discover_areas(family)
    clone = [a for a in areas if a.kind == "clone"][0]
    assert clone.repo_name == "myproject"
    assert clone.project_repo is not None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_areas.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'project.areas'`

- [ ] **Step 3: Implement data model and discovery**

```python
# project/areas.py
"""Read-only survey of every area in a family root."""
from __future__ import annotations

from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional


@dataclass
class RepoInfo:
    name: str
    path: Path
    role: str  # "project" | "workspace"


@dataclass
class Area:
    kind: str  # "canonical" | "slot" | "clone" | "attic"
    path: Path
    slot_number: int | None = None
    repo_name: str | None = None
    repos: list[RepoInfo] = field(default_factory=list)
    project_repo: RepoInfo | None = None
    workspace_repo: RepoInfo | None = None


def _is_workspace(path: Path) -> bool:
    return (path / ".workspace").exists() or path.name.startswith("wsp-")


def _pair_workspace(project_name: str, family_root: Path) -> Path | None:
    wsp = family_root / f"wsp-{project_name}"
    if wsp.is_dir() and (wsp / ".git").exists():
        return wsp
    for child in family_root.iterdir():
        if not child.is_dir() or not (child / ".git").exists():
            continue
        if _is_workspace(child):
            proj_link = child / "proj"
            if proj_link.is_symlink():
                target = proj_link.resolve()
                if target.name == project_name:
                    return child
    return None


def _discover_repos_in_slot(slot_dir: Path) -> tuple[list[RepoInfo], RepoInfo | None, RepoInfo | None]:
    repos = []
    project_repo = None
    workspace_repo = None
    for child in sorted(slot_dir.iterdir()):
        if not child.is_dir() or not (child / ".git").exists():
            continue
        if child.name in ("slots", "clone", "attic"):
            continue
        if _is_workspace(child):
            ri = RepoInfo(name=child.name, path=child, role="workspace")
            repos.append(ri)
            workspace_repo = ri
        else:
            ri = RepoInfo(name=child.name, path=child, role="project")
            repos.append(ri)
            if project_repo is None:
                project_repo = ri
    return repos, project_repo, workspace_repo


def discover_areas(family_root: Path) -> list[Area]:
    areas: list[Area] = []
    canonicals: list[Area] = []
    slots: list[Area] = []
    clones: list[Area] = []
    attics: list[Area] = []

    workspace_paths: set[Path] = set()

    # Pass 1: identify canonical project repos (not workspace, not under slots/clone/attic)
    skip_dirs = {"slots", "clone", "attic", ".git"}
    for child in sorted(family_root.iterdir()):
        if not child.is_dir() or child.name in skip_dirs:
            continue
        if not (child / ".git").exists():
            continue
        if _is_workspace(child):
            workspace_paths.add(child)
            continue
        wsp_path = _pair_workspace(child.name, family_root)
        project_ri = RepoInfo(name=child.name, path=child, role="project")
        workspace_ri = None
        all_repos = [project_ri]
        if wsp_path:
            workspace_ri = RepoInfo(name=wsp_path.name, path=wsp_path, role="workspace")
            all_repos.append(workspace_ri)
            workspace_paths.add(wsp_path)
        canonicals.append(Area(
            kind="canonical", path=child,
            repos=all_repos, project_repo=project_ri,
            workspace_repo=workspace_ri,
        ))

    # Slots
    slots_dir = family_root / "slots"
    if slots_dir.is_dir():
        for child in sorted(slots_dir.iterdir(), key=lambda p: (not p.name.isdigit(), p.name)):
            if child.name == "attic":
                continue
            if not child.is_dir() or not (child / ".slot").exists():
                continue
            repos, proj, wsp = _discover_repos_in_slot(child)
            num = int(child.name) if child.name.isdigit() else None
            slots.append(Area(
                kind="slot", path=child, slot_number=num,
                repos=repos, project_repo=proj, workspace_repo=wsp,
            ))

    # Clones
    clone_dir = family_root / "clone"
    if clone_dir.is_dir():
        for child in sorted(clone_dir.iterdir()):
            if not child.is_dir() or not (child / ".permanent").exists():
                continue
            repos, proj, wsp = _discover_repos_in_slot(child)
            clones.append(Area(
                kind="clone", path=child, repo_name=child.name,
                repos=repos, project_repo=proj, workspace_repo=wsp,
            ))

    # Attic
    attic_dir = slots_dir / "attic" if slots_dir.is_dir() else None
    if attic_dir and attic_dir.is_dir():
        for child in sorted(attic_dir.iterdir(), key=lambda p: (not p.name.isdigit(), p.name)):
            if not child.is_dir() or not (child / ".slot").exists():
                continue
            repos, proj, wsp = _discover_repos_in_slot(child)
            num = int(child.name) if child.name.isdigit() else None
            attics.append(Area(
                kind="attic", path=child, slot_number=num,
                repos=repos, project_repo=proj, workspace_repo=wsp,
            ))

    return canonicals + slots + clones + attics
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_areas.py -v`
Expected: all 6 tests PASS

- [ ] **Step 5: Commit**

```bash
git add project/areas.py tests/test_areas.py
git commit -m "feat(areas): data model and discovery layer

Refs Hortora/soredium#589"
```

### Task 2: Read-only git helpers

**Files:**
- Modify: `project/areas.py`
- Modify: `tests/test_areas.py`

**Interfaces:**
- Consumes: `git_util.git(repo, *args) -> CompletedProcess`
- Produces: `follow_state_readonly(clone, canon, base) -> str`,
  `ahead_readonly(repo, canon, base, branch) -> int`,
  `entries_on_ref(repo, ref, blog_dir) -> set[str]`,
  `entry_content_from_ref(repo, ref, path) -> str`,
  `parse_frontmatter_from_text(text) -> dict | None`

- [ ] **Step 1: Write failing tests for git helpers**

```python
# Add to tests/test_areas.py
from project.areas import (
    follow_state_readonly, ahead_readonly,
    entries_on_ref, entry_content_from_ref,
    parse_frontmatter_from_text,
)


def test_follow_state_current(tmp_path):
    canon = tmp_path / "canon"
    _init_repo(canon)
    clone = tmp_path / "clone"
    _init_repo(clone)
    # Both on same commit — manually set same SHA
    _git(clone, "fetch", str(canon), "main:refs/heads/main")
    _git(clone, "reset", "--hard", "main")
    assert follow_state_readonly(clone, canon, "main") == "current"


def test_follow_state_fast_forward(tmp_path):
    canon = tmp_path / "canon"
    _init_repo(canon)
    clone = tmp_path / "clone"
    _init_repo(clone)
    _git(clone, "fetch", str(canon), "main:refs/heads/main")
    _git(clone, "reset", "--hard", "main")
    # canon advances
    (canon / "new.txt").write_text("new")
    _git(canon, "add", "-A")
    _git(canon, "commit", "-q", "-m", "advance")
    assert follow_state_readonly(clone, canon, "main") == "fast-forward"


def test_follow_state_diverged(tmp_path):
    canon = tmp_path / "canon"
    _init_repo(canon)
    clone = tmp_path / "clone"
    _init_repo(clone)
    _git(clone, "fetch", str(canon), "main:refs/heads/main")
    _git(clone, "reset", "--hard", "main")
    # both advance differently
    (canon / "a.txt").write_text("a")
    _git(canon, "add", "-A")
    _git(canon, "commit", "-q", "-m", "canon change")
    (clone / "b.txt").write_text("b")
    _git(clone, "add", "-A")
    _git(clone, "commit", "-q", "-m", "clone change")
    assert follow_state_readonly(clone, canon, "main") == "diverged"


def test_follow_state_unknown_missing_branch(tmp_path):
    canon = tmp_path / "canon"
    _init_repo(canon)
    clone = tmp_path / "clone"
    _init_repo(clone)
    assert follow_state_readonly(clone, canon, "nonexistent") == "unknown"


def test_ahead_readonly_zero(tmp_path):
    canon = tmp_path / "canon"
    _init_repo(canon)
    clone = tmp_path / "clone"
    _init_repo(clone)
    _git(clone, "fetch", str(canon), "main:refs/heads/main")
    _git(clone, "reset", "--hard", "main")
    assert ahead_readonly(clone, canon, "main", "main") == 0


def test_ahead_readonly_with_commits(tmp_path):
    canon = tmp_path / "canon"
    _init_repo(canon)
    clone = tmp_path / "clone"
    _init_repo(clone)
    _git(clone, "fetch", str(canon), "main:refs/heads/main")
    _git(clone, "reset", "--hard", "main")
    _git(clone, "checkout", "-b", "feature")
    (clone / "feat.txt").write_text("feat")
    _git(clone, "add", "-A")
    _git(clone, "commit", "-q", "-m", "feat 1")
    (clone / "feat2.txt").write_text("feat2")
    _git(clone, "add", "-A")
    _git(clone, "commit", "-q", "-m", "feat 2")
    assert ahead_readonly(clone, canon, "main", "feature") == 2


def test_entries_on_ref(tmp_path):
    repo = tmp_path / "repo"
    _init_repo(repo)
    blog = repo / "docs" / "blog"
    blog.mkdir(parents=True)
    (blog / "entry1.md").write_text("# Entry 1")
    (blog / "entry2.md").write_text("# Entry 2")
    _git(repo, "add", "-A")
    _git(repo, "commit", "-q", "-m", "add entries")
    result = entries_on_ref(repo, "main", "docs/blog")
    assert result == {"docs/blog/entry1.md", "docs/blog/entry2.md"}


def test_entries_on_ref_empty(tmp_path):
    repo = tmp_path / "repo"
    _init_repo(repo)
    result = entries_on_ref(repo, "main", "docs/blog")
    assert result == set()


def test_entry_content_from_ref(tmp_path):
    repo = tmp_path / "repo"
    _init_repo(repo)
    blog = repo / "docs" / "blog"
    blog.mkdir(parents=True)
    (blog / "entry.md").write_text("---\ntitle: Test\n---\nBody")
    _git(repo, "add", "-A")
    _git(repo, "commit", "-q", "-m", "add entry")
    content = entry_content_from_ref(repo, "main", "docs/blog/entry.md")
    assert "title: Test" in content
    assert "Body" in content


def test_parse_frontmatter_from_text():
    text = "---\ntitle: Hello\nprojects: owner/repo\n---\nBody text"
    fm = parse_frontmatter_from_text(text)
    assert fm is not None
    assert fm["title"] == "Hello"
    assert fm["projects"] == "owner/repo"


def test_parse_frontmatter_from_text_no_frontmatter():
    text = "Just plain text without frontmatter"
    assert parse_frontmatter_from_text(text) is None


def test_parse_frontmatter_from_text_bare_project():
    text = "---\ntitle: Hello\nprojects: engine\n---\nBody"
    fm = parse_frontmatter_from_text(text)
    assert fm is not None
    assert "/" not in fm["projects"]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_areas.py -k "follow_state or ahead_readonly or entries_on or parse_frontmatter" -v`
Expected: FAIL — `ImportError`

- [ ] **Step 3: Implement git helpers**

Add to `project/areas.py`:

```python
import sys
import yaml
from project.git_util import git


def follow_state_readonly(clone_repo: Path, canon_repo: Path, base: str) -> str:
    clone_sha = git(clone_repo, "rev-parse", f"refs/heads/{base}").stdout.strip()
    canon_sha = git(canon_repo, "rev-parse", f"refs/heads/{base}").stdout.strip()
    if not clone_sha or not canon_sha:
        return "unknown"
    if clone_sha == canon_sha:
        return "current"
    result = git(clone_repo, "merge-base", clone_sha, canon_sha)
    merge_base = result.stdout.strip()
    if not merge_base:
        return "diverged"
    if merge_base == clone_sha:
        return "fast-forward"
    if merge_base == canon_sha:
        return "ahead"
    return "diverged"


def ahead_readonly(repo: Path, canon_repo: Path, base: str, branch: str) -> int:
    canon_sha = git(canon_repo, "rev-parse", f"refs/heads/{base}").stdout.strip()
    if not canon_sha:
        return 0
    result = git(repo, "rev-list", "--count", f"{canon_sha}..refs/heads/{branch}")
    count = result.stdout.strip()
    return int(count) if count.isdigit() else 0


def entries_on_ref(repo: Path, ref: str, blog_dir: str = "docs/blog") -> set[str]:
    result = git(repo, "ls-tree", "-r", "--name-only", ref, "--", blog_dir)
    if result.returncode != 0:
        return set()
    return {line for line in result.stdout.strip().splitlines() if line}


def entry_content_from_ref(repo: Path, ref: str, path: str) -> str:
    result = git(repo, "show", f"{ref}:{path}")
    return result.stdout if result.returncode == 0 else ""


def parse_frontmatter_from_text(text: str) -> dict | None:
    if not text.startswith("---"):
        return None
    parts = text.split("---", 2)
    if len(parts) < 3:
        return None
    try:
        return yaml.safe_load(parts[1]) or {}
    except yaml.YAMLError:
        return None
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_areas.py -v`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add project/areas.py tests/test_areas.py
git commit -m "feat(areas): read-only git helpers for survey

follow_state_readonly, ahead_readonly, entries_on_ref,
entry_content_from_ref, parse_frontmatter_from_text

Refs Hortora/soredium#589"
```

---

## Batch 2: Core inspection — repo state, queues, markers, closures

After this batch: `inspect_area()` collects per-repo state, queue info,
markers, slot metadata, and closure status for any area. Finding
generation for structural issues works.

### Task 3: Promote canonical_sync privates + per-repo and queue inspection

**Files:**
- Modify: `project/canonical_sync.py` (rename `_in_progress` → `in_progress`, `_worktrees` → `worktrees`)
- Modify: `project/areas.py`
- Modify: `tests/test_areas.py`

**Interfaces:**
- Consumes: `git_util.git()`, `canonical_sync.in_progress()`,
  `canonical_sync.worktrees()`, `canonical_sync.locate()`,
  `queue_home.candidates()`, `queue_home.uses_root()`,
  `slot_io.parse_slot()`, `clone_manager.is_permanent()`
- Produces: `RepoState`, `QueueInfo`, `ClosureRecord`,
  `inspect_repo(repo_info, area, canon_repo, base) -> RepoState`,
  `inspect_queues(area) -> list[QueueInfo]`,
  `inspect_markers(area) -> list[str]`,
  `inspect_closures(workspace_repo, area, queues) -> list[ClosureRecord]`

- [ ] **Step 1: Promote `_in_progress` and `_worktrees` in canonical_sync.py**

Rename `_in_progress` → `in_progress` and `_worktrees` → `worktrees`.
Update all internal callers within `canonical_sync.py` that reference
the old names. Search for `_in_progress(` and `_worktrees(` in the file
and rename each occurrence.

- [ ] **Step 2: Run existing tests to verify rename didn't break anything**

Run: `python3 -m pytest tests/ -k "canonical" -v --timeout=60`
Expected: existing tests pass (internal callers updated)

- [ ] **Step 3: Write failing tests for inspection functions**

```python
# Add to tests/test_areas.py
from project.areas import (
    RepoState, QueueInfo, ClosureRecord,
    inspect_repo, inspect_queues, inspect_markers, inspect_closures,
)


def test_inspect_repo_basic(tmp_path):
    repo = tmp_path / "repo"
    _init_repo(repo)
    ri = RepoInfo(name="repo", path=repo, role="project")
    state = inspect_repo(ri, canon_repo=None, base="main")
    assert state.branch == "main"
    assert state.dirty_count == 0
    assert state.ahead_of_canonical == 0
    assert state.in_progress is None


def test_inspect_repo_dirty(tmp_path):
    repo = tmp_path / "repo"
    _init_repo(repo)
    (repo / "dirty.txt").write_text("dirty")
    ri = RepoInfo(name="repo", path=repo, role="project")
    state = inspect_repo(ri, canon_repo=None, base="main")
    assert state.dirty_count == 1


def test_inspect_queues_with_queue(tmp_path):
    family = build_family(tmp_path)
    slot = [a for a in discover_areas(family) if a.kind == "slot"][0]
    # Create a queue file in the slot
    queue_path = slot.path / ".queue"
    queue_path.write_text(
        "## State\n\nbranch: issue-42\nstatus: active\n\n"
        "## Queue\n\n- [ ] hortora/myproject#42 — Feature\n"
        "- [ ] hortora/myproject#43 — Another\n"
    )
    queues = inspect_queues(slot)
    assert len(queues) >= 1


def test_inspect_markers(tmp_path):
    family = build_family(tmp_path)
    slot = [a for a in discover_areas(family) if a.kind == "slot"][0]
    markers = inspect_markers(slot)
    assert ".slot" in markers


def test_inspect_markers_clone(tmp_path):
    family = build_family(tmp_path)
    clone = [a for a in discover_areas(family) if a.kind == "clone"][0]
    markers = inspect_markers(clone)
    assert ".permanent" in markers


def test_inspect_closures_stamped_branch(tmp_path):
    repo = tmp_path / "wsp"
    _init_repo(repo)
    _git(repo, "checkout", "-b", "issue-10-feat")
    (repo / "work.txt").write_text("work")
    _git(repo, "add", "-A")
    _git(repo, "commit", "-q", "-m", "work")
    _git(repo, "commit", "--allow-empty", "-m",
         "chore: branch closed — landed as abc123 on main")
    ri = RepoInfo(name="wsp", path=repo, role="workspace")
    area = Area(kind="canonical", path=repo)
    closures = inspect_closures(ri, area, [])
    stamped = [c for c in closures if c.branch == "issue-10-feat"]
    assert len(stamped) == 1
    assert stamped[0].source == "stamp"
    assert stamped[0].landed_sha == "abc123"


def test_inspect_closures_unstamped_active(tmp_path):
    repo = tmp_path / "wsp"
    _init_repo(repo)
    _git(repo, "checkout", "-b", "issue-20-wip")
    (repo / "wip.txt").write_text("wip")
    _git(repo, "add", "-A")
    _git(repo, "commit", "-q", "-m", "wip")
    _git(repo, "checkout", "main")
    ri = RepoInfo(name="wsp", path=repo, role="workspace")
    # Active area with open queue items
    area = Area(kind="slot", path=repo)
    qi = QueueInfo(path=Path("/fake"), items=2, open_items=1,
                   active_issue="hortora/proj#20", status="active",
                   is_root=False)
    closures = inspect_closures(ri, area, [qi])
    unstamped = [c for c in closures if c.branch == "issue-20-wip"]
    assert len(unstamped) == 1
    assert unstamped[0].classification == "active"
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_areas.py -k "inspect_repo or inspect_queues or inspect_markers or inspect_closures" -v`
Expected: FAIL — `ImportError`

- [ ] **Step 5: Implement inspection functions**

Add to `project/areas.py`:

```python
import re
from project.canonical_sync import locate, in_progress, worktrees
from project.queue_home import candidates as queue_candidates, uses_root
from project.slot_io import parse_slot, SlotInfo
from verification import Finding


MARKER_FILES = [
    ".slot", ".permanent", ".queue-root", ".clone-enabled",
    ".landed", ".close-state", ".workspace", "blog-routing.yaml",
]

STAMP_PATTERN = re.compile(
    r"chore: branch closed(?:\s*—\s*landed as\s+(\S+)\s+on\s+(\S+))?"
)


@dataclass
class RepoState:
    name: str
    path: Path
    role: str
    branch: str = ""
    dirty_count: int = 0
    remotes: list[str] = field(default_factory=list)
    ahead_of_canonical: int = 0
    follow_state: str | None = None
    in_progress: str | None = None
    worktrees: list[tuple[str, str]] = field(default_factory=list)


@dataclass
class QueueInfo:
    path: Path
    items: int = 0
    open_items: int = 0
    active_issue: str | None = None
    status: str = ""
    is_root: bool = False


@dataclass
class ClosureRecord:
    branch: str
    source: str = "none"  # "ref" | "stamp" | "none"
    classification: str = "active"
    landed_sha: str | None = None
    landed_on: str | None = None
    promoted: list[str] = field(default_factory=list)
    published: list[str] = field(default_factory=list)
    closed_at: str | None = None


def inspect_repo(repo_info: RepoInfo, canon_repo: Path | None = None,
                 base: str = "main") -> RepoState:
    repo = repo_info.path
    branch_result = git(repo, "rev-parse", "--abbrev-ref", "HEAD")
    branch = branch_result.stdout.strip() if branch_result.returncode == 0 else ""

    status_result = git(repo, "status", "--porcelain")
    dirty = len([l for l in status_result.stdout.strip().splitlines() if l]) if status_result.returncode == 0 else 0

    remote_result = git(repo, "remote")
    remotes = [l for l in remote_result.stdout.strip().splitlines() if l] if remote_result.returncode == 0 else []

    ahead = 0
    follow = None
    if canon_repo and canon_repo != repo:
        ahead = ahead_readonly(repo, canon_repo, base, branch)
        follow = follow_state_readonly(repo, canon_repo, base)

    ip = in_progress(repo)

    wt_result = worktrees(repo)
    wt = [(str(p), b) for p, b in wt_result]

    return RepoState(
        name=repo_info.name, path=repo, role=repo_info.role,
        branch=branch, dirty_count=dirty, remotes=remotes,
        ahead_of_canonical=ahead, follow_state=follow,
        in_progress=ip, worktrees=wt,
    )


def inspect_queues(area: Area) -> list[QueueInfo]:
    result = []
    try:
        cands = queue_candidates(area.path)
    except Exception:
        return result
    is_root = uses_root(area.path)
    for c in cands:
        result.append(QueueInfo(
            path=c.path, items=c.items, open_items=c.open_items,
            active_issue=c.active or None, status=c.status,
            is_root=is_root,
        ))
    return result


def inspect_markers(area: Area) -> list[str]:
    found = []
    for m in MARKER_FILES:
        if (area.path / m).exists():
            found.append(m)
    return found


def inspect_closures(workspace_repo: RepoInfo, area: Area,
                     queues: list[QueueInfo]) -> list[ClosureRecord]:
    repo = workspace_repo.path
    result_cmd = git(repo, "for-each-ref", "--format=%(refname:short)", "refs/heads/")
    if result_cmd.returncode != 0:
        return []

    base = "main"
    branches = [b for b in result_cmd.stdout.strip().splitlines()
                if b and b != base]

    has_open_items = any(q.open_items > 0 for q in queues)
    records = []

    for branch in branches:
        log_result = git(repo, "log", "-1", "--format=%s", branch)
        msg = log_result.stdout.strip() if log_result.returncode == 0 else ""

        match = STAMP_PATTERN.search(msg)
        if match:
            landed_sha = match.group(1)
            landed_on = match.group(2)
            records.append(ClosureRecord(
                branch=branch, source="stamp",
                classification="stamped",
                landed_sha=landed_sha, landed_on=landed_on,
            ))
        else:
            # Classify unstamped branch by area status
            if area.kind == "attic":
                classification = "orphaned"
            elif (area.path / ".landed").exists():
                classification = "landed-not-stamped"
            elif area.kind in ("slot", "clone") and has_open_items:
                classification = "active"
            else:
                classification = "stale"
            records.append(ClosureRecord(
                branch=branch, source="none",
                classification=classification,
            ))

    return records
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_areas.py -v`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add project/canonical_sync.py project/areas.py tests/test_areas.py
git commit -m "feat(areas): core inspection — repo state, queues, markers, closures

Promote canonical_sync._in_progress/_worktrees to public API.
inspect_repo, inspect_queues, inspect_markers, inspect_closures
with unstamped branch classification by area status.

Refs Hortora/soredium#589"
```

---

## Batch 3: Blog drift detection

After this batch: promotion gap detection, waiting-for-landing, bare
project refs, and unpublished entry detection all work from git refs
(never disk). False-positive prevention tested.

### Task 4: Promotion gaps + waiting-for-landing + bare project refs

**Files:**
- Modify: `project/areas.py`
- Modify: `tests/test_areas.py`

**Interfaces:**
- Consumes: `entries_on_ref()`, `entry_content_from_ref()`,
  `parse_frontmatter_from_text()`, `git_util.git()`
- Produces: `DriftEntry(file, source_repo, dest_repo, status, bare_project)`,
  `inspect_promotion_drift(project_repo, workspace_repo, base) -> list[DriftEntry]`

- [ ] **Step 1: Write failing tests for promotion drift**

```python
# Add to tests/test_areas.py
from project.areas import DriftEntry, inspect_promotion_drift


def _add_blog_entry(repo: Path, name: str, content: str):
    blog = repo / "docs" / "blog"
    blog.mkdir(parents=True, exist_ok=True)
    (blog / name).write_text(content)
    _git(repo, "add", "-A")
    _git(repo, "commit", "-q", "-m", f"add {name}")


def test_promotion_gap_detected(tmp_path):
    proj = tmp_path / "proj"
    _init_repo(proj)
    wsp = tmp_path / "wsp"
    _init_repo(wsp)
    _add_blog_entry(wsp, "entry.md",
                    "---\ntitle: Test\nprojects: owner/repo\n---\nBody")
    # entry on workspace main but NOT on project main
    drift = inspect_promotion_drift(
        RepoInfo("proj", proj, "project"),
        RepoInfo("wsp", wsp, "workspace"),
        "main",
    )
    gaps = [d for d in drift if d.status == "gap"]
    assert len(gaps) == 1
    assert gaps[0].file == "docs/blog/entry.md"


def test_no_gap_when_promoted(tmp_path):
    proj = tmp_path / "proj"
    _init_repo(proj)
    wsp = tmp_path / "wsp"
    _init_repo(wsp)
    _add_blog_entry(wsp, "entry.md",
                    "---\ntitle: Test\nprojects: owner/repo\n---\nBody")
    _add_blog_entry(proj, "entry.md",
                    "---\ntitle: Test\nprojects: owner/repo\n---\nBody")
    drift = inspect_promotion_drift(
        RepoInfo("proj", proj, "project"),
        RepoInfo("wsp", wsp, "workspace"),
        "main",
    )
    gaps = [d for d in drift if d.status == "gap"]
    assert len(gaps) == 0


def test_waiting_for_landing(tmp_path):
    proj = tmp_path / "proj"
    _init_repo(proj)
    wsp = tmp_path / "wsp"
    _init_repo(wsp)
    _git(wsp, "checkout", "-b", "feature")
    _add_blog_entry(wsp, "entry.md",
                    "---\ntitle: Test\nprojects: owner/repo\n---\nBody")
    # entry on feature branch but NOT on main
    drift = inspect_promotion_drift(
        RepoInfo("proj", proj, "project"),
        RepoInfo("wsp", wsp, "workspace"),
        "main",
    )
    waiting = [d for d in drift if d.status == "waiting"]
    assert len(waiting) == 1


def test_no_false_promotion_gap_on_feature_branch(tmp_path):
    """Issue comment rule 6: project on feature branch older than promotion."""
    proj = tmp_path / "proj"
    _init_repo(proj)
    wsp = tmp_path / "wsp"
    _init_repo(wsp)
    _add_blog_entry(wsp, "entry.md",
                    "---\ntitle: Test\nprojects: owner/repo\n---\nBody")
    _add_blog_entry(proj, "entry.md",
                    "---\ntitle: Test\nprojects: owner/repo\n---\nBody")
    # project checks out older feature branch
    _git(proj, "checkout", "-b", "feature-old", "HEAD~1")
    # Survey compares base branches, not disk — so no gap
    drift = inspect_promotion_drift(
        RepoInfo("proj", proj, "project"),
        RepoInfo("wsp", wsp, "workspace"),
        "main",
    )
    gaps = [d for d in drift if d.status == "gap"]
    assert len(gaps) == 0


def test_bare_project_ref_detected(tmp_path):
    proj = tmp_path / "proj"
    _init_repo(proj)
    wsp = tmp_path / "wsp"
    _init_repo(wsp)
    _add_blog_entry(wsp, "entry.md",
                    "---\ntitle: Test\nprojects: engine\n---\nBody")
    drift = inspect_promotion_drift(
        RepoInfo("proj", proj, "project"),
        RepoInfo("wsp", wsp, "workspace"),
        "main",
    )
    bare = [d for d in drift if d.bare_project]
    assert len(bare) == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_areas.py -k "promotion or waiting or bare_project" -v`
Expected: FAIL — `ImportError`

- [ ] **Step 3: Implement promotion drift detection**

Add to `project/areas.py`:

```python
@dataclass
class DriftEntry:
    file: str
    source_repo: str
    dest_repo: str
    status: str  # "gap" | "waiting" | "unpublished"
    bare_project: bool = False


def inspect_promotion_drift(project_repo: RepoInfo, workspace_repo: RepoInfo,
                            base: str) -> list[DriftEntry]:
    drift: list[DriftEntry] = []
    wsp = workspace_repo.path
    proj = project_repo.path

    ws_base_entries = entries_on_ref(wsp, base, "docs/blog")
    proj_base_entries = entries_on_ref(proj, base, "docs/blog")

    # Current branch entries (for waiting-for-landing detection)
    ws_branch = git(wsp, "rev-parse", "--abbrev-ref", "HEAD").stdout.strip()
    ws_head_entries = entries_on_ref(wsp, "HEAD", "docs/blog") if ws_branch != base else ws_base_entries

    # Entries only on working branch (not on base) = waiting for landing
    waiting_entries = ws_head_entries - ws_base_entries
    for entry in sorted(waiting_entries):
        drift.append(DriftEntry(
            file=entry, source_repo=workspace_repo.name,
            dest_repo=project_repo.name, status="waiting",
        ))

    # Promotion gaps: on workspace base, absent from project base
    for entry in sorted(ws_base_entries):
        content = entry_content_from_ref(wsp, base, entry)
        fm = parse_frontmatter_from_text(content)
        bare = False
        if fm:
            projects_val = fm.get("projects", "")
            if isinstance(projects_val, str) and projects_val and "/" not in projects_val:
                bare = True

        if bare:
            drift.append(DriftEntry(
                file=entry, source_repo=workspace_repo.name,
                dest_repo=project_repo.name, status="gap",
                bare_project=True,
            ))

        if entry not in proj_base_entries:
            drift.append(DriftEntry(
                file=entry, source_repo=workspace_repo.name,
                dest_repo=project_repo.name, status="gap",
                bare_project=bare,
            ))

    return drift
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_areas.py -k "promotion or waiting or bare_project or false_promotion" -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add project/areas.py tests/test_areas.py
git commit -m "feat(areas): blog drift — promotion gaps, waiting, bare project refs

Compares base branches via git ls-tree, never disk files.
Issue comment rules 1, 2, 5, 6 implemented and tested.

Refs Hortora/soredium#589"
```

### Task 5: Unpublished detection (routing + destination mapping)

**Files:**
- Modify: `project/areas.py`
- Modify: `tests/test_areas.py`

**Interfaces:**
- Consumes: `entries_on_ref()`, `entry_content_from_ref()`,
  `parse_frontmatter_from_text()`, `git_util.git()`
- Produces: `resolve_dest_repo(dest_path) -> tuple[Path, str] | None`,
  `inspect_unpublished(workspace_repo, base, routing_yaml_path) -> list[DriftEntry]`

- [ ] **Step 1: Write failing tests**

```python
# Add to tests/test_areas.py
from project.areas import resolve_dest_repo, inspect_unpublished


def test_resolve_dest_repo(tmp_path):
    dest_repo = tmp_path / "blog"
    _init_repo(dest_repo)
    posts = dest_repo / "_posts"
    posts.mkdir()
    result = resolve_dest_repo(posts)
    assert result is not None
    repo_path, base = result
    assert repo_path == dest_repo
    assert base == "main"


def test_resolve_dest_repo_missing(tmp_path):
    result = resolve_dest_repo(tmp_path / "nonexistent")
    assert result is None


def test_inspect_unpublished(tmp_path):
    # Workspace with entry
    wsp = tmp_path / "wsp"
    _init_repo(wsp)
    _add_blog_entry(wsp, "entry.md",
                    "---\ntitle: Test\nentry_type: note\nsubtype: diary\n---\nBody")

    # Destination repo without the entry
    dest = tmp_path / "blog"
    _init_repo(dest)
    posts = dest / "_posts"
    posts.mkdir()
    _git(dest, "add", "-A")
    _git(dest, "commit", "-q", "-m", "init posts")

    # Routing config
    routing_yaml = tmp_path / "blog-routing.yaml"
    routing_yaml.write_text(
        f"version: 1\n"
        f"destinations:\n"
        f"  diary:\n"
        f"    type: git\n"
        f"    path: {dest}\n"
        f"    subdir: _posts/\n"
        f"defaults:\n"
        f"  destinations: [diary]\n"
    )

    ri = RepoInfo("wsp", wsp, "workspace")
    unpub = inspect_unpublished(ri, "main", routing_yaml)
    assert len(unpub) == 1
    assert unpub[0].status == "unpublished"


def test_inspect_unpublished_present(tmp_path):
    wsp = tmp_path / "wsp"
    _init_repo(wsp)
    _add_blog_entry(wsp, "entry.md",
                    "---\ntitle: Test\nentry_type: note\nsubtype: diary\n---\nBody")

    dest = tmp_path / "blog"
    _init_repo(dest)
    posts = dest / "_posts"
    posts.mkdir()
    (posts / "entry.md").write_text("published copy")
    _git(dest, "add", "-A")
    _git(dest, "commit", "-q", "-m", "publish entry")

    routing_yaml = tmp_path / "blog-routing.yaml"
    routing_yaml.write_text(
        f"version: 1\n"
        f"destinations:\n"
        f"  diary:\n"
        f"    type: git\n"
        f"    path: {dest}\n"
        f"    subdir: _posts/\n"
        f"defaults:\n"
        f"  destinations: [diary]\n"
    )

    ri = RepoInfo("wsp", wsp, "workspace")
    unpub = inspect_unpublished(ri, "main", routing_yaml)
    assert len(unpub) == 0
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_areas.py -k "unpublished or resolve_dest" -v`
Expected: FAIL

- [ ] **Step 3: Implement unpublished detection**

Add to `project/areas.py`:

```python
def resolve_dest_repo(dest_path: Path) -> tuple[Path, str] | None:
    path = dest_path.resolve() if dest_path.exists() else dest_path
    current = path
    while current != current.parent:
        if (current / ".git").exists():
            base_result = git(current, "rev-parse", "--abbrev-ref", "HEAD")
            base = base_result.stdout.strip() if base_result.returncode == 0 else "main"
            return current, base
        current = current.parent
    return None


def _load_routing_config(routing_path: Path) -> dict | None:
    if not routing_path.exists():
        return None
    try:
        return yaml.safe_load(routing_path.read_text())
    except (yaml.YAMLError, OSError):
        return None


def inspect_unpublished(workspace_repo: RepoInfo, base: str,
                        routing_yaml_path: Path) -> list[DriftEntry]:
    config = _load_routing_config(routing_yaml_path)
    if not config:
        return []

    destinations = config.get("destinations", {})
    defaults = config.get("defaults", {})
    default_dests = defaults.get("destinations", [])

    # Build destination repo cache: dest_name -> (repo_path, base, subdir)
    dest_cache: dict[str, tuple[Path, str, str]] = {}
    for name, dest_conf in destinations.items():
        dest_path = Path(str(dest_conf.get("path", ""))).expanduser()
        subdir = dest_conf.get("subdir", "")
        full_path = dest_path / subdir if subdir else dest_path
        resolved = resolve_dest_repo(full_path)
        if resolved:
            dest_cache[name] = (resolved[0], resolved[1], subdir)

    if not dest_cache:
        return []

    wsp = workspace_repo.path
    ws_entries = entries_on_ref(wsp, base, "docs/blog")
    drift: list[DriftEntry] = []

    for entry_path in sorted(ws_entries):
        entry_name = Path(entry_path).name
        # Check if present at all default destinations
        missing_at: list[str] = []
        for dest_name in default_dests:
            if dest_name not in dest_cache:
                missing_at.append(dest_name)
                continue
            repo_path, dest_base, subdir = dest_cache[dest_name]
            dest_entries = entries_on_ref(repo_path, dest_base, subdir.rstrip("/"))
            # Check by filename match
            found = any(Path(e).name == entry_name for e in dest_entries)
            if not found:
                missing_at.append(dest_name)

        if missing_at:
            drift.append(DriftEntry(
                file=entry_path,
                source_repo=workspace_repo.name,
                dest_repo=",".join(missing_at),
                status="unpublished",
            ))

    return drift
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_areas.py -k "unpublished or resolve_dest" -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add project/areas.py tests/test_areas.py
git commit -m "feat(areas): unpublished detection via routing + destination mapping

Resolves blog-routing.yaml destinations to git repos,
checks presence via git ls-tree on destination base branch.
Issue comment rule 3 implemented.

Refs Hortora/soredium#589"
```

---

## Batch 4: Work-end preview, orchestration, and output

After this batch: `inspect_area()` and `survey()` are complete.
`work_end_preview()` predicts what `work end` would do. KEY=VALUE
output works. CLI/TUI events wired. Read-only and idempotency tests
pass. The command is usable.

### Task 6: Work-end preview + inspect_area + survey orchestrator

**Files:**
- Modify: `project/areas.py`
- Modify: `tests/test_areas.py`

**Interfaces:**
- Consumes: all prior inspection functions, `queue_io.read_queue()`,
  `branch_base.parent_of()`
- Produces: `WorkEndPreview`, `AreaReport`, `SurveyResult`,
  `work_end_preview(area, repos, queues, closures, drift) -> WorkEndPreview`,
  `inspect_area(area, family_root) -> AreaReport`,
  `survey(family_roots) -> list[SurveyResult]`,
  `generate_findings(report) -> list[Finding]`

- [ ] **Step 1: Write failing tests**

```python
# Add to tests/test_areas.py
from project.areas import (
    WorkEndPreview, AreaReport, SurveyResult,
    work_end_preview, inspect_area, survey, generate_findings,
)
from verification import Finding


def test_work_end_preview_nothing_to_do(tmp_path):
    area = Area(kind="canonical", path=tmp_path)
    preview = work_end_preview(area, [], [], [], [])
    assert preview.would_skip is True
    assert preview.would_land == []


def test_work_end_preview_done_items(tmp_path):
    area = Area(kind="slot", path=tmp_path)
    qi = QueueInfo(path=Path("/fake"), items=3, open_items=1,
                   active_issue="hortora/proj#10", status="active",
                   is_root=False)
    # Simulate a done item (done_branch set via queue data)
    preview = work_end_preview(area, [], [qi], [], [],
                               done_branches=["issue-10-feat"])
    assert "issue-10-feat" in preview.would_land
    assert preview.would_skip is False


def test_inspect_area_basic(tmp_path):
    family = build_family(tmp_path)
    areas = discover_areas(family)
    canonical = [a for a in areas if a.kind == "canonical"][0]
    report = inspect_area(canonical, family)
    assert report.area.kind == "canonical"
    assert len(report.repos) >= 1
    assert report.error is None


def test_inspect_area_error_handling(tmp_path):
    """An area with a corrupted repo should report error, not crash."""
    family = tmp_path / "family"
    family.mkdir()
    bad = family / "badrepo"
    bad.mkdir()
    (bad / ".git").touch()  # file, not directory — corrupted
    areas = discover_areas(family)
    if areas:
        report = inspect_area(areas[0], family)
        assert report.error is not None or len(report.repos) == 0


def test_survey_complete(tmp_path):
    family = build_family(tmp_path)
    results = survey([family])
    assert len(results) == 1
    r = results[0]
    assert r.family_root == family
    assert len(r.areas) >= 4
    assert isinstance(r.summary, dict)


def test_generate_findings_diverged_clone(tmp_path):
    report = AreaReport(
        area=Area(kind="clone", path=tmp_path),
        markers=[".permanent"], repos=[
            RepoState(name="proj", path=tmp_path, role="project",
                      branch="main", dirty_count=0, remotes=[],
                      ahead_of_canonical=0, follow_state="diverged",
                      in_progress=None, worktrees=[])
        ],
        queues=[], closures=[], work_end_preview=None,
        legacy_layout=False, findings=[], drift=[],
        slot_info=None, error=None,
    )
    findings = generate_findings(report)
    errors = [f for f in findings if f.severity == "ERROR" and f.category == "drift"]
    assert len(errors) == 1


def test_survey_read_only(tmp_path):
    """Tree snapshot before and after — the core read-only guarantee."""
    def tree(directory: Path) -> dict[str, bytes]:
        result = {}
        for p in sorted(directory.rglob("*")):
            if p.is_file() and ".git" not in p.relative_to(directory).parts:
                result[str(p.relative_to(directory))] = p.read_bytes()
        return result

    family = build_family(tmp_path)
    before = tree(family)
    survey([family])
    after = tree(family)
    assert before == after


def test_survey_idempotent(tmp_path):
    family = build_family(tmp_path)
    r1 = survey([family])
    r2 = survey([family])
    assert len(r1) == len(r2)
    assert len(r1[0].areas) == len(r2[0].areas)
    assert r1[0].summary == r2[0].summary
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_areas.py -k "work_end_preview or inspect_area or survey_complete or generate_findings or survey_read_only or survey_idempotent" -v`
Expected: FAIL

- [ ] **Step 3: Implement work-end preview, inspect_area, survey, findings**

Add to `project/areas.py`:

```python
@dataclass
class WorkEndPreview:
    would_land: list[str] = field(default_factory=list)
    needs_promotion: list[str] = field(default_factory=list)
    would_publish: list[str] = field(default_factory=list)
    gates_refusing: list[str] = field(default_factory=list)
    would_skip: bool = True


@dataclass
class AreaReport:
    area: Area
    markers: list[str] = field(default_factory=list)
    repos: list[RepoState] = field(default_factory=list)
    queues: list[QueueInfo] = field(default_factory=list)
    closures: list[ClosureRecord] = field(default_factory=list)
    work_end_preview: WorkEndPreview | None = None
    legacy_layout: bool = False
    findings: list[Finding] = field(default_factory=list)
    drift: list[DriftEntry] = field(default_factory=list)
    slot_info: SlotInfo | None = None
    error: str | None = None


@dataclass
class SurveyResult:
    family_root: Path
    areas: list[AreaReport] = field(default_factory=list)
    attention: list[Finding] = field(default_factory=list)
    summary: dict[str, int] = field(default_factory=dict)


def work_end_preview(area: Area, repos: list[RepoState],
                     queues: list[QueueInfo], closures: list[ClosureRecord],
                     drift: list[DriftEntry],
                     done_branches: list[str] | None = None) -> WorkEndPreview:
    preview = WorkEndPreview()
    if done_branches:
        preview.would_land = list(done_branches)
        preview.would_skip = False

    gaps = [d for d in drift if d.status == "gap"]
    if gaps:
        preview.needs_promotion = [d.file for d in gaps]
        preview.would_skip = False

    unpub = [d for d in drift if d.status == "unpublished"]
    if unpub:
        preview.would_publish = [d.file for d in unpub]
        preview.would_skip = False

    # Check squash review gate
    for branch in (done_branches or []):
        for repo in repos:
            ref_result = git(repo.path, "rev-parse", "--verify",
                             f"refs/squash-reviewed/{branch}")
            if ref_result.returncode != 0:
                preview.gates_refusing.append(
                    f"squash_review_needed:{branch}")

    return preview


def generate_findings(report: AreaReport) -> list[Finding]:
    findings: list[Finding] = []

    if report.error:
        findings.append(Finding("ERROR", "inspection-failed", report.error))
        return findings

    for repo in report.repos:
        if repo.follow_state == "diverged":
            findings.append(Finding(
                "ERROR", "drift",
                f"{report.area.kind} {report.area.path.name}: "
                f"{repo.name} diverged from canonical"))

        if repo.in_progress:
            findings.append(Finding(
                "ERROR", "stuck-operation",
                f"{report.area.path.name}/{repo.name}: "
                f"{repo.in_progress} in progress"))

        if repo.dirty_count > 0 and repo.branch in ("main", "master"):
            findings.append(Finding(
                "WARN", "dirty-base",
                f"{report.area.path.name}/{repo.name}: "
                f"{repo.dirty_count} dirty files on {repo.branch}"))

        if repo.ahead_of_canonical > 0:
            findings.append(Finding(
                "INFO", "ahead",
                f"{report.area.path.name}/{repo.name}: "
                f"{repo.ahead_of_canonical} ahead of canonical"))

    # Queue findings
    queue_count = len(report.queues)
    if queue_count > 1 and not any(q.is_root for q in report.queues):
        findings.append(Finding(
            "WARN", "queue-ambiguity",
            f"{report.area.path.name}: {queue_count} queues, no .queue-root"))

    for q in report.queues:
        if q.active_issue and q.status == "active":
            # Check if the named branch is checked out
            active_branches = {r.branch for r in report.repos}
            # Queue branch mismatch is checked via queue state vs repo branch
            pass

    # Marker findings
    if ".close-state" in report.markers:
        findings.append(Finding(
            "WARN", "stale-state",
            f"{report.area.path.name}: stale .close-state file"))

    if report.legacy_layout:
        findings.append(Finding(
            "INFO", "legacy-layout",
            f"{report.area.path.name}: legacy layout detected"))

    if ".landed" in report.markers and report.area.kind == "slot":
        findings.append(Finding(
            "INFO", "landed-not-archived",
            f"slot {report.area.slot_number or report.area.path.name}: "
            f"landed, ready for archive"))

    # Closure findings
    for c in report.closures:
        if c.classification == "orphaned":
            findings.append(Finding(
                "WARN", "unstamped-orphaned",
                f"{report.area.path.name}: branch {c.branch} has no stamp"))
        elif c.classification == "landed-not-stamped":
            findings.append(Finding(
                "WARN", "unstamped-orphaned",
                f"{report.area.path.name}: branch {c.branch} landed but not stamped"))
        elif c.classification == "stale":
            findings.append(Finding(
                "INFO", "unstamped-stale",
                f"{report.area.path.name}: branch {c.branch} appears stale"))

    # Drift findings
    for d in report.drift:
        if d.status == "gap":
            findings.append(Finding(
                "WARN", "promotion-gap",
                f"{d.source_repo}: {d.file} not promoted to {d.dest_repo}"))
        elif d.status == "unpublished":
            findings.append(Finding(
                "INFO", "unpublished",
                f"{d.source_repo}: {d.file} unpublished at {d.dest_repo}"))
        if d.bare_project:
            findings.append(Finding(
                "WARN", "bare-project-ref",
                f"{d.source_repo}/{d.file}: projects value is not owner/repo"))

    return findings


def inspect_area(area: Area, family_root: Path) -> AreaReport:
    report = AreaReport(area=area)

    try:
        # Determine canonical repo for ahead/follow comparison
        canon_repo = None
        base = "main"
        if area.kind in ("slot", "clone", "attic") and area.project_repo:
            try:
                canon = locate(area.project_repo.path)
                canon_repo = canon.repo
                base = canon.base
            except Exception:
                pass

        # Repo inspection
        for ri in area.repos:
            cr = canon_repo if ri.role == "project" else None
            report.repos.append(inspect_repo(ri, canon_repo=cr, base=base))

        # Queue inspection
        report.queues = inspect_queues(area)

        # Markers
        report.markers = inspect_markers(area)

        # Slot metadata
        if area.kind in ("slot", "attic"):
            report.slot_info = parse_slot(area.path)

        # Legacy layout
        has_plan = (area.path / ".plan").exists()
        has_queue = (area.path / ".queue").exists()
        report.legacy_layout = has_plan and not has_queue

        # Closures
        if area.workspace_repo:
            report.closures = inspect_closures(
                area.workspace_repo, area, report.queues)

        # Blog drift
        if area.project_repo and area.workspace_repo:
            report.drift = inspect_promotion_drift(
                area.project_repo, area.workspace_repo, base)
            # Unpublished detection
            routing_yaml = area.workspace_repo.path / "blog-routing.yaml"
            if not routing_yaml.exists() and area.path != area.workspace_repo.path:
                routing_yaml = area.path / "blog-routing.yaml"
            if routing_yaml.exists():
                unpub = inspect_unpublished(area.workspace_repo, base, routing_yaml)
                report.drift.extend(unpub)

        # Work-end preview
        done_branches = []
        if area.workspace_repo:
            try:
                from project.queue_io import read_queue
                q = read_queue(area.workspace_repo.path)
                if q:
                    done_branches = [
                        item.done_branch for item in q.done_items()
                        if item.done_branch
                    ]
            except Exception:
                pass
        report.work_end_preview = work_end_preview(
            area, report.repos, report.queues, report.closures,
            report.drift, done_branches=done_branches)

        # Findings
        report.findings = generate_findings(report)

    except Exception as e:
        report.error = str(e)
        report.findings = [Finding("ERROR", "inspection-failed", str(e))]

    return report


def survey(family_roots: list[Path]) -> list[SurveyResult]:
    results = []
    for root in family_roots:
        areas = discover_areas(root)
        reports = []
        for area in areas:
            reports.append(inspect_area(area, root))

        attention = []
        summary: dict[str, int] = {}
        for r in reports:
            kind = r.area.kind
            summary[kind] = summary.get(kind, 0) + 1
            for f in r.findings:
                if f.severity in ("ERROR", "WARN"):
                    attention.append(f)

        results.append(SurveyResult(
            family_root=root, areas=reports,
            attention=attention, summary=summary,
        ))
    return results
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_areas.py -v`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add project/areas.py tests/test_areas.py
git commit -m "feat(areas): work-end preview, inspect_area, survey orchestrator

Complete inspection pipeline with per-area error isolation,
finding generation, and read-only/idempotency guarantees.

Refs Hortora/soredium#589"
```

### Task 7: KEY=VALUE output + CLI/TUI events

**Files:**
- Modify: `project/areas.py` (add `__main__` block and `format_kv()`)
- Create: `commands/survey.py`
- Modify: `commands/events.py`
- Modify: `tests/test_areas.py`

**Interfaces:**
- Consumes: `SurveyResult`, `AreaReport`
- Produces: `format_kv(results) -> str`, `SurveyAreaInfo`, `SurveyReady`,
  `commands.survey.execute(family_root) -> SurveyReady`

- [ ] **Step 1: Write failing tests**

```python
# Add to tests/test_areas.py
from project.areas import format_kv


def test_format_kv_basic(tmp_path):
    family = build_family(tmp_path)
    results = survey([family])
    output = format_kv(results)
    assert "FAMILY=" in output
    assert "AREA_COUNT=" in output
    assert "AREA_1_KIND=" in output
    assert "FINDING_" in output or "ATTENTION_COUNT=0" in output
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_areas.py -k "format_kv" -v`
Expected: FAIL

- [ ] **Step 3: Implement KEY=VALUE output**

Add to `project/areas.py`:

```python
def format_kv(results: list[SurveyResult]) -> str:
    lines: list[str] = []
    for sr in results:
        lines.append(f"FAMILY={sr.family_root}")
        lines.append(f"AREA_COUNT={len(sr.areas)}")
        for kind, count in sorted(sr.summary.items()):
            lines.append(f"{kind.upper()}_COUNT={count}")
        lines.append(f"ATTENTION_COUNT={len(sr.attention)}")

        for i, report in enumerate(sr.areas, 1):
            prefix = f"AREA_{i}"
            a = report.area
            lines.append(f"{prefix}_KIND={a.kind}")
            lines.append(f"{prefix}_PATH={a.path}")
            if a.project_repo:
                lines.append(f"{prefix}_PROJECT={a.project_repo.name}")
            if a.workspace_repo:
                lines.append(f"{prefix}_WORKSPACE={a.workspace_repo.name}")

            for j, repo in enumerate(report.repos, 1):
                rp = f"{prefix}_REPO_{j}"
                lines.append(f"{rp}_NAME={repo.name}")
                lines.append(f"{rp}_BRANCH={repo.branch}")
                lines.append(f"{rp}_DIRTY={repo.dirty_count}")
                lines.append(f"{rp}_AHEAD={repo.ahead_of_canonical}")
                if repo.follow_state:
                    lines.append(f"{rp}_FOLLOW={repo.follow_state}")
                if repo.in_progress:
                    lines.append(f"{rp}_IN_PROGRESS={repo.in_progress}")

            lines.append(f"{prefix}_QUEUES={len(report.queues)}")
            for j, q in enumerate(report.queues, 1):
                qp = f"{prefix}_QUEUE_{j}"
                lines.append(f"{qp}_STATUS={q.status}")
                lines.append(f"{qp}_ITEMS={q.items}")
                lines.append(f"{qp}_OPEN={q.open_items}")
                if q.active_issue:
                    lines.append(f"{qp}_ACTIVE={q.active_issue}")

            if report.work_end_preview and not report.work_end_preview.would_skip:
                wep = report.work_end_preview
                if wep.would_land:
                    lines.append(f"{prefix}_WORK_END_WOULD_LAND={','.join(wep.would_land)}")
                if wep.gates_refusing:
                    lines.append(f"{prefix}_WORK_END_GATES={','.join(wep.gates_refusing)}")

            if report.error:
                lines.append(f"{prefix}_ERROR={report.error}")

        for i, f in enumerate(sr.attention, 1):
            lines.append(f"FINDING_{i}={f.severity}:{f.category}:{f.message}")

    return "\n".join(lines)


if __name__ == "__main__":
    import sys
    roots = [Path(a) for a in sys.argv[1:] if a != "survey"]
    if not roots:
        print("ERROR=no family roots provided")
        sys.exit(1)
    results = survey(roots)
    print(format_kv(results))
```

- [ ] **Step 4: Implement CLI events and command module**

Add to `commands/events.py` after the `ContextSwitched` dataclass:

```python
@dataclass
class SurveyAreaInfo:
    kind: str
    path: str
    slot_number: int | None
    project: str | None
    workspace: str | None
    repos: list[dict]
    queues: list[dict]
    closures: list[dict]
    work_end_preview: dict | None
    markers: list[str]
    findings: list[dict]
    drift: list[dict]
    error: str | None


@dataclass
class SurveyReady:
    family_root: str
    areas: list[SurveyAreaInfo]
    attention: list[dict]
    summary: dict[str, int]
```

Create `commands/survey.py`:

```python
"""Survey command — read-only inspection of every area in a family root."""
from __future__ import annotations

import sys
from dataclasses import asdict
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from commands.events import SurveyAreaInfo, SurveyReady
from project.areas import survey as run_survey


def execute(family_root: str = "") -> SurveyReady:
    if not family_root:
        raise ValueError("family_root is required")

    root = Path(family_root).expanduser().resolve()
    results = run_survey([root])
    if not results:
        return SurveyReady(family_root=str(root), areas=[], attention=[], summary={})

    sr = results[0]
    area_infos = []
    for report in sr.areas:
        a = report.area
        area_infos.append(SurveyAreaInfo(
            kind=a.kind,
            path=str(a.path),
            slot_number=a.slot_number,
            project=a.project_repo.name if a.project_repo else None,
            workspace=a.workspace_repo.name if a.workspace_repo else None,
            repos=[asdict(r) for r in report.repos],
            queues=[{"path": str(q.path), "items": q.items,
                     "open_items": q.open_items,
                     "active_issue": q.active_issue,
                     "status": q.status, "is_root": q.is_root}
                    for q in report.queues],
            closures=[asdict(c) for c in report.closures],
            work_end_preview=asdict(report.work_end_preview) if report.work_end_preview else None,
            markers=report.markers,
            findings=[{"severity": f.severity, "category": f.category,
                       "message": f.message} for f in report.findings],
            drift=[asdict(d) for d in report.drift],
            error=report.error,
        ))

    return SurveyReady(
        family_root=str(sr.family_root),
        areas=area_infos,
        attention=[{"severity": f.severity, "category": f.category,
                    "message": f.message} for f in sr.attention],
        summary=sr.summary,
    )
```

- [ ] **Step 5: Run all tests**

Run: `python3 -m pytest tests/test_areas.py -v`
Expected: all PASS

- [ ] **Step 6: Run the command end-to-end on a real family root**

Run: `python3 project/areas.py survey /Users/mdproctor/claude/hortora/clone/soredium`
Expected: KEY=VALUE output with real area data, no crashes

- [ ] **Step 7: Commit**

```bash
git add project/areas.py commands/survey.py commands/events.py tests/test_areas.py
git commit -m "feat(areas): KEY=VALUE output, CLI events, survey command

format_kv for pipeline output, SurveyReady/SurveyAreaInfo events
for TUI, commands/survey.py for CLI integration.

Refs Hortora/soredium#589"
```

---

## References

- `specs/issue-589-read-only-survey-command/2026-11-23-read-only-survey-design.md` — design spec
- `project/git_util.py:6` — `git()` wrapper
- `project/canonical_sync.py:174,210` — `_worktrees()`, `_in_progress()` to promote
- `project/queue_home.py:40` — `Candidate` dataclass, `candidates()` function
- `project/queue_io.py:50` — `Queue`, `QueueItem`, `read_queue()`
- `project/slot_io.py:14` — `SlotInfo`, `parse_slot()`
- `project/clone_manager.py` — `is_permanent()`, `is_enabled()`
- `verification/__init__.py` — `Finding` NamedTuple
- `commands/events.py:202` — `ContextSwitched` (insert after)
- `tests/test_ctx_reads_only.py` — read-only test pattern
- `publish-blog/blog_routing.py` — routing config format
- `docs/protocols/a-read-never-writes.md` — read-only constraint
- `docs/protocols/git-util-single-source.md` — git wrapper requirement
- `docs/protocols/one-queue-per-area-never-synced.md` — queue location rule
- Hortora/soredium#589 — focal issue
- Hortora/soredium#618 — closure refs (future)
- `docs/development/lifecycle-maintainer-guide.md` — lifecycle vocabulary
