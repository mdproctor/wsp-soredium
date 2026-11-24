# Session Handover

**Date:** 2026-11-24
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`).
**Branch:** `issue-600-work-start-branch-exists` — done, ready to land

## What was done (this session)

**#600 — work start branch already exists failures:**
- `branch_base.cut()` now guards against `branch == base_branch` and `branch == "HEAD"` with structured `refused:` error
- Known branches (with `refs/soredium/base/<branch>`) are resumed via `git checkout` instead of failing — returns `resumed:<parent>`
- Unknown existing branches get structured `branch_exists:` error instead of raw git message
- `create_branches_typed()` rollback adjusted: pre-existing branches are not deleted on workspace failure
- `CreateResult.resumed` field tracks whether any primary repo branch was resumed (scoped to project + workspace, not slot siblings)
- `branch-name.md` handler updated to reflect new behavior
- 13 tests: `TestCutGuards` (5), CLI (4), `TestSlotResumePartial` (2), API (2)

**SVG mandate for diary/blog entries:**
- `diary.md` — "Include diagrams" changed to mandatory gate with diagram-type menu
- `wrap-diary.md` — diagram gate: entry must reference at least one `.svg` on disk before completion
- `diary-visual-elements.md` — SVG generation promoted to primary visual element with creation guide
- `mandatory-rules.md` — "consider" changed to "every diary entry includes at least one SVG"

**#636 filed:** `create_slot()` never sets up `.queue-root` — slot workspaces use inherited queue instead of root queue, causing rebase conflicts on `.queue` and `.work-progress`

## What is next

Land #600. Then #636 (slot queue-root gap) is queued and ready — the fix is `queue_home.adopt(slot_dir, apply=True)` added to `create_slot()`.

## Notes for next session

- The SVG mandate commit rides on the #600 branch but is a separate concern — may want to split it at squash time
- Engine and eidos forks still need a one-time manual force-push (from #627)
- `sync-local` was run to install the updated skills
