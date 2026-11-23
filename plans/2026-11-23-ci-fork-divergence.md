# CI Fork Divergence Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #627 — CI fork divergence — mirror fails silently on parallel commits

**Goal:** Make `mirror()` diagnose non-fast-forward failures accurately and fix the CI action that creates the divergence.

**Architecture:** `mirror()` gains a fetch-before-push step that classifies failures as `"diverged"` (fork has unique commits) vs `"failed"` (network/auth). The misleading "catches up on next publish" warning is replaced with an accurate diagnosis naming the fork's divergent commits. The CI action on casehub repos is gated to only commit on the org's repos, not forks.

**Tech Stack:** Python (canonical_sync.py), pytest, GitHub Actions YAML

## Global Constraints

- `mirror()` return type stays `str` — values expand to include `"diverged"`
- No force-push in normal operation — force-push is emergency behaviour only
- Mirror failure remains non-fatal — publish succeeds regardless
- Diagnostic detail goes in `PublishResult.warnings` (existing pattern)

---

## Batch 1: mirror() fetch-diagnose-report

### Task 1: Test and implement mirror divergence detection

**Files:**
- Modify: `project/canonical_sync.py:553-559` (mirror function)
- Modify: `project/canonical_sync.py:746-750` (_publish_base warning handling)
- Test: `tests/test_canonical_sync.py`

**Interfaces:**
- Consumes: `Canonical` dataclass, `git()` from `git_util`, `_sha()`, `_is_ancestor()`, `_out()`
- Produces: `mirror(canon: Canonical) -> str` — returns `canon.mirror`, `"none"`, `"diverged"`, or `"failed"`

- [ ] **Step 1: Write failing test for divergence detection**

Add to `tests/test_canonical_sync.py` in the `TestPublish` class (after the existing `test_mirror_failure_warns_and_the_publish_stands` at line 639):

```python
def test_mirror_diagnoses_divergence(self, fam):
    """When the fork has commits canonical doesn't, mirror returns 'diverged'."""
    cs.land(fam.slot, "feat")
    # Simulate CI creating a commit directly on the fork (bare repo).
    # Clone the mirror, commit, push back — bare repos have no working tree.
    ci_clone = fam.root / "ci-sim"
    subprocess.run(["git", "clone", "-q", str(fam.mirror), str(ci_clone)],
                   capture_output=True, check=True)
    commit(ci_clone, {"ci-docs.txt": "generated\n"},
           "docs: regenerate API docs [skip ci]")
    g(ci_clone, "push", "-q", "origin", "main")

    result = cs.publish_base(fam.canon)

    assert result.mirrored == "diverged"
    assert result.published == "upstream"  # publish still succeeds
    assert any("diverged" in w for w in result.warnings)
    assert any("regenerate" in w for w in result.warnings)  # commit subject appears
    assert not any("catches up" in w for w in result.warnings)  # old message gone
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_canonical_sync.py::TestPublish::test_mirror_diagnoses_divergence -v`
Expected: FAIL — `mirror()` returns `"failed"` not `"diverged"`

- [ ] **Step 3: Write failing test for network failure (not diverged)**

```python
def test_mirror_reports_network_failure_not_diverged(self, fam):
    """When the fork is unreachable, mirror returns 'failed', not 'diverged'."""
    cs.land(fam.slot, "feat")
    g(fam.canon, "remote", "set-url", "origin", str(fam.root / "gone.git"))

    result = cs.publish_base(fam.canon)

    assert result.mirrored == "failed"
    assert result.published == "upstream"
    assert not any("diverged" in w for w in result.warnings)
```

- [ ] **Step 4: Run test to verify it fails**

Run: `python3 -m pytest tests/test_canonical_sync.py::TestPublish::test_mirror_reports_network_failure_not_diverged -v`
Expected: FAIL — the existing `test_mirror_failure_warns_and_the_publish_stands` passes for this case but the new test checks the warning content differently

- [ ] **Step 5: Write failing test for normal fast-forward mirror**

```python
def test_mirror_succeeds_when_canonical_ahead(self, fam):
    """Normal case: canonical has work the fork doesn't, push is ff."""
    cs.land(fam.slot, "feat")

    result = cs.publish_base(fam.canon)

    assert result.mirrored == "origin"
    assert result.published == "upstream"
    assert not result.warnings  # no warnings at all
```

- [ ] **Step 6: Run test to verify it passes (baseline)**

Run: `python3 -m pytest tests/test_canonical_sync.py::TestPublish::test_mirror_succeeds_when_canonical_ahead -v`
Expected: This should PASS with the existing code — it's the happy path. If it doesn't, there's a pre-existing issue.

- [ ] **Step 7: Implement the new mirror()**

Replace `mirror()` in `project/canonical_sync.py` (lines 553-559) with:

```python
def mirror(canon: Canonical) -> str:
    """Push canonical's base branch to the mirror. Returns the remote, "none",
    "diverged" or "failed".

    Fetches the mirror's base branch first to diagnose non-fast-forward failures:
    "diverged" means the fork has commits canonical doesn't (invariant violation);
    "failed" means a network or auth error.
    """
    if not canon.mirror:
        return "none"
    mirror_ref = f"refs/remotes/{canon.mirror}/{canon.base}"
    fetched = git(canon.repo, "fetch", "-q", canon.mirror, canon.base,
                  timeout=_NETWORK_TIMEOUT)
    if fetched.returncode != 0:
        return "failed"
    local = _sha(canon.repo, f"refs/heads/{canon.base}")
    remote = _sha(canon.repo, mirror_ref)
    if remote and not _is_ancestor(canon.repo, remote, local):
        return "diverged"
    result = git(canon.repo, "push", "-q", canon.mirror,
                 f"refs/heads/{canon.base}:refs/heads/{canon.base}",
                 timeout=_NETWORK_TIMEOUT)
    return canon.mirror if result.returncode == 0 else "failed"
```

- [ ] **Step 8: Update the warning in _publish_base()**

Replace the mirror warning block in `_publish_base()` (lines 746-750) with:

```python
        result.mirrored = mirror(canon)
        if result.mirrored == "diverged":
            mirror_ref = f"refs/remotes/{canon.mirror}/{canon.base}"
            subjects = _out(canon.repo, "log", "--oneline", "--format=%s",
                            f"refs/heads/{canon.base}..{mirror_ref}", "-5")
            count = _out(canon.repo, "rev-list", "--count",
                         f"refs/heads/{canon.base}..{mirror_ref}")
            result.warnings.append(
                f"{canon.repo.name}: {canon.mirror}/{canon.base} has diverged — "
                f"{count} commit(s) on the fork not in canonical; "
                f"manual resolution needed"
                + (f"\n  " + "\n  ".join(subjects.splitlines()) if subjects else ""))
        elif result.mirrored == "failed":
            result.warnings.append(
                f"{canon.repo.name}: {canon.mirror}/{canon.base} could not be reached")
```

- [ ] **Step 9: Run all mirror-related tests**

Run: `python3 -m pytest tests/test_canonical_sync.py::TestPublish::test_mirror_diagnoses_divergence tests/test_canonical_sync.py::TestPublish::test_mirror_reports_network_failure_not_diverged tests/test_canonical_sync.py::TestPublish::test_mirror_succeeds_when_canonical_ahead tests/test_canonical_sync.py::TestPublish::test_mirror_failure_warns_and_the_publish_stands -v`
Expected: ALL PASS

- [ ] **Step 10: Run the full test_canonical_sync.py suite**

Run: `python3 -m pytest tests/test_canonical_sync.py -v`
Expected: ALL PASS — no existing tests broken by the change

- [ ] **Step 11: Commit**

```bash
git add project/canonical_sync.py tests/test_canonical_sync.py
git commit -m "fix(mirror): fetch and diagnose before push — detect fork divergence

mirror() now fetches the mirror remote's base branch before pushing.
When the fork has commits canonical doesn't (invariant violation),
it returns 'diverged' with a warning naming the specific commits
instead of the misleading 'catches up on the next publish'.

Network/auth failures still return 'failed' as before.

Refs Hortora/soredium#627

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: CI action gating

### Task 2: Gate the generate-api-docs workflow on repository owner

**Files:**
- Modify: `~/claude/casehub/engine/.github/workflows/generate-api-docs.yml`
- Modify: `~/claude/casehub/eidos/.github/workflows/generate-api-docs.yml`
- Modify: `~/claude/casehub/agency/.github/workflows/generate-api-docs.yml`

**Interfaces:**
- Consumes: nothing from Task 1
- Produces: CI workflows that only commit on casehubio/* repos

- [ ] **Step 1: Update engine workflow**

In `~/claude/casehub/engine/.github/workflows/generate-api-docs.yml`, add the owner check to the job:

```yaml
name: Generate API Docs
on:
  push:
    branches: [main]
    paths:
      - 'api/src/main/java/**'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  generate:
    if: github.repository_owner == 'casehubio'
    uses: casehubio/parent/.github/workflows/generate-api-docs.yml@main
    secrets: inherit
```

- [ ] **Step 2: Update eidos workflow**

In `~/claude/casehub/eidos/.github/workflows/generate-api-docs.yml`, same change:

```yaml
name: Generate API Docs
on:
  push:
    branches: [main]
    paths:
      - 'api/src/main/java/**'
  workflow_dispatch:
permissions:
  contents: write
jobs:
  generate:
    if: github.repository_owner == 'casehubio'
    uses: casehubio/parent/.github/workflows/generate-api-docs.yml@main
```

- [ ] **Step 3: Update agency workflow**

Read `~/claude/casehub/agency/.github/workflows/generate-api-docs.yml` and apply the same `if: github.repository_owner == 'casehubio'` guard on the job.

- [ ] **Step 4: Commit each repo separately**

```bash
git -C ~/claude/casehub/engine add .github/workflows/generate-api-docs.yml
git -C ~/claude/casehub/engine commit -m "fix(ci): gate API docs generation on repository owner

Prevents the action from committing to forks, which creates parallel
commits that diverge from origin and break the mirror push.

Refs Hortora/soredium#627

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

Repeat for eidos and agency with the same commit message.

- [ ] **Step 5: Verify no other workflows need the same fix**

Check all three repos for other workflows that commit directly to main:

```bash
grep -r "contents: write" ~/claude/casehub/engine/.github/workflows/
grep -r "contents: write" ~/claude/casehub/eidos/.github/workflows/
grep -r "contents: write" ~/claude/casehub/agency/.github/workflows/
```

If other workflows have `contents: write` and commit to main, they need the same `if: github.repository_owner == 'casehubio'` guard.

## References

- [2026-11-23-ci-fork-divergence-design.md] — design spec this plan implements
- [project/canonical_sync.py:553] — current mirror() implementation
- [project/canonical_sync.py:746-750] — misleading warning in _publish_base()
- [project/canonical_sync.py:331] — settle_base() as fetch-then-diagnose pattern
- [tests/test_canonical_sync.py:639] — existing mirror failure test
- [~/claude/casehub/engine/.github/workflows/generate-api-docs.yml] — CI workflow to fix
- [GitHub #627] — focal issue
