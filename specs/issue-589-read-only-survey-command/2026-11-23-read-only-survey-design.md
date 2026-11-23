# Read-only Survey Command — Design Spec

**Issue:** Hortora/soredium#589
**Branch:** issue-589-read-only-survey-command
**Date:** 2026-11-23

## Purpose

A read-only command that surveys every area in a family root — canonical
repos, slots, clones, attic — and reports their state: branches, dirty
counts, remotes, ahead-of-canonical, queue status, legacy layout, blog
drift, and closure status. One row per area, attention items summarised
at the end. Output is KEY=VALUE for pipeline scripts and JSON Lines
events for the TUI.

The hand-survey that motivated this found a diverged clone, queues
naming branches not checked out, a stale `.close-state`, and five
differing queues in one slot. It also found 33 blog promotion gaps of
which 32 were false positives — the files were on feature branches, not
the base branch. This command replaces that manual process with
repeatable, correct drift detection.

## Scope

- `project/areas.py` — discovery, inspection, reporting (new file)
- `commands/survey.py` — CLI/TUI event wrapper (new file)
- `commands/events.py` — `SurveyReady` event type (extend)
- `tests/test_areas.py` — comprehensive tests including read-only proof (new file)

Out of scope: writing closure refs (#618), modifying the close ceremony,
TUI view implementation.

## Architecture

Three layers, each independently testable:

```
discover_areas(family_root)  →  list[Area]
inspect_area(area, family)   →  AreaReport
survey(family_roots)         →  list[SurveyResult]
```

### Layer 1: Discovery

`discover_areas(family_root: Path) -> list[Area]`

Walks the filesystem to find every area. No git operations, no file
reads beyond `Path.exists()` / `Path.iterdir()` / `Path.is_dir()`.

**Area kinds:**

| Kind | Location | Marker |
|------|----------|--------|
| `canonical` | Top-level children of family root with `.git`, not under `slots/`/`clone/`/`attic/` | `.git` directory |
| `slot` | `slots/<N>/` | `.slot` file |
| `clone` | `clone/<repo>/` | `.permanent` file |
| `attic` | `slots/attic/<N>/` | `.slot` file |
| `standalone-workspace` | Workspace dir without paired project | No `proj` symlink |

**Ordering:** canonical first (alphabetical), then slots (by number),
then clones (alphabetical), then attic (by number), then standalone
workspaces.

### Layer 2: Inspection

`inspect_area(area: Area, family_root: Path) -> AreaReport`

For each discovered area, collects state using existing building blocks.
All git operations via `git_util.git()`. All reads via existing library
functions.

**Per-repo state** (every git repo in the area):

| Field | Source |
|-------|--------|
| Current branch | `git rev-parse --abbrev-ref HEAD` |
| Dirty file count | `git status --porcelain` line count |
| Remotes | `git remote` |
| Ahead-of-canonical | `canonical_sync.ahead_of_canonical()` |
| Follow state | Inline SHA comparison (see below) |
| In-progress operation | `canonical_sync._in_progress()` |
| Worktrees | `canonical_sync._worktrees()` |

**Queue state** (via `queue_home.candidates()`):

| Field | Source |
|-------|--------|
| Queue paths | `queue_home.candidates()` |
| Items / open items | Each candidate's `items` / `open_items` |
| Active issue | Each candidate's `active` |
| Queue status | Each candidate's `status` |
| Root queue marker | `queue_home.uses_root()` |

**Follow state (read-only inline):**

`canonical_sync.follow()` is NOT safe for the survey — it fetches and
fast-forwards. The survey inlines the read-only comparison:

```python
def follow_state_readonly(clone_repo: Path, canon_repo: Path, base: str) -> str:
    clone_sha = git(clone_repo, "rev-parse", f"refs/heads/{base}").stdout.strip()
    canon_sha = git(canon_repo, "rev-parse", f"refs/heads/{base}").stdout.strip()
    if clone_sha == canon_sha:
        return "current"
    merge_base = git(clone_repo, "merge-base", clone_sha, canon_sha).stdout.strip()
    if merge_base == clone_sha:
        return "fast-forward"   # clone is behind canonical
    if merge_base == canon_sha:
        return "ahead"          # clone has commits canonical lacks
    return "diverged"           # both have unique commits
```

No fetch, no merge, no mutation. Uses `git_util.git()` which never raises.

**Markers** (via `Path.exists()`):

`.slot`, `.permanent`, `.queue-root`, `.clone-enabled`, `.landed`,
`.close-state`, `.workspace`, `blog-routing.yaml`

**Slot metadata** (slot/attic only):

`slot_io.parse_slot()` — repos, primary, branch, title, state, slot number.

**Blog/artifact drift** (areas with `blog-routing.yaml`):

**IMPORTANT:** `blog_routing.plan()` reads the working tree (`.glob()`,
`.read_bytes()`), which produces false positives when the checkout is on
a feature branch older than a promotion. The survey enumerates entries
from `git ls-tree` on base refs, not from disk. `blog_routing.plan()`
is used only for routing resolution (which destinations an entry maps
to), after the entry list is determined from refs.

```python
def entries_on_base(repo: Path, base: str, blog_dir: str = "docs/blog") -> set[str]:
    result = git(repo, "ls-tree", "-r", "--name-only", base, "--", blog_dir)
    return {line for line in result.stdout.strip().splitlines() if line}
```

Rules from #589 comment:

1. **Promotion gap** = entry present on workspace **base branch** (`git
   ls-tree -r <base> -- docs/blog/`) AND absent from project **base
   branch** (`git ls-tree -r <base> -- docs/blog/`). Never compare files
   on disk — always base refs.
2. **Waiting for landing** = entry only on a workspace working branch
   (present on HEAD but not on base). Reported separately, not a gap.
3. **Unpublished** = absent from every destination the routing plan names.
   Determined by `blog_routing.plan()` for routing resolution, but entry
   presence at destinations checked via `git ls-tree` on the destination
   repo's base branch.
4. **Idempotent** = consecutive runs with no changes between produce
   identical numbers. Tested.

**Closure status** (D8 — dual-path):

For each branch in the workspace repo:
- If `refs/soredium/closed/<branch>` exists: read the ref and
  `refs/soredium/closed-meta/<branch>` blob for structured status
  (landed SHA, promoted files, published destinations, timestamp).
- If no closure ref: check for a stamp commit (`chore: branch closed`)
  on the branch tip; parse the landing SHA from the message.
- Classify: stamped-and-landed, stamped-but-not-verified, unstamped,
  no-closure-record.

### Layer 3: Reporting

`survey(family_roots: list[Path]) -> list[SurveyResult]`

Orchestrates discovery and inspection for each family root. Rolls up
per-area findings into a summary attention list.

**Finding generation:**

| Condition | Severity | Category |
|-----------|----------|----------|
| Clone diverged from canonical | ERROR | `drift` |
| Rebase/merge/cherry-pick in progress | ERROR | `stuck-operation` |
| Queue names a branch not checked out | WARN | `queue-mismatch` |
| Multiple queues, no `.queue-root` | WARN | `queue-ambiguity` |
| Stale `.close-state` file | WARN | `stale-state` |
| Dirty working tree on base branch | WARN | `dirty-base` |
| Blog promotion gap (base ref comparison) | WARN | `promotion-gap` |
| Unstamped branch (closed but no stamp) | WARN | `unstamped-branch` |
| Unpublished entries (routing plan) | INFO | `unpublished` |
| Ahead of canonical (normal during work) | INFO | `ahead` |
| Legacy layout detected | INFO | `legacy-layout` |
| Landed but not archived slot | INFO | `landed-not-archived` |

Findings use `verification.Finding(severity, category, message)`.

## Data Model

```python
@dataclass
class Area:
    kind: str             # "canonical"|"slot"|"clone"|"attic"|"standalone-workspace"
    path: Path
    slot_number: int | None
    repo_name: str | None

@dataclass
class RepoState:
    name: str
    path: Path
    branch: str
    dirty_count: int
    remotes: list[str]
    ahead_of_canonical: int
    follow_state: str | None
    in_progress: str | None
    worktrees: list[tuple[str, str]]

@dataclass
class QueueInfo:
    path: Path
    items: int
    open_items: int
    active_issue: str | None
    status: str
    is_root: bool

@dataclass
class DriftEntry:
    file: str
    source_repo: str
    dest_repo: str
    status: str           # "gap"|"waiting"|"unpublished"|"stale"

@dataclass
class ClosureRecord:
    branch: str
    source: str           # "ref"|"stamp"|"none"
    landed_sha: str | None
    landed_on: str | None
    promoted: list[str]
    published: list[str]
    closed_at: str | None

@dataclass
class AreaReport:
    area: Area
    markers: list[str]
    repos: list[RepoState]
    queues: list[QueueInfo]
    closures: list[ClosureRecord]
    legacy_layout: bool
    findings: list[Finding]
    drift: list[DriftEntry]
    slot_info: SlotInfo | None

@dataclass
class SurveyResult:
    family_root: Path
    areas: list[AreaReport]
    attention: list[Finding]
    summary: dict[str, int]
```

## Output Formats

### Script mode (KEY=VALUE)

```
FAMILY=/path/to/family
AREA_COUNT=8
CANONICAL_COUNT=2
SLOT_COUNT=4
CLONE_COUNT=1
ATTIC_COUNT=1
ATTENTION_COUNT=3
AREA_1_KIND=canonical
AREA_1_PATH=/path/to/family/soredium
AREA_1_BRANCH=main
AREA_1_DIRTY=0
AREA_1_AHEAD=0
AREA_1_QUEUES=1
AREA_1_QUEUE_1_STATUS=active
AREA_1_QUEUE_1_ITEMS=81
AREA_1_QUEUE_1_OPEN=30
...
FINDING_1=error:drift:clone soredium diverged from canonical
FINDING_2=warning:queue-mismatch:slot 3 queue names issue-500 but branch is main
FINDING_3=info:landed-not-archived:slot 7 landed, ready for archive
```

### CLI/TUI mode (JSON Lines events)

New event in `commands/events.py`:

```python
@dataclass
class SurveyAreaInfo:
    kind: str
    path: str
    slot_number: int | None
    repos: list[dict]        # serialised RepoState
    queues: list[dict]        # serialised QueueInfo
    closures: list[dict]      # serialised ClosureRecord
    markers: list[str]
    findings: list[dict]      # serialised Finding
    drift: list[dict]         # serialised DriftEntry

@dataclass
class SurveyReady:
    family_root: str
    areas: list[SurveyAreaInfo]
    attention: list[dict]
    summary: dict[str, int]
```

New command module `commands/survey.py` with
`execute(family_root: str) -> SurveyReady`.

## Read-Only Guarantee

**Structural enforcement:**
- `discover_areas`: only `Path.exists()`, `Path.iterdir()`, `Path.is_dir()`
- `inspect_area`: only `git_util.git()` with read-only git subcommands
  (`status`, `rev-parse`, `for-each-ref`, `ls-tree`, `log`, `remote`,
  `branch`), `Path.read_text()`, and existing read-only functions
  (`canonical_sync.follow()`, `queue_home.candidates()`, etc.)
- No `open(path, 'w')`, no `Path.write_text()`, no mutating git commands
- No worktrace recording — the survey observes, it does not act

**Test enforcement:**
Following `test_ctx_reads_only.py` pattern:

```python
def tree(directory: Path) -> dict[str, bytes]:
    return {str(p.relative_to(directory)): p.read_bytes()
            for p in sorted(directory.rglob("*"))
            if p.is_file() and ".git" not in p.parts}

def test_survey_read_only(tmp_path):
    family = build_family(tmp_path, ...)
    before = {str(d): tree(d) for d in family.rglob("*") if d.is_dir()}
    result = survey([family])
    after = {str(d): tree(d) for d in family.rglob("*") if d.is_dir()}
    assert before == after
    for repo in all_repos(family):
        assert git(repo, "status", "--porcelain").stdout.strip() == ""
```

**Idempotency test:**
```python
def test_survey_idempotent(tmp_path):
    family = build_family(tmp_path, ...)
    result1 = survey([family])
    result2 = survey([family])
    assert result1 == result2
```

## File Layout

```
project/
  areas.py           # discover_areas, inspect_area, survey (new)
commands/
  survey.py           # execute() -> SurveyReady (new)
  events.py           # SurveyAreaInfo, SurveyReady (extend)
tests/
  test_areas.py       # discovery, inspection, read-only, idempotency (new)
```

## Dependencies

Imports from existing modules (all read-only usage):

- `project.git_util` — `git()`, `collect_slot_repos()`
- `project.canonical_sync` — `locate()`, `reference()`,
  `ahead_of_canonical()`, `_in_progress()`, `_worktrees()`
  (**NOT `follow()`** — it mutates; inlined as `follow_state_readonly`)
- `project.queue_home` — `candidates()`, `uses_root()`
- `project.queue_io` — `queue_dir()`, `read_queue()`
- `project.slot_io` — `parse_slot()`
- `project.clone_manager` — `is_permanent()`, `is_enabled()`
- `project.branch_base` — `parent_of()`, `base_of()`
- `publish-blog.blog_routing` — `plan()` (**routing resolution only**;
  entry enumeration via `git ls-tree`, not `plan()`'s glob)
- `verification` — `Finding` (severities: ERROR, WARN, INFO)

## Protocol Compliance

| Protocol | How complied |
|----------|--------------|
| a-read-never-writes | Structural: no write calls. Test: tree snapshot before/after |
| one-queue-per-area | Uses `queue_home.candidates()`, never builds queue paths |
| git-util-single-source | All git via `git_util.git()` |
| externalised-scripts-require-tests | `test_areas.py` ships with `areas.py` |
| the-script-that-acts-records-it | N/A — the survey does not act |

## Future: #618 Closure Refs

When #618 ships, `inspect_area` will read structured closure refs from
`refs/soredium/closed/` and `refs/soredium/closed-meta/` in workspace
repos. Until then, falls back to parsing stamp commit messages. The
`ClosureRecord.source` field distinguishes the two paths.

## References

- Hortora/soredium#589 — issue and drift rules comment
- Hortora/soredium#618 — structured closure refs (created during this brainstorm)
- `project/ctx.py` — KEY=VALUE output pattern
- `commands/events.py` — JSON Lines event pattern
- `tests/test_ctx_reads_only.py` — read-only test pattern
- `project/canonical_sync.py` — drift measurement functions
- `project/queue_home.py` — queue discovery
- `publish-blog/blog_routing.py` — blog routing plan
- `docs/protocols/a-read-never-writes.md` — read-only protocol
- `docs/protocols/one-queue-per-area-never-synced.md` — queue location protocol
- `docs/protocols/git-util-single-source.md` — git wrapper protocol
- `docs/protocols/externalised-scripts-require-tests.md` — test requirement
