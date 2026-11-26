# Session Handover

**Date:** 2026-11-26
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`).
**Branch:** `issue-618-structured-closure-refs` — done, landed

## What was done (this session)

**#606 — canonical_sync.locate rejects URL local remotes:**
- `locate()` now detects network URLs (`://`, `git@host:`) in the `local` remote and raises `local_is_url` with redacted credentials
- Relative paths resolved from repo directory, not Python CWD
- 4 new tests: HTTPS URL, SSH URL, credentials, relative path

**#610 — Untrack pipeline state files:**
- `.work-progress`, `.wrap-log.jsonl`, `.land-ledger.jsonl`, `.workspace` untracked from workspace repos via `git rm --cached`
- `ctx.py` no longer writes `.workspace` marker on read — reports `marker_stale` instead (protocol violation fixed)
- `_EXCLUDE_ENTRIES` in `slot_git.py` ensures new clones/slots get exclude entries
- `.workspace` added to `_LIFECYCLE_IGNORE` in close.py
- 7 new tests across 3 files

**#618 — Structured closure refs:**
- New module `project/closure_refs.py`: `record()`, `read()`, `list_closed()`, `clear()`
- `close.py` writes closure refs after each ceremony
- `refs/soredium/closed/<branch>` (landed SHA), `refs/soredium/closed-meta/<branch>` (structured blob)
- 11 new tests (9 unit + 2 integration)

## What is next

#623 — plan_manager populate and render scale from labels

## Notes for next session

- Engine and eidos forks still need a one-time manual force-push (from #627)
- Three done items landed this session: #606, #610, #618
