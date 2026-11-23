## D1: Scope — explicit family roots only

**Choice:** Only survey family roots passed as arguments; no auto-discovery.
**Alternatives:**
- Auto-discover + explicit — scans ~/claude/*/ by default; couples tool to directory layout
- Single family only — no multi-family in one invocation; too restrictive
**Rationale:** The filesystem layout is a convention, not a contract. Explicit args keep the tool portable and predictable.
**Trade-offs:** User must know their family roots. Mitigated by discoverability in other tools.
**Sources:** Issue #589 description (`[family-root ...]` syntax)
**Exploration:** quick
**Status:** captured

## D2: Output format — dual internal + CLI

**Choice:** Python dataclasses internally; KEY=VALUE as script; JSON Lines events via commands/survey.py for TUI.
**Alternatives:**
- JSON Lines only — simpler but breaks KEY=VALUE convention for pipeline scripts
- KEY=VALUE only — no TUI integration; defers the work
**Rationale:** Matches the established pattern (ctx.py returns dict, prints KEY=VALUE; commands/status.py wraps into StatusReady events).
**Trade-offs:** Two output paths to maintain. Mitigated by the dataclass being the single source.
**Sources:** commands/events.py (RepoSlotInfo, StatusReady patterns), project/ctx.py
**Exploration:** quick
**Status:** captured

## D3: Blog drift detection — included in initial survey

**Choice:** Include blog/artifact drift detection from the start, following the rules in the issue comment.
**Alternatives:**
- Defer to follow-up — ship structural survey first, add drift later
- Stub only — wire routing but don't compare ls-tree yet
**Rationale:** The hand-survey found false positives precisely because it didn't follow the base-ref comparison rules. Getting drift right is the stated purpose.
**Trade-offs:** Adds complexity (routing resolution, cross-repo ls-tree). Justified by the false-positive history.
**Sources:** Issue #589 comment (drift rules 1-6), publish-blog/blog_routing.py (plan function)
**Exploration:** quick
**Status:** captured

## D4: Severity model — reuse verification.Finding

**Choice:** Use the existing verification.Finding NamedTuple (severity: error/warning/info, category, message).
**Alternatives:**
- Custom severity levels — survey-specific (action-needed, drift, stale); more descriptive but adds vocabulary
- No severity — flat list; simpler but harder to triage
**Rationale:** One severity vocabulary across the codebase. Survey attention items are findings.
**Trade-offs:** Finding's three-level severity may be coarser than ideal. Sufficient for triage.
**Sources:** verification/__init__.py, verification/repo_checks.py
**Exploration:** quick
**Status:** captured

## D5: Discovery mechanism — filesystem walk

**Choice:** Walk the directory tree to discover areas. Canonical repos = top-level dirs with .git (not under slots/clone/attic/). Slots = slots/N/. Clones = clone/repo/. Attic = slots/attic/N/.
**Alternatives:**
- DB-first with filesystem fallback — queries worklog DB, verifies on disk; misses unregistered areas
- Both and compare — most thorough but doubles work
**Rationale:** The filesystem IS truth. The DB can be stale (reconcile_slots.py exists for this reason). The survey should report what exists, not what the DB says.
**Trade-offs:** Slower than a DB query for large families. Acceptable for a survey tool.
**Sources:** project/topology.py (_detect_* functions), scripts/reconcile_slots.py
**Exploration:** quick
**Status:** captured

## D6: Architecture — three-layer (discover, inspect, report)

**Choice:** Three distinct layers: discover_areas() -> inspect_area() -> SurveyResult with attention findings.
**Alternatives:**
- Flat single-pass — one loop walks and inspects; simpler but tangled
- Strategy per type — per-type strategy classes; over-engineered for 3-4 types
**Rationale:** Each layer independently testable. Discovery is purely filesystem. Inspection reuses existing functions. Read-only property is structural.
**Trade-offs:** More code than flat single-pass. Justified by testability.
**Sources:** project/canonical_sync.py, project/queue_home.py, verification/ library
**Exploration:** quick
**Status:** captured

## D7: Area kinds — five types

**Choice:** canonical, slot, clone, attic, standalone-workspace.
**Alternatives:**
- Add agent sessions — broader scope, may be out of scope for lifecycle
- Fewer (merge attic into slot) — attic is just archived state; loses visibility into the distinction
**Rationale:** These five cover everything the lifecycle touches. Each has distinct markers and inspection concerns.
**Trade-offs:** standalone-workspace may be rare. Including it prevents silent omissions.
**Sources:** project/topology.py, project/clone_manager.py, project/slot_io.py
**Exploration:** quick
**Status:** captured
