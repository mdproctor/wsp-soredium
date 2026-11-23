# Session Handover

**Date:** 2026-11-23
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`).
**Branch:** `issue-590-work-start-create-clone` — scaffolded, not started

## What was done

**#589 — Read-only survey command** (landed, closed)

Built `project/areas.py` — a read-only command that surveys every area in a family root. Three layers: discover (filesystem walk), inspect (repo state, queues, markers, closures, blog drift, work-end preview), report (14 finding categories).

Key design decision: three `canonical_sync` functions (`follow`, `ahead_of_canonical`, `reference`) secretly fetch and cannot be used by a read-only tool. Inlined read-only replacements using `rev-parse --verify` and `merge-base`.

Blog drift compares base branches via `git ls-tree`, never disk files — fixing the false-positive problem (32/33 hand-survey gaps were wrong because the project was on a feature branch).

37 tests including read-only guarantee (tree snapshot before/after), idempotency, and false-positive prevention.

Code review found and fixed 4 HIGHs: bare-project false gap, YAML list projects field, Path serialization, hardcoded base branch.

**#618** filed during brainstorm — structured closure refs in workspace git refs (queued).

**3 follow-up issues filed:** #620 (gate duplicates), #621 (workspace canonical comparison), #622 (format_kv detail).

## References

- Spec: `specs/issue-589-read-only-survey-command/2026-11-23-read-only-survey-design.md`
- Plan: `plans/2026-11-23-read-only-survey.md`
- Blog: `blog/2026-11-23-mdp01-survey-command.md`
