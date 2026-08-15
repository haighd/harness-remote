# Team Sprint: 2026-08-15

**Project**: harness-remote
**Status**: Reviewed
**Started**: 2026-08-15 17:46
**Completed**: —

Status vocabulary: `Reviewed` is written after Gate 1 approves the master plan. `Complete` is written on `sprint/2026-08-15` during Step 4.5 after all group PRs and reviewed summaries have merged.

---

## 1. Sprint Overview

### Configuration

- **Project Root**: /Users/danhaight/Projects/harness-remote
- **Main Branch**: main
- **Sprint Branch**: sprint/2026-08-15
- **Package Manager**: bun
- **Test Command**: node --test --experimental-strip-types web/src/*.test.mjs
- **Model Strategy**: balanced
- **Source Plan**: none
- **Source Plan Blob**: none
- **Antigravity Available**: true
- **opencode-go Available**: false
- **Codex Strategy**: worktree

Worktree/stack notes (corrected after Gate 1 round-1 code review):
- The repo IS npm-based and commits `web/package-lock.json` (tracked). It has no root `package.json` and no `test` script (tests are individual node:test files) — that gap is what agents-config#349 tracks.
- This harness blocks `npm`, so installs use `bun` (`cd web && bun install`), which writes an untracked `web/bun.lock`. Do NOT commit `web/bun.lock`; the repo's canonical lockfile is `web/package-lock.json`. The Team Leader updates `web/package-lock.json` at integration if `web/package.json` changes (e.g. a DOM-test dependency).
- Fresh group worktrees have no `node_modules`; each worktree MUST run `cd web && bun install` before baseline/test runs or the Test Command fails (agents-config#350).
- The pinned Test Command uses bare `node --experimental-strip-types`, which requires **node ≥ 23** (stable strip-types). This machine has BOTH `/opt/homebrew/bin/node` v26.7.0 (passes 9/0) and an nvm/`/usr/local` node v20 (rejects `--experimental-strip-types`); a login shell may resolve `node` to v20. **Group worktree setup MUST, before any gate run, verify `node --version` ≥ 23 and prepend the v26 path to `PATH` if the default resolves lower** (the pinned command is immutable, so the environment — not the command — must supply a compliant `node`). Verified green on v26.7.0: 9 pass / 0 fail.

### Goals

- Complete 1 issue across 1 group
- The group runs through full plan → review → implement → code review → codex review pipeline
- No merge conflicts or blocking dependencies at integration time

### Scope

**Included Issues**: 1
**Included Issue IDs**: #2
**No-Close Issue IDs**: #2 (cross-project / skill-complete mode; issue lives in haighd/harness-remote and is not auto-closed by this run)

- Priority High: 0
- Priority Medium: 1
- Priority Low: 0

**Excluded**:

- Blocked issues: none
- Documentation issues: none
- Research issues: none
- Out of sprint scope (other epic #302 sub-issues): #1 (deployment/infra), #3 (attention inbox, depends on #2), #4–#9

### Success Criteria

- [ ] Group completes assigned issue #2
- [ ] All commits pass code review + review-loop
- [ ] All tests pass (Test Command green in the group worktree after `bun install`)
- [ ] Group PR merged to sprint branch
- [ ] Integration PR created once and handed synchronously to PR-management (to main, for the human)
- [ ] No critical bugs introduced

---

## 2. Group Assignments
Group names are lowercase ASCII slugs; `plan-finalize`, `plan-reviewed`, and `summary-*` are reserved.

### Group: topology

**Issues**: #2
**Total Effort**: 2 points (effort:medium)
**Worktree**: `./harness-remote-worktrees/sprint-2026-08-15-topology`
**Branch**: `sprint-2026-08-15-topology`
**Team**: Group Lead + Planning + Sr. SWE + SWE + Code Review (tiers per balanced)
**Group Lead Agent ID**: [once spawned]
**PR**: [once created]

#### Reviewed Implementation Contract: #2 — Add multi-server project topology
**Acceptance Criteria**:
- An All Servers mode aggregates reachable saved profiles and groups sessions with **nested server → project grouping** and **deterministic ordering** (interleaved sessions from one project must not produce repeated project groups; adjacent-only coalescing is insufficient).
- Sessions are keyed by a **composite `{serverKey, sessionId}` identity**, not a bare session ID. `serverKey` is a canonical key independent of any stored profile ID (stored duplicate profile IDs are normalized or rejected). All selection, React keys, caches, and the Stop/Rename/Delete/approval action routing use the composite key so two servers sharing a session ID cannot cross-route.
- Every session exposes typed topology/ownership fields with explicit semantics: `server`, `harness`, `project`, `worktree`, and ownership modeled as **provenance (bridge-created vs externally-created)** distinct from **current live ownership**. Provenance persists across bridge restart and across a prompt/hand-off (it must NOT flip bridge-created sessions to "external" after restart or after a prompt). Missing `project`/`worktree` render as explicit `unknown`/null, never blank or guessed.
- Server/project/worktree identity remains visible before Stop, Rename, Delete, or later approval actions (covered by a rendering test).
- Aggregation uses **per-profile error isolation with a bounded per-request timeout and cancellation** (`Promise.allSettled` / equivalent, never a naive `Promise.all`): a rejecting or never-resolving server cannot block the roster or hide responsive servers' results. Multi-server polling uses **bounded concurrency**. The isolation/timeout/concurrency orchestration MUST live in the dependency-free `topology.ts` as a **dependency-injected aggregation runner** (accepts injected per-profile fetch + timer/clock functions), so `web/src/topology.test.mjs` can drive hung/rejecting/slow servers deterministically with fake injected fetchers; `web/src/api.ts` becomes a thin wiring layer that supplies the real fetch/timer to that runner (it holds no independently-tested branching logic, avoiding the `api.ts` ERR_MODULE_NOT_FOUND import problem).
- Sequential desktop/mobile hand-off semantics are documented; concurrent turns are never presented as serialized. **Verification paths for these two non-aggregation criteria:** (a) hand-off docs — verify/extend the existing `README.md` hand-off section (README.md:331-335 already documents this almost verbatim; reuse, do not duplicate); (b) non-serialization — a `topology.ts` unit assertion that two concurrently-active sessions in one project are both surfaced as running (never coalesced into one serialized entry), so the presentation logic is unit-checked rather than left to manual UI inspection.
- **Field-source split:** `server` identity is stamped at the aggregation layer (App.tsx/api.ts), which knows which `SavedServerProfile` a session came from — do NOT add a redundant `server` field to the bridge payload (the per-server bridge does not know its own external profile name). `harness`/backend identity comes from `bridge/src/server.js` config, NOT the ACP event (`AcpService` emits only `{type, sessionId}`).
- **Provenance vs live attachment (persistence contract):** the current `sessionView(...external)` signal is derived from the in-memory `#ownedSessions` set (declared at `bridge/src/acp-service.js:174`; external flag emitted/derived at `:25,34,223,261,290,299`; snapshot persistence is separate, at `:570-598`), which is **ephemeral** — it flips on hand-off and is empty after restart, so it CANNOT satisfy a restart-persistence requirement on its own. Model two distinct fields: (1) **persisted provenance** (`bridge-created` vs `external-created`) persisted to the bridge's snapshot store and immutable once set — `bridge-created` stamped at creation, `external-created` stamped on **first discovery via `#refreshSessions()`** (the bridge has no external-creation hook, so "at creation time" applies only to bridge-created sessions), with an explicit `unknown` state for legacy snapshots that predate the field (migration); and (2) **live bridge attachment** (is this bridge currently the owner), which may legitimately be ephemeral. The plan must NOT claim external *activity* the bridge cannot observe after restart. Restart-persistence tests target persisted provenance, not the ephemeral attachment flag.

**Likely Files/Tests**: `web/src/types.ts` (typed topology/ownership + provenance fields), `web/src/serverProfiles.ts` (canonical serverKey, duplicate-ID handling), a **new dependency-free `web/src/topology.ts` module** holding the pure aggregation/grouping/keying/ownership logic so `node:test` can import it directly (do NOT put testable logic behind `web/src/api.ts`, whose extensionless runtime imports fail under the node runner with `ERR_MODULE_NOT_FOUND`), `web/src/api.ts` + `bridge/src/acp-service.js` (typed provenance/topology events), `web/src/App.tsx` (roster grouping UI). New topology test file `web/src/topology.test.mjs` (discovered by the `web/src/*.test.mjs` glob) plus a bridge test where typed events are added.
- **Required deterministic tests** (against the dependency-injected `topology.ts` runner with fake injected fetchers/clock): nested server→project grouping with interleaved input and a fully specified ordering (server, then project, then session, with a defined unknown-bucket position and tie-break); never-resolving/rejecting server isolation (responsive servers still returned, hung one times out) + an **observed abort/cancellation** assertion (the timed-out request is actually aborted, e.g. its AbortSignal fires — not merely ignored) + a **peak-in-flight** assertion proving bounded concurrency (requests are not all launched simultaneously) + a total request-count assertion against a **concrete budget** that accounts for per-directory hydration and per-session latest-message fan-out (the exact ceiling is a Gate 2 deliverable, below); duplicate session IDs across two servers routing Stop/Rename/Delete via the composite key; **persisted** provenance surviving a simulated bridge restart for BOTH bridge-created and external-created (refresh-discovered) sessions (not the ephemeral attachment flag) + legacy-snapshot `unknown` migration; duplicate/normalized profile IDs. Identity-visible-before-destructive-actions is validated by a render seam (below).

- Group gates (additive to the pinned sprint Test Command; baseline AND final acceptance must both stay green): (1) the pinned `node --test --experimental-strip-types web/src/*.test.mjs`, (2) the web typecheck/build (`cd web && bun run build`, i.e. `tsc -b && vite build`) so the React/TS changes type-check, and (3) the bridge test suite (`cd bridge && node --test` — real suite `bridge/test/*.test.js`, `bridge/package.json` `"test": "node --test"`). The pinned Test Command remains the sprint's canonical gate; (2) and (3) are additive because the pinned command alone neither type-checks the UI nor exercises the bridge.

**Exclusions**: iOS deployment/Tailscale/Pages (#1); attention inbox / attention-state engine (#3); iOS Web Push (#4); OMP Agent Hub bridge + mobile controls (#5, #6); credential rendering/approval hardening (#7); roster density/accessibility polish (#8); scale/token-neutrality proof (#9). No terminal/PTY parsing. Do not infer ownership from display names. Do not commit `web/bun.lock` (the harness-forced bun artifact) on the group branch; `web/package-lock.json` is the canonical lockfile, owned by the Team Leader at integration (so it may legitimately change there if a test dependency is added — this prohibition applies to the group branch's bun artifact, not to that integration update).

**Sequencing**: #2 is the topology foundation that #3 (attention inbox) depends on. #1 is not required for local implementation. Land the typed topology/ownership fields + dependency-free topology module (with tests) before the roster grouping UI in App.tsx.

**Design decisions delegated to the group's Gate 2 plan (each a REQUIRED deliverable with the stated acceptance bar):** these are resolved by the group's Planning Agent and validated at Gate 2, not pre-decided in this master plan; the master plan commits that they WILL be specified and tested.
- **serverKey derivation**: exact algorithm (canonical host+port normalization, equivalent-spelling handling, behavior on profile edit/credential change) and its collision/dedup rule. Acceptance: a `topology.test.mjs` case proving two spellings of one endpoint map to one key and two distinct endpoints never merge.
- **Correct routing key per operation scope**: session-scoped operations (history, prompts, commands, models, questions, permissions, project-data reads, Stop/Rename/Delete) route by the full `{serverKey, sessionId}`; server-scoped operations that run without a selected session (capabilities, global SSE — `web/src/api.ts:223-235`, no `sessionId` exists) route by `serverKey` alone. Do NOT force a composite key where there is no session. Acceptance: a test proving a non-active-server *session-scoped* op cannot read/write the active server's config, and that server-scoped ops target the correct server by `serverKey`.
- **Normalized topology/ownership schema**: exact typed shape and authoritative source per field — `server`=aggregation; **`harness`/backend from `bridge/src/server.js` config** (`AcpService` emits only `{type, sessionId}` and has no backend identity, so harness cannot come from the ACP event alone); live attachment from the bridge; `project`/`worktree` from a trustworthy bridge source (since `sessionView` currently exposes only `cwd`); sessions that arrive via `listSessions()` without an event, and direct OpenCode sessions, get explicit `unknown`. Acceptance: type definitions in `types.ts` + a test asserting `unknown` for missing project/worktree and for event-less `listSessions()` sessions.
- **Concrete 100-session request budget**: a named maximum request count covering per-directory hydration and per-session latest-message fan-out under bounded concurrency. Acceptance: the request-count test asserts `<=` that budget.
- **Render seam for identity-before-destructive-actions**: an **executable, gate-runnable** assertion (a jsdom/DOM-capable `node:test`) proving App.tsx renders server/project/worktree in the Stop/Rename/Delete confirmation UI — a purely "documented browser check" is NOT acceptable because it is not gate-enforceable. Acceptance: that render assertion runs and is green as part of the group gate. If a DOM test harness dependency is required, the group adds it to `web/package.json` (permitted for this test dependency) and the Team Leader reconciles `web/package-lock.json` at integration; the group still does not commit `web/bun.lock`.

---

## 3. File Ownership Matrix

| File/Directory | Primary Group | Shared? | Notes |
|-|-|-|-|
| web/src/topology.ts | topology | No | NEW dependency-free module: keying/ownership/grouping + a dependency-injected aggregation runner (accepts injected fetch/timer), unit-testable by node:test under --experimental-strip-types |
| web/src/topology.test.mjs | topology | No | NEW node:test for the topology module (matched by the Test Command glob) |
| web/src/App.tsx | topology | No | Roster grouping / All Servers view + identity in destructive-action UI (consumes topology.ts) |
| web/src/serverProfiles.ts | topology | No | Canonical serverKey, duplicate-ID handling |
| web/src/types.ts | topology | No | Typed session/topology fields; persisted provenance vs live attachment |
| web/src/api.ts | topology | No | Thin wiring: supplies real fetch/timer to the topology.ts runner (no independently-tested branching) |
| web/src/i18n.ts | topology | No | Any new "All Servers"/grouping strings — or constrain to existing keys |
| web/src/styles.css | topology | No | Roster grouping styles — or constrain to existing components/styles |
| bridge/src/acp-service.js | topology | No | Typed provenance/topology events; persisted provenance in snapshot store |
| bridge/src/server.js | topology | No | Authoritative harness/backend identity + surfacing project/worktree beyond cwd |
| bridge/test/*.test.js | topology | No | Bridge typed-event tests (part of group gate) |
| web/package.json | SHARED | Yes | Team Leader handles at integration (avoid dependency drift) |
| web/package-lock.json | SHARED | Yes | Repo's canonical (npm) lockfile, tracked; Team Leader updates at integration if web/package.json changes |
| web/bun.lock | topology | No | Untracked artifact of the harness-forced `bun install`; do NOT commit (canonical lockfile is package-lock.json) |

---

## 4. Shared File Change Requests

Groups document shared file needs here. Team Leader applies during Phase 4 integration.

| Requesting Group | File | Change Type | Rationale | Diff Snippet | Status |
|-|-|-|-|-|-|
| | | add/modify/delete | | | pending/applied/rejected |

**Resolution Notes**: _(Team Leader fills during Phase 4)_

---

## 5. Integration Points

Single group — no inter-group coordination required.

### Shared Types/Interfaces

- `web/src/types.ts` session/ownership fields consumed later by #3 (attention inbox); keep additive and typed.

### API Contracts

- Aggregation API in `web/src/api.ts` + bridge typed events in `bridge/src/acp-service.js` must stay typed (no PTY/terminal parsing).

---

## 6. Review Loop Tracking

| Group | Plan Review Rounds | Code Review Rounds | Codex Review Rounds | Escalations |
|-|-|-|-|-|
| topology | - | - | - | - |

---

## 7. Progress Notes

Chronological log. Format: `[YYYY-MM-DD HH:MM] [Group/Leader] - Note`

**Latest entries on top:**

- [2026-08-15 18:52] [Leader] - Gate 1 round 3: codex APPROVE (0 blocking), architect-reviewer APPROVE (0 blocking). Master plan review complete — VERDICT: APPROVED after 3 rounds. Both round-2 contradictions verified resolved against source. Status → Reviewed.
- [2026-08-15 18:35] [Leader] - Gate 1 round 2: codex ITERATE (7 blocking, escalating into group-level design detail), architect-reviewer ITERATE (2 blocking, source-verified). Both converge on 2 real contradictions from the round-1 fixes: (A) provenance mapped to ephemeral #ownedSessions can't satisfy restart-persistence; (B) isolation/concurrency logic in api.ts (unimportable by node:test) made its mandated tests unwritable. Fixed both: (A) split persisted immutable provenance (with unknown/migration) from ephemeral live attachment; (B) moved orchestration into a dependency-injected runner in topology.ts (api.ts = thin wiring). Adopted topology.ts over .mjs (codex). Delegated codex's remaining design-detail demands (serverKey derivation, normalized schema, request budget, render seam, full-interaction-path composite key) to the group's Gate 2 as named deliverables with acceptance bars — the correct master-plan abstraction. Non-convergence note: codex escalated master-plan review into implementation-design granularity (5→7); logging as a friction finding.
- [2026-08-15 18:05] [Leader] - Gate 1 round 1: codex ITERATE (5 blocking), architect-reviewer APPROVE (0 blocking). Addressed all 5 blocking topics: composite server+session key + cross-server action routing; per-profile isolation/timeout (no naive Promise.all) + bounded concurrency; ownership provenance vs current + restart/hand-off persistence + unknown/null; additive web-build + bridge-test gates; dependency-free topology.ts test seam. Folded architect-reviewer's warning (named verification paths for docs/non-serialization ACs) and server-identity-at-aggregation note. Verified against source: bridge DOES have tests (bridge/test/*.test.js, bridge "test":"node --test") — architect-reviewer's "no bridge tests" claim was false.
- [2026-08-15 17:46] [Leader] - Sprint started. Phase 0 complete (branch pushed, bootstrap pinned, codex=worktree). Dogfood run of team-sprint against harness-remote; scope resolved interactively as issue #2.

---

## 8. Testing Status

| Phase | Unit Tests | Integration Tests | Result | Notes |
|-|-|-|-|-|
| **Baseline** (before merges) | 9 | - | pass | Installed clone; worktree needs `bun install` first |
| After topology merge | - | - | - | - |
| **Final** (all merges + shared files) | - | - | - | Must pass before integration PR |

**Group gate (baseline + final), all three must be green:** (1) pinned `node --test --experimental-strip-types web/src/*.test.mjs`; (2) web typecheck/build `cd web && bun run build`; (3) bridge test suite `cd bridge && node --test` (real: `bridge/test/*.test.js`). The pinned Test Command is the sprint's canonical gate; (2) and (3) are additive because it alone neither type-checks the UI nor exercises the bridge (Gate 1 round-1 finding).

---

## 9. Checkpoint & Resume References

| Checkpoint | Timestamp | File Path | Phase | Notes |
|-|-|-|-|-|
| CP-1 | 2026-08-15 17:46 | .git/team-sprint/bootstrap-2026-08-15.json | 0 | Bootstrap pinned |

---

## 10. Merge & Completion Status

| Group | PR Number | PR Status | Commits | Review Rounds | Completed | Notes |
|-|-|-|-|-|-|-|
| topology | - | - | 0 | - | - | - |

**Overall Status**: 0/1 groups complete

---

## 11. Decisions & Trade-offs

- Dogfood run: team-sprint invoked against the consuming repo `haighd/harness-remote` (installed runtime = agents-config), the intended cross-project mode, deliberately avoiding self-hardening of the skill's own repo.
- Target #2 chosen over #3 because #3 depends on #2 and #2 is dependency-free for local implementation.
- Fork intentionally NOT synced to upstream first (7 commits behind as of 2026-08-08); acceptable because group PRs target the sprint branch, not upstream, and a sync would import unrelated churn into a friction-focused dogfood.

---

## 12. Final Summary

Pending sprint completion.
