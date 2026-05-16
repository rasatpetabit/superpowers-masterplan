# WORKLOG

Append-only handoff notes for collaboration with other LLMs (Codex, future Claude sessions). Read at the start of substantive work; append a brief dated entry before ending substantive work. Diff shows _what_; this captures _why_.

Pre-v2.0.0 entries were pruned in the v2.0.0 release; institutional knowledge from those entries was migrated into `docs/internals.md` (deep-dive: architecture, dispatch model, failure modes, design history, recipes, anti-patterns). Per-version release detail lives in `CHANGELOG.md` (preserved verbatim).

---

## 2026-05-16 — codex-routing-fix shipped (v5.8.0, branch codex-routing-fix)

**Scope:** Closes the T8 wave-mode-review-via-Claude misfire and the broader Codex under-dispatch chain (F1→F6 in `docs/masterplan/codex-routing-fix/brainstorm.md`). Bundle delivered as Phase A (instrument) + Phase B (policy fix) + Phase C (failure-class hooks) per the approved plan at `/home/ras/.claude/plans/steady-sparking-nygaard.md`. 15 tasks across 5 waves: Wave 1 parallel Codex EXEC × 3 (T1/T2/T3), Wave 2 serial Codex on `parts/step-c.md` (T4→T5→T6→T7), Wave 3 inline-Claude multi-file audit (T8), Wave 4 batched Codex EXEC (T9-T12 to `parts/failure-classes.md`), Wave 5 inline-Claude finale (T13 internals docs, T14 version+CHANGELOG, T15 this entry). All green.

**Worktree isolation:** `.worktrees/codex-routing-fix` on branch `codex-routing-fix`. Necessary because the work modifies `parts/step-c.md` and `parts/step-b.md` while the orchestrator reads them to drive itself — self-modification hazard. PR → main pending user merge approval.

**Three locked decisions baked into the plan (vs my conservative recommendations):**
1. **B1 granularity:** N Codex reviews per wave (one per member, diff scoped to member's `**Files:**`) — user picked "maximum precision per task" over my "single wave-level review". Required new contract `codex.review_wave_member_v1`.
2. **B2 default:** Aggressive — `**Codex:** ok` default for ALL single-file edits including doc edits. User picked "maximum offload" over my "code-only" recommendation. Risk-bounded by per-task `**Codex:** no` escape hatch.
3. **F6 scope (B4):** Bundled into this work — inline-reads/dispatch-brief audit of `parts/step-c.md` + `parts/doctor.md` runs in the same plan. User picked "fix while we're in here" over my "defer to follow-up bundle".

**Why instrument first (Phase A before Phase B):** Per `feedback_failures_drive_instrumentation_not_fixes` memory — never design /masterplan fixes on the spot; framework auto-files issues, analysis drives prioritization. Phase A adds the detectors (doctor #43, `subagent_return_bytes` telemetry field, mandatory event compliance) so Phase B's policy changes have measurable signal AND so future regressions auto-file as failure classes (Phase C).

**Asymmetric review rule:** Codex gets ALL code-review work unless the code under review came from Codex itself (`dispatched_by == "codex"`). Codified at both serial 4b site (`parts/step-c.md:492`) and the new wave-member 4b block. The principle existed in `docs/internals.md:577` but wasn't enforced in the wave-dispatch path before this bundle.

**Doctor check #43 backfill:** Running #43 against existing bundles `concurrency-guards` and `p4-suppression-smoke` WARNs both — neither emits `review→` events because both predate this bundle. Backfill warnings are documented as expected; no remediation work added.

**Subagent return-bytes telemetry:** The `subagent_return_bytes` field on per-subagent JSONL records is the long-missing piece for measuring the user's original context-pollution concern. Detector for new `subagent_return_oversized` class (threshold 5120 bytes = v3.3.0 WORKLOG-regression threshold).

**Commit range:** `80b96d5..<T15-sha>` (T1 first to T15 last; see CHANGELOG v5.8.0 for per-task SHA map).

**Notes for future maintainers:**

- The new aggressive Codex annotation default in `parts/step-b.md` may surface new categories of Codex EXEC failures (e.g., doc edits where Codex's interpretation differs from intent). Watch the `subagent_return_oversized` class for the first few v5.8.0 runs.
- The wave-member Codex REVIEW dispatch path is new and untested at scale. Doctor #43 will warn on coverage gaps; the `wave_codex_review_skip` failure class auto-files.
- The 5 new dispatch-brief contracts in `commands/masterplan-contracts.md` raise the per-dispatch overhead slightly (registry lookup) but pay back by making lifecycle dispatch sites lintable via `bin/masterplan-self-host-audit.sh --brief-style`.

---

## 2026-05-16 — concurrency-guards implemented (Guards B + C, branch concurrency-guards)

**Scope:** Full brainstorm → plan → execute run against `docs/masterplan/concurrency-guards/spec.md`. Two waves, 10 tasks, all green. Feature worktree at `/home/ras/dev/sp-mp-wt-concurrency-guards`, branch `concurrency-guards` (two commits: `662b54f` Wave 1, `f98e514` Wave 2). Pending: PR → main.

**Wave 1 — Guard B (slug-uniqueness):** `bin/masterplan-state.sh check-slug-collision <slug>` subcommand enumerates all git worktrees via `git worktree list --porcelain`, scans for in-progress state.yml matches, emits JSON `{collisions, suggested_suffix}`. Stale-detection fix: checks `[ -d "$wt" ]` BEFORE reading state.yml (the exec-form originally had the check unreachable). Global suffix scoping per D3: `max(N)+1` across all worktrees. Integrated into Step B0 sub-step 1d (`parts/step-b.md`) and Step I3 cross-worktree slug pass (`parts/import.md`); explicitly excluded from B0a (`--from-spec`) per D6. Smoke: `bin/masterplan-guard-b-smoke.sh` — synthetic two-worktree git fixture, passes collision + stale-peer detection.

**Wave 2 — Guard C (flock) + doctor #42:** `with_bundle_lock()` helper uses fd-based `flock -w 5` form — `(flock -w 5 9; "$@") 9>"$lockfile"` — because bash functions can't be exec'd by the raw `flock cmd` form. Inlined in `hooks/masterplan-telemetry.sh` (can't source state.sh; sourcing risks set -u dispatch-path failures). Wraps two rename sites in state.sh and five bundle-mode JSONL append sites in telemetry hook; || true preserves bail-silent contract. macOS fallback: WARN once per process via `MASTERPLAN_FLOCK_WARNED` env. Doctor check #42: stale `.lock` mtime > 1h, WARN severity, report-only, added to all complexity tiers + Haiku brief. Smoke: 100-concurrent appends (100 valid JSONL lines), state.yml race (21 entries), stale-lock detection (7200s age).

**Why fd-based flock:** `flock -w 5 "$lockfile" bash_function` executes the function name as an external binary → Permission Denied. The fd form runs everything in a subshell where bash functions are visible. This is the standard bash flock-with-functions pattern.

**6 brainstorm decisions baked into spec (D1-D6):** silent cd (D1), orphan-peer 4th AUQ option (D2), global suffix (D3), blocking flock -w 5 not -n (D4), stale-lock doctor WARN (D5), Guard B on import not --from-spec (D6).

---

## 2026-05-16 — concurrency-guards spec drafted (no code change)

**Scope:** Created `docs/masterplan/concurrency-guards/spec.md` as input for a future `/masterplan brainstorm` run. Documents two unguarded surfaces and proposes Guards B + C; explicitly defers D (owner sentinel) and rejects A (worktree-scoped paths).

**Why:** Surfaced via "can plans/state step on each other across worktrees?" question. The framework is worktree-aware on *discovery* (Step A globs across worktrees, orchestrator cd's to picked plan's worktree) but has no mutex, no per-worktree slug namespace, and no peer-session detection on *write*. Two real collision surfaces: (1) same slug in two worktrees → divergent `state.yml` + irreconcilable `events.jsonl` EOF conflicts on merge (jsonl readers in `hooks/masterplan-telemetry.sh:195,249,552,773` will break on conflict markers); (2) Stop-hook ↔ foreground-orchestrator race on `events.jsonl` and `state.yml` within a single worktree — `bin/masterplan-state.sh` does atomic-single-write via mktemp+rename (lines 380, 439) but holds zero locks, no `flock` anywhere in `bin/` or `hooks/`.

**Why B+C, not D:** Guard B (slug-uniqueness check at creation + auto-suffix via AUQ) eliminates the *creation* of shared paths, which cascades to eliminating the events.jsonl merge problem entirely (no shared path = no shared file = no merge conflict). Zero schema change. Guard C (`flock -w 5` around writes, with `command -v flock` fallback for non-Linux) is independently worth it for the same-worktree hook race. Guard D (owner sentinel: re-add `worktree:` + add `owner:{host,pid,started_at,last_heartbeat}` to state.yml) would catch active-peer detection but costs a schema bump, bundle migration, doctor additions, and a force-take UI — deferred until B+C in production reveal incidents B+C cannot catch. Guard A (worktree-scoped paths) rejected outright: breaks every existing bundle and couples intentional slug to accidental worktree-of-origin.

**Spec format:** Mirrors existing `docs/masterplan/p4-suppression-smoke/spec.md` structure (Purpose, Scope in/out, Desired behavior with negative tests, Constraints citing CD-2/3/7/9, Open questions for brainstorm, Success criterion, How to run). Six open questions explicitly carved out for brainstorm rather than pre-decided in the spec (peer-resume cd silence, stale-peer-worktree handling, suffix scope global vs per-worktree, telemetry hook block vs non-block on lock, stale-lock doctor check, whether B applies to `--from-spec`/`import`).

**Explicitly NOT in scope:** Code change (no edits to `bin/masterplan-state.sh`, `hooks/masterplan-telemetry.sh`, `parts/step-b.md`, or schema). No `MEMORY.md` entries (spec is artifact; rule isn't a behavior change). No version bump.

---

## 2026-05-16 — Claude `/goal` interop (observability-only) — telemetry field + audit rollup + docs

**Scope:** Asymmetry resolution for Claude Code's native `/goal` (shipped v2.1.139, 2026-05-12) vs the existing Codex `codex_goal` bidirectional integration. Plan v2 at `~/.claude/plans/structured-floating-rain.md` (revised after Codex adversarial review caught two errors in v1 — "no programmatic surface" claim ignored the Stop-hook input contract, and a Step C runtime hint would fight intentional `loose`/`gated` per-task checkpoints at `parts/step-c.md:625-650`). Locked v1 scope shipped together: telemetry hook field, audit rollup, docs subsection + telemetry-signals row, README compatibility note.

**Why observability-only (not bidirectional):** Claude `/goal` has no agent-callable surface — no create/read/clear analogue to Codex's `create_goal`/`get_goal`/`update_goal` MCP tools. The only available signal is `stop_hook_active` from the Stop hook input JSON (per https://code.claude.com/docs/en/hooks), which is `true` when Stop fires inside an autonomous-continuation loop. Writing a `claude_goal` field to `state.yml` without reconciliation would violate CD-7 (canonical authority of run bundle); adding a Step C runtime hint suggesting `/goal` is redundant under `--autonomy=full` and harmful under `--autonomy=loose`/`gated` (would override user-chosen per-task gates). The honest integration is to capture the observability boolean and document why no further integration exists.

**Files changed:**
- `hooks/masterplan-telemetry.sh` — Section 0b captures Stop hook JSON from stdin (defensive on missing/malformed/empty input), extracts `.stop_hook_active`, emits `claude_stop_hook_active` in the per-turn JSONL record at the existing jq -nc assembly. Bail-silent contract preserved.
- `lib/masterplan_session_audit.py` — `TelemetryStats.claude_continuation_records` count of records with `claude_stop_hook_active is True`; JSON output exposes `claude_continuation_records` + computed `claude_continuation_share`; table output adds "Claude autonomous-continuation share" section when any plan has non-zero count. Existing 18 audit tests still pass.
- `docs/internals.md` — new §8.5 "Claude `/goal` interop (host-only, observability-only)" with the asymmetry table, the explicit non-additions (no `claude_goal` schema field, no Step C runtime hint, no transcript parsing) with rationale; §9 record schema + field semantics updated; redacted-audit description notes the new per-plan rollup; TOC entry.
- `docs/design/telemetry-signals.md` — `claude_stop_hook_active` row in canonical field reference, schema example updated.
- `README.md` — compatibility note under Flags: `/goal` ok with `--autonomy=full` as outer wrapper, avoid under `loose`/`gated`.

**Verification:** `bash -n hooks/masterplan-telemetry.sh` clean; standalone bash unit-test of stdin-parse logic returns true/false/false/false for true/false/malformed/empty inputs; `python3 -c 'import lib.masterplan_session_audit'` clean; `python3 -m pytest tests/test_masterplan_session_audit.py` → 18 passed; `python3 -m lib.masterplan_session_audit --hours=1 --repo-roots=/tmp/nonexistent` renders the new section header with "(none)" when there are no continuation records. Existing fixture at `tests/fixtures/session-audit/repos/active-with-telemetry/.../telemetry-test.jsonl` omits the new field; audit handles this gracefully (defaults to 0 in rollup).

**Explicitly NOT in scope (preserved from plan):** No `state.yml` schema change. No new doctor checks. No Step C runtime hint. No upstream feature request filed. No transcript-based goal-text extraction. No version bump in this commit — separate release decision.

---

## 2026-05-15 — v5.4.0 — Parallelism wave: 4 new safe parallel-dispatch sites in the orchestrator

**Scope:** Minor release. After v5.3.3 shipped, surveyed the orchestrator for additional safe parallel-dispatch opportunities to speed up future masterplan runs. Identified 5; user authorized 4 (#1-#4 implemented; #5 deferred). Each preserves the existing output contract:

- **#1 — Doctor repo-scoped checks #26 / #30 / #31 / #36 / #39 → single Haiku batch.** Was 5 inline serial reads at the orchestrator. Now bundled into one Haiku dispatched in the SAME Agent batch as the existing per-worktree Haikus. New `contract_id: "doctor.repo_scoped.schema_v1"`. Haiku loads deferred `CronList` via `ToolSearch` for #26.
- **#2 — Step B1 intent-anchor → 3-way Haiku fan-out.** Was one Haiku reading 7 source files serially. Now Haiku A (AGENTS.md+CLAUDE.md+WORKLOG.md), Haiku B (state.yml+events.jsonl+spec.md), Haiku C (rg --files repo sketch) dispatched in one Agent message. Each returns extracted facts + hints (not classified anchor); orchestrator merges per documented precedence rules (run-state wins on `mode`, project-docs wins on `repo_role`/`yocto_ownership`/scope, repo-sketch ground-truths) and persists.
- **#3 — Doctor parent re-verify → parallel Bash batch.** Was a serial grep loop over the sample set. Now one Bash invocation backgrounds all N greps with `&` + `wait` and emits line-delimited JSON. Latency is the longest single grep, not the sum.
- **#4 — Step C eligibility cache → sharded build with parallel-group affinity.** Was one Haiku building the entire cache JSON. Now shards by parallel-group (each group in one shard so rule-5 cohort visibility holds) when groups exist; else by index range (ceil(N/10), min 1, max 4) for plans ≥10 tasks. Plans <10 tasks AND no groups still use the single-Haiku path. New `shard_id` field on Haiku return; orchestrator merges by idx-sort + contiguity-validate + atomic write.

**Why:** User asked "look for more opportunities to SAFELY engage in parallel tasks with subagents to speed up future Masterplan work" after the v5.3.3 ship. Mapped 7 already-parallel sites (Step A worktree, Step A specs, Step B0, Step C wave, Doctor plan-scoped, Import I1, Import I3.4) and identified the 5 inline-serial sites that didn't yet parallelize despite having no shared-state hazard. Selection criteria for SAFE: read-only or canonical-writer-merge pattern, no implementer conflicts, deterministic merge, preserves existing output contract for back-compat. The fifth opportunity (deferred) involved an implementer-class dispatch and was correctly excluded by user.

**Design choices confirmed via AUQ before writing:**
- #1: single Haiku for all 5 checks (not 5 parallel; not inline Bash) — dispatch overhead vs in-Haiku serial wash is small for 5 cheap checks; atomic partial-failure handling
- #2: 3-way split on natural source-class seams (not 5-way per file; not 2-way) — clean merge precedence by source class
- #3: parallel Bash batch (not Haiku-with-N-paths) — these are deterministic file/grep ops; LLM in the loop adds latency without signal
- #4: shard by parallel-group when present; index-range fallback; preserve plan-order on merge — preserves rule-5 cohort visibility, falls back gracefully on plans without group annotations

**Verification:** Static grep for the new identifiers (`doctor.repo_scoped.schema_v1`, `shard_id`, `source_class: "project-docs"`, `parent re-verify` parallel-Bash language). Cross-manifest version drift check — 4 fields at 5.4.0 (3 manifest files; README current-release line). No runtime smoke (this release is orchestrator-prompt edits; verification happens on next real masterplan run).

**Why minor (5.3.x → 5.4.0):** New parallel dispatch sites are behavior-affecting (different telemetry shapes, new event types in doctor logs, new `shard_id` fields in eligibility caches built post-upgrade) but every existing input/output contract holds. CHANGELOG details the back-compat surface.

**Rollout:** Dual-surface refresh per the patched rollout macro (Claude Code: marketplace update + plugin update; Codex CLI: marketplace upgrade) on both ras@epyc2 and grojas@epyc1.

---

## 2026-05-15 — v5.3.3 — Plugins UI errors: frontmatter on contract registry + drop dead auq-guard.sh

**Scope:** Patch release. Two static issues surfaced by Claude Code's Plugins UI Errors tab after v5.3.2: (1) `commands/masterplan-contracts.md` had no frontmatter — the command loader treats every `commands/*.md` as a slash command and rejects ones without frontmatter; (2) `hooks/auq-guard.sh` was a 238-line dead file shipped since v2.17.0 but never registered in `hooks/hooks.json` (the active AUQ-guard moved to user-global `~/.claude/hooks/` long ago). Fixed both — added minimal frontmatter to contract registry (referenced by path in `parts/step-b.md`, `parts/doctor.md`, `docs/internals.md`; heading anchors and refs unaffected), deleted the dead hook.

**Why:** The post-v5.3.2 rollout investigation surfaced a parallel discovery — the "rollout everywhere" macro in `feedback_rollout_macro.md` only covered Claude Code; Codex CLI was silently skipped for three releases (v5.3.0/1/2 never landed in `~/.codex/plugins/cache/`). User reported "still 5.2.1 on this host" with a `/plugin` UI screenshot; truth-checked `installed_plugins.json` on both hosts (already 5.3.2 on disk), then ran `codex plugin marketplace upgrade` to refresh the Codex side. Memory file patched to include the Codex CLI upgrade verb (verb is `marketplace upgrade`, not `plugin update` — Codex CLI has no separate plugin update command). The v5.3.3 issues themselves were ambient — the Plugins UI errors had been latent since v4.0.0 (contracts.md) and v2.17.0 (auq-guard.sh); they only became visible when the user clicked the Errors tab during the rollout verification.

**Verification:** Static scan against the installed v5.3.2 cache (and the working tree post-fix): all manifest JSON parses, all `.sh` files `bash -n`-clean, all SKILL.md and command frontmatter present and valid, no remaining no-frontmatter files in `commands/`. Skipped runtime smoke (the change is purely static metadata — frontmatter addition and file deletion).

**Rollout:** Dual-surface macro this time. epyc2 Claude: 5.3.2 → 5.3.3 (sha a945ea7). epyc2 Codex: cache replaced 5.3.2 → 5.3.3. epyc1 Claude: 5.3.2 → 5.3.3. epyc1 Codex: skipped (not configured on grojas's account). All four pin targets verified against `installed_plugins.json` / Codex cache directory listing — disk truth, not UI display.

---

## 2026-05-15 — v5.3.1 — Doctor #41 bash bug: `|| echo 0` produced "0\n0", silently skipping sub-fires

**Scope:** Patch release immediately after v5.3.0. Fixes a pre-existing bug (introduced in v5.1.1 alongside sub-fires (a)/(b); inherited by v5.3.0 sub-fire (c)) in `parts/doctor.md` Check #41 bash. The idiom `grep -cE 'foo' "$events" 2>/dev/null || echo 0` produces a two-line string when `$events` is readable with zero matches — `grep -c` always prints `"0"` and exits 1, so `|| echo 0` fires and appends a second `"0"`. The downstream `[ "$var" -eq 0 ]` then errors with `bash: [: 0\n0: integer expected` and silently skips the if-branch. Net effect on sub-fire (a): only fired when `events.jsonl` was entirely unreadable, never on the intended "file exists with zero degraded events" case. Sub-fire (b) similarly broken. Sub-fire (c) (new in v5.3.0) inherited the pattern but was already guarded by `[ -r "$events" ]`, so it tripped the bug only when events existed with zero matches. Fix: drop the `|| echo 0` fallbacks, guard the per-bundle loop with `[ -r "$events" ] || continue`, use `${var:-0}` parameter expansion as a belt-and-suspenders default in integer tests.

**Why:** Found during the retroactive Doctor #41 lint sweep across 426 run bundles (`/home/ras/dev/*`) immediately after v5.3.0 rollout completed. The "integer expected" stderr noise tipped off the investigation. Confirms an important framework lesson: a check that fires WARN on most repos forever is doing zero work and creating false confidence — the only thing that was actually firing for years was the narrow "events.jsonl missing entirely" branch, which is a different signal than "silent override without evidence".

**Verification:** Re-ran the cross-repo sweep against 426 bundles post-fix. Zero "integer expected" stderr lines. Sub-fire (c) result: PASS across all repos (by design — legacy ping-mode false-positives left no `events.jsonl` audit trail; the new `degradation_self_doubt` event closes that gap going forward). Static checks: `grep -n '|| echo 0' parts/doctor.md` shows only the three legitimate `date -u -d` sites remain (date emits one line or nothing; `|| echo 0` works correctly there); the five `grep -c` sites are now `|| true`-free with explicit `${var:-0}` defaults.

**Rollout:** Same macro — commit → push → tag `v5.3.1` → marketplace + plugin update on ras@epyc2 and grojas@epyc1.

---

## 2026-05-15 — v5.3.0 — Step 0 scan-then-ping detection default + Doctor #41 ERROR escalation

**Scope:** Fixes recurring false-positive `⚠ Codex plugin not detected — codex_routing and codex_review degraded to off for this run` against installs where Codex is fully present and active. Three coordinated changes: (1) `parts/step-0.md` adds a 4th `detection_mode` value `scan-then-ping` and makes it the default; Stage A is a literal-substring scan of the system-reminder skills list for `codex:` (zero judgment surface, modeled on `codex_host_suppressed` precedent at line 94), Stage B falls back to the legacy 5-token ping only when Stage A returns zero matches; (2) Step 0 self-doubt event `degradation_self_doubt` written to `events.jsonl` whenever the degrade-loudly path is about to emit the warning but both auth + plugin-manifest on-disk probes say the install is healthy; (3) `parts/doctor.md` Check #41 gains sub-fire (c) at ERROR severity — fires when the self-doubt event is present OR when an older bundle has `codex degraded — plugin not detected` alongside healthy auth + plugin files on disk. `docs/config-schema.md` documents the new value, softens the "fragile" framing on `scan`. Bump 5.2.3 → 5.3.0 across 4 manifest fields + README. CHANGELOG entry includes migration note: explicit `detection_mode: ping` in `.masterplan.yaml` keeps ping-only semantics; only the unset default flips.

**Why:** User repro just now in `yanos-mgmt/.worktrees/pivot-landing-4b-yanos-wireguard` via `/loop /masterplan --autonomy=full`: warning emitted despite the session's own system-reminder skills list containing `codex:codex-rescue`, `codex:setup`, `codex:rescue`, `codex:gpt-5-4-prompting`, `codex:codex-cli-runtime`, `codex:codex-result-handling`, and despite `bin/masterplan-codex-usage.sh` (commit b140246) showing the same epyc2 host actively dispatching Codex in the same window. Root cause: default `detection_mode: ping` requires the orchestrator (an LLM) to dispatch a 5-token `codex:codex-rescue` Agent and judge the result; observed failure modes include (a) skipping the dispatch entirely and confabulating the conclusion, (b) misinterpreting a normal Agent response as an error, (c) inheriting stale "codex absent" belief across compaction. No proof-of-dispatch in the audit trail (events.jsonl entry is written *after* the visible-stdout warning, so the orchestrator can emit the warning without ever proving it ran the ping). The `scan` mode was already specified but documented as "fragile" — in practice the `codex:` prefix is structural (Anthropic plugin namespacing), so a literal substring scan is robust and deterministic. Pure scan-then-ping captures the deterministic upside while keeping the rare "plugin truly absent" detection via Stage B.

**Design notes:** Drafted with Plan agent pressure-test. Initially proposed an "evidence-required guardrail" (require `detection_evidence_*` field in `codex degraded` event) but dropped it on Plan-agent advice — a model that confabulates "ping returned error" will equally confabulate the evidence string. Real load-bearing leverage is post-hoc deterministic detection via Doctor, which is bash-checkable. Self-doubt event is Step 0's contribution to that detection (it's harder to confabulate "the auth check passed but I'm still degrading" inline because the LLM is explicitly checking before writing — and even if it confabulates, the bash auth check at Doctor time catches the lie). The `scan-then-ping` new-value approach (vs. renaming `ping`) preserves back-compat for users with explicit config; only the unset default flips.

**Verification:** Static — `grep -n "scan-then-ping\|degradation_self_doubt\|detection_source" parts/step-0.md docs/config-schema.md parts/doctor.md` returns identifiers in all expected sites. Cross-manifest version drift (Check #30 territory): all four `version` fields + README at 5.3.0. Live smoke is deferred to the first post-release `/masterplan` invocation against the repro repo — expectation: no degradation warning, `events.jsonl` records `codex_ping ok` with `detection_source=scan`. Negative smoke (forced `detection_mode: ping` with stubbed dispatch failure) is theoretically possible but not run in this turn — Doctor #41 sub-fire (c) ERROR escalation can be exercised retroactively against any historical bundle that has both `codex degraded` and healthy current auth.

**Rollout:** "And roll it out everywhere" macro — commit → push → tag `v5.3.0` → `claude plugin marketplace update` + `claude plugin update` on ras@epyc2 and grojas@epyc1.

---

## 2026-05-15 — v5.2.3 — Auto-retro backfill + Codex JWT cosmetic-expiry fix

**Scope:** Two coupled refinements bundled into one release. (1) Step 0
resume controller item 4 now invokes Step R inline as a backfill on any
`/masterplan` touch of a `status: complete` (or `pending_retro`) bundle
missing `retro.md` (schema_v3+, `retro_policy.waived/exempt != true`).
Catches Step C 6 bypasses: manual state edits, brainstorm-only completions,
prior retro failures. Adds `retro_policy.exempt: true` field for smoke
fixtures. `bin/masterplan-state.sh transition-guard` auto-heals
`status: retro_pending` (one outlier bundle typo) to canonical
`status: pending_retro` on read. (2) Step 3 of CC-2, doctor Check #39, and
Check #41's `auth_healthy` probe skip the token-expired/expires-within-24h
sub-fires under `auth_mode == "chatgpt"` + `tokens.refresh_token` +
`last_refresh` within 7 days — that shape is the normal steady state of
ChatGPT auto-refresh, not degradation. Sub-fire (c) `last_refresh > 30d`
still fires (a stale refresh_token IS a real signal). All three sites also
fixed to read tokens from `.tokens.<field>` (schema_v3+) with a top-level
fallback. Retires the `codex_health_check_jwt_only` watcher in
`lib/masterplan_session_audit.py` — purpose served; user-visible boot
banner is the regression detector now.

**Why:** (1) Auto-retro was already the documented default per
`parts/step-c.md:724` ("6b — Auto-retro by default") but had gap paths
that left bundles `complete` without `retro.md`. Spec wanted to match
docs. (2) Live `/masterplan` invocation on epyc2/yanos-mgmt today emitted
`↳ Codex: degraded (id_token expired 0d ago)` despite Codex being fully
healthy — false positive that the v5.2.1 watcher was added to surface,
now the proper fix lands. Bonus: the original bash read `.id_token`
top-level but the real schema nests under `.tokens.id_token`; the bash
was technically broken though the LLM-driven Read tool compensated.

**Verification:** Check #39 bash extracted and run against live
`~/.codex/auth.json`: `Check #39: PASS (auth_mode=chatgpt; JWT auto-refresh
healthy; last_refresh 0d ago)`. Check #41 `auth_healthy` probe sets
`auth_healthy=1` under the same gate. `python3 -m py_compile` clean.
`bash -n` clean on both bin/ scripts. Router byte ceiling: 11460 bytes.
Smoke-tested the retro_pending auto-heal shim against a temp copy of
`pivot-landing-2-yang-namespace-sweep`: file rewritten successfully.

**Rollout:** "And roll it out everywhere" macro executed — commit
719bfe8 (auto-retro) → d8201eb (Codex fix) → 5f0ca60 (release v5.2.3),
tagged `v5.2.3`, pushed to origin/main, `claude plugin marketplace
update rasatpetabit-superpowers-masterplan` + `claude plugin update
"superpowers-masterplan@rasatpetabit-superpowers-masterplan"` ran on
both ras@epyc2 (local) and grojas@epyc1 (ssh) — both went 5.2.0 → 5.2.3.
Memory `feedback_rollout_macro.md` updated to pin the confirmed two-call
download sequence and PATH note for remote ssh invocation.

---

## 2026-05-14 — v5.1.0 — Failure-instrumentation framework

**Scope:** Six-class anomaly taxonomy + Stop-hook detector + auto-filed GitHub
issues + over-time analyzer + flush queue + synthetic-transcript smoke
fixture + Doctor Check #38 + breadcrumb stream emitted from
`parts/step-{0,a,b,c}.md` and `parts/import.md`. New files:
`parts/failure-classes.md`, `bin/masterplan-failure-analyze.sh`,
`bin/masterplan-anomaly-flush.sh`, `bin/masterplan-anomaly-smoke.sh`,
`docs/failure-analysis/.gitkeep`. Modified: Section 9 of
`hooks/masterplan-telemetry.sh` (~280 lines added), Doctor Check #38 in
`parts/doctor.md`, `docs/internals.md` §9 subsection, all 3 plugin
manifests, CHANGELOG. Plugin version 5.0.1 → 5.1.0.

**Why:** User's verbatim demand: "absolutely nothing tested at all" — every
recent `/masterplan` failure became a guess-fix-regress cycle because there
was no structured way to capture what failed, why, or whether the next fix
actually held. The framework replaces that workflow: the running plugin is
the data source; the accumulated GitHub-issue stream is the analysis
substrate. No `/masterplan` bug fix (Issue #5 included) ships before the
framework — fixes are designed against issue bodies, not transcript paste.

**Fix shape:** Stop hook's Section 9 inspects per-turn breadcrumbs +
`state.yml` + last-events tail and classifies into six anomaly classes
(silent-stop-after-skill, unexpected-halt, state-mutation-dropped,
orphan-pending-gate, step-trace-gap, verification-failure-uncited). Each
detection writes a canonical JSONL record to
`<run-dir>/anomalies.jsonl`, then attempts `gh issue create` with a stable
SHA1 signature (`<class>|<step>|<verb>|<halt_mode>|<autonomy>|<skill|gate>`)
embedded as `[auto:<sig12>]` in the title. Branches: no-match→create,
open-match→comment, closed-match→reopen (this is the regression detector —
fixes that don't actually fix get their tombstones reopened automatically).
gh-failure path writes to a sidecar `anomalies-pending-upload.jsonl`
drained by `bin/masterplan-anomaly-flush.sh`.
`bin/masterplan-failure-analyze.sh` queries the issue stream and emits
frequency / TTC / recurrence / per-verb / per-step / co-occurrence tables
to stdout + dated snapshot under `docs/failure-analysis/`. Defaults to repo
`rasatpetabit/superpowers-masterplan`; configurable via
`.masterplan.yaml#failure_reporting`.

**Verified:** `bin/masterplan-anomaly-smoke.sh` — 11/11 assertions pass:
six anomaly classes detect against synthetic state.yml + breadcrumb
fixtures, signature dedup produces one `gh issue comment` (not a second
create), closed-issue regression produces one `gh issue reopen`, dry-run
mode writes local JSONL with zero gh calls. Smoke runs in an isolated
`HOME=$tmp/fake-home` + mock `gh` (logs all calls, returns canned JSON).
`bash -n` clean on all four new scripts + hook. Doctor Check #38 fires on
a synthetic `anomalies.jsonl` (Warning per record) and stays silent when
the file is empty. Cross-manifest version consistency confirmed —
`.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`,
`.codex-plugin/plugin.json` all at 5.1.0.

**Known followups:** (1) Redaction layer for `state_yml_at_failure` /
`events_tail` if a future user needs to file against a public repo with
sensitive paths — currently the default destination is the plugin's
private repo so this isn't gating. (2) Codex-host parity: the Stop-hook
detector currently fires only under the Claude Code host. Codex Stop-hook
equivalent is a separate piece of work. (3) The first batch of real-world
anomaly records will drive Issue #5's actual fix design — that's the
intended workflow, not a backlog item.

---

## 2026-05-12 — v3.2.9 — Fix `/masterplan import` dedup false-positive

**Scope:** Bugfix in `bin/masterplan-state.sh` plus precision edits in
`commands/masterplan.md` Step I. Bumped 3.2.8 → 3.2.9.

**Why:** `/masterplan import` dry-run in a real user repo reported
`would-migrate: 1` for a record that was already migrated. Two stacked dedup
bugs in `bin/masterplan-state.sh`: (1) `canonical_slug()` was applied to
directory names but not to the explicit `slug:` frontmatter field, so a
date-prefixed legacy `slug:` would miss the canonical bundle; (2) existing
bundles' `legacy:` pointers — populated on every migrate — were never read
back. Pretty much guarantees duplicate completed work if the user ever
accepted the dry-run.

**Fix shape:** Two parallel indices in `existing_new_runs()`'s callsite:
`by_canonical` (canonical slug of every bundle dir) and `by_legacy_path`
(every `legacy.{status,plan,spec,retro}` value parsed back from existing
state.yml files). `find_existing_match()` checks both; skip-reason strings
preserve which check fired (`canonical slug match` vs `legacy: pointer
reference`) for debuggability. Both fixes ship together — slug-strip handles
the common date-prefix case; legacy-pointer scan is authoritative for
user-renamed bundles. Step I1.4 prose now spells out the two-part predicate
explicitly so future edits don't drift back to string-equal dedup. Step I3
pre-flight notes that "pre-existing collision" includes legacy-pointer match
too (defense-in-depth for `--file=`/`--branch=` direct routing that skips
discovery).

**Verified:** Synthetic fixture in a tmp git repo exercised all three paths
— canonical match (Bug 1 scenario), legacy: pointer match (Bug 2: user
renamed bundle), and the happy path (truly-new record still flagged). `bash
-n` clean. Inventory on the real repo still returns the same shape.

---

## 2026-05-11 — Unreleased — Scrub `bin/masterplan-state.sh` from user-facing surfaces

**Scope:** Remove all end-user-facing recommendations to run
`bin/masterplan-state.sh inventory` / `migrate`.

**Why:** The plugin runs in *other* projects, so `bin/masterplan-state.sh` is
never present in the user's CWD — it lives in the plugin install dir
(`~/.claude/plugins/.../bin/...`). Telling users (or the orchestrator running
in the user's CWD) to invoke it as a relative path always 404s. `/masterplan
import` already covers the migration end-to-end; doctor/clean/status/next
already cover the inventory side. The script stays as plugin-internal dev
tooling (`bin/masterplan-self-host-audit.sh` and dogfood docs still reference
it).

**Changes:** 13 edits across 5 user-facing files —
`skills/masterplan-detect/SKILL.md` (frontmatter + rendered suggestion),
`skills/masterplan/SKILL.md` (summary-first phrasing + dropped "if present,
prefer" block), `commands/masterplan.md` (Step 0 host loading, legacy
migration text, Step D discovery, doctor `--fix` row, clean `legacy`
category, state.yml schema comment), `README.md` (prose paragraph +
user-runnable command block), and `docs/masterplan/README.md` (migration
instruction). Doctor `--fix` action now reads "invoke `/masterplan import`
and select `<slug>` from the picker" rather than introducing a `--slug=`
short-circuit to Step I (decision: keep Step I unchanged).

**Kept references (Tier 2, repo-internal):** `CLAUDE.md`, `CHANGELOG.md`,
`docs/internals.md`, `bin/masterplan-state.sh`, `bin/masterplan-self-host-audit.sh`.

---

## 2026-05-10 — Unreleased — Codex global config bootstrap

**Scope:** Fix Codex-hosted `/masterplan` behavior where user-global
`~/.masterplan.yaml` defaults could be missed by the Codex entrypoint.

**Why:** `commands/masterplan.md` already specifies Step 0 config loading, but
the Codex-visible `skills/masterplan/SKILL.md` had its own host-adaptation and
run-bundle preamble without an explicit config bootstrap. Codex could derive
state defaults from built-ins before honoring the canonical prompt.

**Changes:** Added a Codex entrypoint config-bootstrap section that reads
`~/.masterplan.yaml`, then repo-local `.masterplan.yaml`, then shallow-merges
invocation flags. Clarified that Codex host suppression only disables recursive
`codex:codex-rescue` routing/review for the current invocation; other global
defaults such as `autonomy`, `complexity`, `runs_path`, and `parallelism` still
apply. Added a `bin/masterplan-self-host-audit.sh --codex` regression check and
updated README, internals, and CHANGELOG.

---

## 2026-05-09 — Unreleased — `next` follow-up / background / completion hardening

**Scope:** Close the remaining "processing loop stopped and user had to type `next`"
cases found in the last-24-hour Claude transcript scan.

**Why:** The initial Step N fix stopped `next` from becoming a fresh topic, but transcript
evidence also showed three non-cascade stop causes: completed plans that still had real
`next_action` follow-ups, background Codex/Agent work with no wakeup/poll contract, and
completion finalizers that marked state complete while task-scope work was still dirty.

**Changes:** Extended Step N with a `follow_up_pending` category and routing for completed
plans with concrete `next_action`; added optional `background:` run-state metadata plus
Step C resume polling before redispatch; hardened Codex background returns so they must
write state or ask a structured poll/schedule/pause question; and moved `status: complete`
behind a live `git status --porcelain` task-scope dirty gate. README, internals, and
CHANGELOG now document the contracts.

---

## 2026-05-09 — Unreleased — `next` verb / Step N

**Scope:** Fix cascade bug where typing "next" after a completed phase launched a
`/masterplan full next` brainstorm cycle (bare-topic catch-all), bloated context,
triggered auto-compaction, wrote `last-prompt: next` metadata, and replayed itself.

**Why:** Transcript analysis of last 24 hours showed 4 `last-prompt: next` cascade entries
in os-mgmt sessions and a similar post-completion gap in meta-petabit sessions. Root cause:
"next" was not in the reserved-verb set, so it hit the catch-all → Step B.

**Changes:** Added `next` to the routing table (→ Step N), arg-parse match set, reserved-verbs
warning, README command table, internals routing table, and frontmatter `description:` (6
sync'd locations per anti-pattern #4). Implemented Step N between Step M and Step A:
inline state scan → AUQ with resume/new-plan/status options. No subagent dispatch needed
(file count ≤ 20).

---

## 2026-05-08 — Unreleased — Codex-native plugin packaging

**Scope:** Make `rasatpetabit/superpowers-masterplan` installable as a Codex
plugin without moving the existing Claude command surface.

**Changes:** Added `.codex-plugin/plugin.json`, `.agents/plugins/marketplace.json`,
and the `plugins/superpowers-masterplan -> ..` Git symlink required by Codex's
marketplace plugin-path convention. Documented the portable Codex invocation as
`/superpowers-masterplan:masterplan`, and added a runtime compatibility block to
`commands/masterplan.md` so Claude tool names map cleanly onto Codex tool
contracts. README, internals, release docs, and self-host audit now cover both
plugin surfaces.

---

## 2026-05-08 — v3.0.0 completion finalizer + release

**Scope:** Made retro generation and old-state cleanup the default successful
completion behavior for `/masterplan`.

**Changes:** Step C now marks the run complete, invokes Step R internally for
all complexity levels, archives the run state in `state.yml`, and runs a
completion-safe Step CL subset (`legacy` + `orphans`, archive-only, no prompts,
no deletes). Added `--no-retro`, `--no-cleanup`, `completion.auto_retro`, and
`completion.cleanup_old_state` opt-outs. Extended `bin/masterplan-state.sh` to
catch standalone legacy specs/retros, not only plan-backed records. Updated
README, internals, `docs/masterplan/README.md`, release docs, manifests, and
CHANGELOG for v3.0.0.

**Cleanup dogfood:** Six legacy records now exist as run bundles under
`docs/masterplan/`. Verified originals were archived under
`legacy/.archive/2026-05-08/`; only legacy README placeholders remain under
`docs/superpowers/`.

---

## 2026-05-08 — v3 run-bundle architecture + legacy migration

**Scope:** Moved the architecture toward `docs/masterplan/<slug>/state.yml`
run bundles to prevent pre-status compaction loss, orphaned plan/spec/retro
sets, and non-persisted steering gates.

**Changes:** Added `bin/masterplan-state.sh` with `inventory` and copy-only
`migrate --write`; updated `commands/masterplan.md` to create `state.yml`
before brainstorming, persist `pending_gate` before every AUQ, prefer bundled
`spec.md`/`plan.md`/`retro.md`, and make `clean` handle whole run bundles plus
legacy cleanup. Updated routing stats, telemetry hook, detect skill, README,
CLAUDE, and internals for the new layout.

**Migration:** Dogfooded the migration on this repo. Four previous-version
archived plans were copied into `docs/masterplan/<slug>/` with `state.yml`,
`events.jsonl`, and available plan/spec/retro artifacts. Original
`docs/superpowers/...` files were intentionally preserved; clean owns any later
archive/delete.

---

## 2026-05-07 (PM) — AUQ-enforcement multi-track + --resume worktree fix

**Scope:** User pasted evidence of 8 distinct AskUserQuestion violations across
6 projects on 2026-05-07. Diagnosis dispatched two parallel Sonnet Explore
agents (transcript scan + config audit). Three root causes identified:
(1) `superpowers/5.1.0/skills/brainstorming/SKILL.md` lines 76, 127–131, 152
prescribe literal prose user-review templates including the verbatim string
`"Want to try it? (Requires opening a local URL)"` that produced incident #8;
(2) `~/.claude/CLAUDE.md`'s fluffmods-generated "Direct Questions" stanza
(lines 52–64) describes the rule without naming the AskUserQuestion tool, so
prose markdown lists satisfy the letter-of-the-rule; (3) zero hook
enforcement — the existing Stop hook is telemetry-only.

**Plan:** `/home/ras/.claude/plans/sprightly-scribbling-wozniak.md` — four
remediation tracks (A prompt strengthening / B warn-only Stop hook / C local
override skill / D in-house regressions) + a folded-in `--resume=<path>`
worktree fix from a separate user request paste.

**Why each track:**

- **Track A** — added a "Skill counter-pressure" subsection + end-of-turn
  self-check to `~/.claude/CLAUDE.md` Interaction Style. Names AUQ tool
  explicitly, lists the observed failure shapes verbatim ("you'll want to
  push and tag", "let me know if anything needs changing", "Want to try
  it?"), and forbids markdown-list-as-substitute-for-AUQ. Persistent
  surface, loaded every session globally.
- **Track C** — created `~/.claude/skills/auq-override/SKILL.md`. Confirmed
  it loads (appears in skill list system-reminder). Description targets
  every-conversation activation. Body explicitly rebuts the brainstorming
  SKILL.md prose templates and lists 8+ failure shapes with their AUQ
  rewrites. Belt-and-suspenders with Track A.
- **Track D + --resume worktree fix** — added v2.17.0+ block to
  `commands/masterplan.md` Step 0 (after the argument-parse precedence
  rules). When `--resume=<rel-path>` doesn't exist at cwd, search
  `<cwd>/.worktrees/*/<path>` and `<repo-root>/.worktrees/*/<path>`. Single
  match → auto-cd to that worktree before Step 0's repo-local config reload
  + Step C entry. Zero or multiple matches → AskUserQuestion with concrete
  options. Doctor `--fix` AUQ gate (lines 1689–1701) already shipped in
  v2.15.0; verified intact, no changes needed.
- **Track B** — `hooks/auq-guard.sh`, registered as a Stop hook in
  `~/.claude/settings.json` between the existing telemetry hook and the
  statusline updater. Reads `$CLAUDE_TRANSCRIPT_PATH` (or stdin payload),
  finds last "real" user message boundary (skipping tool_result-only user
  events), checks all subsequent assistant content for an AskUserQuestion
  tool_use; if absent, scans the last text block for trailing `?` or any of
  ~25 prose-question/implicit-offer patterns. Warn-only (stderr). Test
  fixtures: incidents #1, #3, #8 all fire (6/6 pass: 3 violations + 3
  silent cases including AUQ option labels with `?`).

**Decisions worth preserving:**
- Skipped touching the fluffmods-managed block (lines 40–88 of
  `~/.claude/CLAUDE.md`). Reinforced *outside* the managed territory in the
  hand-written Interaction Style section. If we ever want fluffmods to
  generate stronger language, that's a fluffmods PR, not a CLAUDE.md edit.
- Did NOT fork `obra/superpowers/skills/brainstorming` upstream; opted for
  the local override skill + the persistent CLAUDE.md clause + the hook.
  Three layers of pressure means at least one survives any session
  compaction.
- Hook is intentionally warn-only (user picked this over block-and-retry).
  Block-and-retry is the obvious next step if warn-only doesn't change
  behavior; revisit after a 24h follow-up scan.

**Verification path forward:** re-run the May-7 transcript-scan methodology
over the next 24h. Target: zero violations the hook didn't catch. Any caught
violations are evidence the prompt layer alone wasn't enough — that's the
data-collection mechanism for deciding whether to escalate to block-and-retry.

---

## 2026-05-07 - v2.16.0 - May 7 failure resolution

**Scope:** Audited 16 Claude Code transcripts in `~/dev` from 2026-05-07 (~36 MB
total) for `/masterplan` failures. Two parallel Sonnet survey agents + a
deep-read of `commands/masterplan.md` triangulated four distinct root causes
that survived v2.10.x–v2.15.x. Three are orchestrator bugs with prompt-level
fixes; one is a Claude Code harness bug we mitigate with a sentinel + docs.

**Why each fix:**

- **Bug A — per-task CD-9 hole (Step C step 4→5).** Reproducer: petabit-www T10→T11
  free-text "Want me to continue?" exit. Step 5's "skip scheduling silently"
  branch under non-`/loop` runs gave the orchestrator implicit license to stop
  improvising free-text gates. New **Step C step 4e — Post-task router** routes
  deterministically by autonomy + `ScheduleWakeup` availability. Under `gated`
  / `loose` without `/loop`, fires structured `AskUserQuestion(Continue / Pause
  here / Schedule wakeup)`. Under `/loop` step 5 runs (existing behavior).
  Under `--autonomy=full`, silent re-enter step 2. Wave-end fires gate once,
  not N times.

- **Bug B — `/masterplan execute <topic>` discarded.** Reproducer:
  petabit-os-mgmt 00:53 — explicit `execute` verb silently routed to
  brainstorm. Routing table only matched `execute <status-path>`; non-path
  args fell into Step A which discarded the verb. Added `execute
  <topic-or-fuzzy-slug>` row, stash `requested_verb` in argument-parse,
  added Step A step 7 verb-explicit override that surfaces 4-option
  `AskUserQuestion` instead of silent brainstorm.

- **Bug C — compaction summary ignored.** Reproducer: petabit-os-mgmt
  00:46→00:54 (compaction summary said "interrupted before Step B1"; orchestrator
  re-derived from filesystem). Added Step 0 compaction-recent notice (3 detection
  signals; non-blocking single line). Pairs with Bug B's verb-explicit override.

- **Bug D — dead session after `/reload-plugins`.** Reproducer: optoe-ng
  23:14→23:19 — sequence `/compact` → `/plugin` → `/reload-plugins` →
  `/masterplan` produced zero assistant response. Likely harness-level command
  de-registration. Added Step 0 invocation sentinel as first line of every turn
  so absence proves harness ate the invocation. CHANGELOG documents workaround;
  README has a Troubleshooting section.

- **Audit catches new gate phrasings.** `bin/masterplan-self-host-audit.sh`
  `check_cd9` regex extended with: `Want me to (continue|proceed|advance|run|execute)`,
  `Should I (continue|proceed|advance)`, `Shall I (continue|proceed)`, `Let me
  know (when|if|how)`, `(when|after) you're ready, (let me|I'll)`, `Continue to
  T<N>?`. Catches future regressions of Bug A at audit time before commit.

**Per-task gate contract decided this session.** Under `gated` / `loose`
autonomy without `/loop`, every task boundary is a structured
`AskUserQuestion`. Free-text "Want me to continue?" forbidden by CD-9. Under
`--autonomy=full` no gate fires. Under `/loop` step 5 takes precedence. The
v2.16.0 contract is autonomy-aware by design.

**No regressions.** v2.14.0/2.14.1's `git for-each-ref` import discovery,
v2.14.0's `doctor --fix` for #20/#21/#1a, v2.15.0's doctor end-gate, v2.15.0's
noargs precedence — all preserved. CD-9 audit reports clean.

**Not done this session.** Filing the upstream Claude Code issue for
`/reload-plugins` de-registering slash commands (placeholder URL in CHANGELOG).
The harness fix is not in scope for this plugin.

---

## 2026-05-04 - Unreleased - codex prevention layer (P1-P5) + /masterplan stats verb

**Scope:** Postmortem from a previous Fixes-1-5 session pinned the optoe-ng project-review
silent-codex-bypass to a specific transcript (`~/.claude/projects/-home-ras-dev-optoe-ng/7db6706e-…jsonl`,
2026-05-04T14:09–18:00). That session dispatched 7 agents (5 general-purpose wave + 1 kernel-c-reviewer +
1 abi-reviewer) and ZERO `codex:codex-rescue`. ZERO Step 0 codex-degradation warning text emitted. ZERO
eligibility-cache file writes. Easy-wins T7-T13 ran inline in orchestrator's own context with no Agent
dispatches at all. The Fixes 1-5 session only addressed the post-completion tag layer; this session adds
the prevention layer that catches the failure regardless of whether Step 0 ran.

**Why these specific changes:**

- **P1 (Step C step 1 evidence-of-attempt entry).** Mandatory one-line `eligibility cache: <verdict>`
  activity-log entry per Step C entry. Five variants (built / rebuilt / loaded / skipped-routing-off /
  skipped-codex-degraded) plus a wave-pinned exception. Makes the silent-skip failure mode impossible to
  hide: if Step C ran, the activity log shows what happened. The optoe-ng pattern (Step C step 1 ran zero
  times across an entire plan) becomes a glaring absence flagged by doctor check #21.

- **P2 (Step 3a precondition halt).** Before any task routing, verify `eligibility_cache` is loaded under
  `codex_routing != off`. If absent, HALT — do NOT silently fall through to inline. Branches on
  `config.codex.unavailable_policy`: `degrade-loudly` (default) surfaces a 4-option AskUserQuestion (Rebuild
  cache / Run inline with degradation marker / Set codex_routing: off / Abort); `block` sets status: blocked
  with a wave-mode-aware single-writer exception (the block-write defers to wave-end barrier when in a wave).
  Turns the previous "silent fallthrough" default into the loudest possible signal short of a process exit.

- **P3 (Resume sanity check for prior silent-failure footprint).** On every Step C resume entry, scan the
  activity log for `**Codex:** ok`-annotated tasks completed inline without `degraded-no-codex`
  decision_source. If found, append `## Notes` warning + 4-option AskUserQuestion (Continue / Run doctor /
  Investigate transcript / Suppress). Forensic recovery: pre-v2.4.0 plans without P1's evidence get one
  shot at human attention before being silently ignored forever.

- **P4 (`config.codex.unavailable_policy`).** New schema key under `codex:`. Default `degrade-loudly`
  preserves Fix 1's behavior. Opt-in `block` swaps Step 0's loud-degrade and Step 3a's precondition AUQ
  paths for hard halts. For users who'd rather a stuck plan than silent-codex-skip.

- **P5 (Doctor check #21).** Catches the activity-log footprint of the silent-skip failure (no
  `eligibility cache:` entry across the whole plan despite codex_routing != off + ≥1 task completion).
  Distinct from #20 which catches the cache-FILE footprint — both fire together on plans that pre-date
  v2.4.0 OR experienced silent skip; either alone catches single-source-of-truth corruption. Doctor table
  bumped 20 → 21 checks; parallelization brief mirrored.

**Stats command:** new `/masterplan stats` verb (Step T) shells out to `bin/masterplan-routing-stats.sh`
(~280-line bash + python3 heredoc). Discovers plans dirs via active worktree's plans/ + sibling
`.worktrees/*/docs/superpowers/plans/` (matches Fix-3's hook fan-out); dedupes by slug across linked
worktrees (linked worktrees check out the same plans/ files at different absolute paths). Per-plan stats:
codex/inline tag census, inline model breakdown (Sonnet/Haiku/Opus from `[subagent: <model>]` activity-log
hints + subagents.jsonl `model` field), token totals by `routing_class`, decision_source breakdown,
silent-skip detection (matches `**Codex:** ok` plan annotations against inline completions), health flags
(degraded / cache-missing / cache-evidence-missing / silent-skip-suspected). Three output formats: table
(default), JSON, GitHub-flavored md. Smoke output from petabit-os-mgmt: 7 plans, 31 codex / 32 inline
(49.2% codex), opus_share=N/A (subagents.jsonl backfill needs next /masterplan turn). From optoe-ng:
project-review correctly flags `silent-skips=5`.

**Sync'd locations updated** (per CLAUDE.md anti-pattern #4 + reviewer findings): doctor table count
20 → 21 in both `commands/masterplan.md:1278` and `docs/internals.md:511`; new doctor check #21 row in
both files; verb routing table mirror in `docs/internals.md:537` includes `stats`; reserved-verbs prose
mirror in `docs/internals.md:561` includes `stats` (M-1 fix); Step 0 self-reference at
`commands/masterplan.md:55` cites both #20 and #21 (M-2 fix); P2 `block` branch carries wave-mode
single-writer exception language (M-3 fix); README.md command-reference table + reserved-verbs
disclaimer + new "Routing stats" subsection; `docs/design/telemetry-signals.md` cross-reference
paragraph pointing at `bin/masterplan-routing-stats.sh`; `commands/masterplan.md` verb routing table +
reserved-verbs warning + argument-parse precedence list + frontmatter `description:` all include `stats`.

**Verification:** `bash -n` clean on hook (untouched, regression check) + new stats script. Negative greps
clean: zero `Total: 20 checks` / `all 20 checks` references in active docs (CHANGELOG retains historical
wording). Smoke from both petabit-os-mgmt (rich-data case, 31/32 split) and optoe-ng (silent-failure case,
project-review flagged with 5 silent-skips). Three output formats render cleanly. Fresh-eyes Sonnet
code-reviewer dispatched per CLAUDE.md anti-pattern #5 — caught three MEDIUM issues (M-1: internals.md
reserved-verbs missing `stats`; M-2: Step 0 self-ref missing #21; M-3: P2 block-branch single-writer
violation in wave mode) and two LOW (L-1: dead `opus_tokens` var; L-2: render_md crashes on None
codex_share_pct) — all five fixed.

**Followups (for future session):** Sample a real /masterplan turn after upgrade to confirm P1's evidence
entry actually lands (the orchestrator must follow the new MUST). Validate P3 doesn't false-positive on
plans that legitimately ran before any codex tagging existed (the codex_ok_tasks set is empty for those
→ silent_skip_count stays 0). Consider adding `--watch` flag to stats script for live tail use during
long /masterplan loops.

---

## 2026-05-04 - Unreleased - codex routing observability (Fixes 1-5)

**Scope:** User reported "no evidence codex delegation/review is working" across two
active projects (`../optoe-ng`, `../petabit-os-mgmt`). Audit found codex IS provably
firing in petabit-os-mgmt (31+ `[codex]` tags + 4 eligibility caches across 5 status
files), but silently NOT firing in optoe-ng's project-review plan despite
`codex_routing: auto + codex_review: on` (no eligibility cache, zero routing tags,
no degradation marker). Telemetry hook also broken everywhere: 0-line subagents.jsonl
files, missing telemetry sidecars for worktree-resident plans.

**Why these specific fixes:**

- **Fix 1 (Step 0 codex-degradation visibility).** Original spec said degradation
  marker writes "on next status-file update" — kickoff sessions that aborted before
  any subsequent write left the user with no signal that codex was bypassed. Now
  writes immediately on the next of {Step B3 close, Step C step 1 first write,
  Step I3} and falls back to a `## Notes`-only forced write if none would naturally
  occur. Adds stdout warning + `## Notes` one-liner. The optoe-ng failure pattern
  was exactly this: status frontmatter still claimed `codex_routing: auto` but no
  routing actually happened.

- **Fix 2 (doctor check #20).** Codex routing configured + eligibility cache absent
  + activity log shows ≥1 routing/completion entry → footprint of silent degradation.
  Stands on its own when #18 doesn't fire (codex re-installed by lint time).

- **Fix 3 (telemetry hook walks worktrees).** `git rev-parse --show-toplevel` returns
  the active worktree root; the optoe-ng project-review plan lives in
  `.worktrees/project-review/docs/superpowers/plans/`, invisible to a hook firing
  from the main worktree. Now fans out across `<root>/.worktrees/*/docs/superpowers/plans/`,
  matches by `worktree:` field equality OR `$PWD` prefix OR branch, picks
  most-recently-modified. Guarded against stray non-worktree dirs under `.worktrees/`
  via `[[ -e "$wt/.git" ]]` check.

- **Fix 4 (subagents.jsonl agent_id dedup).** Cursor file was plan-keyed (not
  transcript-keyed) and broke across multi-session runs — typical symptom: cursor
  pinned at end-of-transcript-from-session-1 means session-2's first N lines
  silently skipped. New mechanism reads existing JSONL into a seen-set and dedups
  by `agent_id` (16-byte hex unique per dispatch). Also adds `routing_class` field
  per record (`"codex"` / `"sdd"` / `"explore"` / `"general"`) for greppable
  routing-distribution analytics.

- **Fix 5 (pre-dispatch routing visibility, user-requested).** Today routing
  decisions are only visible POST-completion via `[codex]/[inline]` tags buried
  inside multi-line activity-log entries. Now Step 3a emits a stdout banner
  (`→ Task T9 (...) → CODEX (annotated **Codex:** ok; 4 files)`) AND a pre-dispatch
  activity-log entry (`routing→CODEX (annotation; 4 files in scope)`) BEFORE
  dispatching, plus stamps the eligibility cache with `dispatched_to`/`dispatched_at`/
  `decision_source` fields. Step 4b gets symmetric `review→CODEX` / `review→SKIP`
  pre-dispatch entries with skip-reason templates. Two activity-log lines per task
  (sometimes three when codex_review fires) — annotated in Step 4d's rotation note.

**Sync'd locations updated** (per CLAUDE.md anti-pattern #4 / fresh-eyes review
findings): doctor table count `19 → 20` in both `commands/masterplan.md:1216` and
`docs/internals.md:510` (parallelization brief mirrors the count); doctor check #19
description acknowledges legacy cursor file; `docs/design/telemetry-signals.md:122`
rewritten to describe agent_id dedup; `commands/masterplan.md:200` (telemetry capture
contract) updated to mention `routing_class` and the v2.4.0 dedup mechanism.

**Verification:** `bash -n hooks/masterplan-telemetry.sh` clean. Negative greps
clean (`Total: 19 checks`, `all 19 checks`, `Step A's resume-confirm write`,
`Cursor-based incremental parsing` — none in current-tense docs; CHANGELOG retains
historical wording per convention). Positive greps confirm every Fix landed.
Fresh-eyes Sonnet code-reviewer dispatched per CLAUDE.md anti-pattern #5 caught two
HIGH drift issues (internals.md count + telemetry-signals.md cursor description) and
three MEDIUM issues (Step A phantom step, Step 4d rotation note missing, hook
worktree-dir guard) — all five subsequently fixed.

**Followups (for future session):** doctor checks for the new `**Codex:** ok` +
`routing→` consistency (e.g., a task annotated `**Codex:** ok` whose post-completion
entry shows `[inline]` with no `degraded-no-codex` decision_source could surface as
warning #21 — but that's marginal observability for a marginal failure mode and not
worth a check yet). Smoke tests on the in-progress petabit-os-mgmt
`phase-3-netconf-transport` plan and the optoe-ng `.worktrees/project-review` plan
pending user's next /masterplan turn — both will exercise the changes.

---

## 2026-05-04 - Unreleased - resume-first default + local-only telemetry guard

**Scope:** Fresh audit follow-up plus user-requested behavior change for bare
`/masterplan`, telemetry check-in prevention, and local Codex default reasoning
effort.

**Findings fixed:**

- Bare `/masterplan` still defaulted to the broad phase/operations menu even
  though the common interrupted-work path is resume. Step M is now
  resume-first: auto-route current/only in-progress plans to Step C, send
  ambiguous active work to Step A list+pick, and show the broad menu only when
  no active plan exists.
- Telemetry sidecars could be generated as normal untracked files, so a broad
  `git add -A` in this repo or a downstream user's repo could stage local
  runtime data. The repo now ignores generated telemetry sidecars, and the Stop
  hook writes a managed `.git/info/exclude` block before creating telemetry.
  If any would-be sidecar is tracked or cannot be ignored, telemetry is skipped.
- Audit drift: docs still said the command prompt was ~1100 lines, telemetry
  signal docs said "three JSONL streams," release docs still referenced v2.2.3,
  and v2.3.0 dispatch-site notes described inline edits instead of the live
  central contract table.
- Follow-up docs gap: README install instructions were CLI-first and did not
  tell Claude Desktop users to use the Code tab, the + → Plugins browser, scope
  choices, `/reload-plugins`, or the `/superpowers-masterplan:masterplan`
  collision fallback.

**Verification:** `git diff --check`, `bash -n hooks/masterplan-telemetry.sh`,
JSON validation, `claude plugin validate .`, `claude plugin validate
.claude-plugin/plugin.json`, stale-wording greps, tracked-telemetry greps, and
two temporary Git repo hook smokes passed. The positive smoke proved
`smoke-telemetry.jsonl`, `smoke-subagents.jsonl`, and
`smoke-subagents-cursor` are ignored by `.git/info/exclude` and not staged by
`git add -A`. The negative smoke proved an already tracked telemetry sidecar
causes the hook to skip new telemetry writes.

**External config:** `/home/ras/.codex/config.toml` top-level
`model_reasoning_effort` is now `high`; the explicit `[profiles.deep]` override
remains `xhigh`.

---

## 2026-05-04 — v2.2.3 — marketplace readiness and Claude validator fixes

**Scope:** Pre-public-release audit for Claude plugin directory submission.

**Findings fixed:**

- `claude plugin validate .` failed before this pass because
  `.claude-plugin/plugin.json` used npm-style `repository: {type,url}` while
  current Claude Code expects a string, and `commands/masterplan.md`
  frontmatter used an unquoted description containing `Verbs:`. Both blocked
  marketplace-quality packaging.
- README documented `/plugin marketplace add rasatpetabit/superpowers-masterplan`,
  but the repository did not contain `.claude-plugin/marketplace.json`, which
  current Claude Code marketplace docs require for GitHub marketplace sources.
- Plugin metadata treated the official `superpowers` dependency as prose-only.
  v2.2.3 declares it as a plugin dependency and allowlists
  `claude-plugins-official` for cross-marketplace resolution.

**Changes:**

- `.claude-plugin/plugin.json`: version `2.2.3`, concise marketplace-facing
  description, string `repository`, explicit
  `superpowers@claude-plugins-official` dependency.
- `.claude-plugin/marketplace.json`: new direct-install catalog for
  `rasatpetabit-superpowers-masterplan`.
- `commands/masterplan.md`: quoted YAML frontmatter description.
- `README.md`, `CHANGELOG.md`, `CLAUDE.md`, `docs/internals.md`: release and
  packaging docs synced to v2.2.3.
- `docs/release-submission.md`: durable submission checklist plus form-copy draft
  for the Claude plugin directory and Anthropic Verified request.

**Verification:** `claude plugin validate .`, `claude plugin validate
.claude-plugin/plugin.json`, JSON validation, `bash -n
hooks/masterplan-telemetry.sh`, `git diff --check`, local link check, and
`claude plugin tag --dry-run --force .` all pass. Isolated clean install smoke
with a temporary `HOME` passed after adding the official marketplace over HTTPS:
Claude installed `superpowers-masterplan@rasatpetabit-superpowers-masterplan`
and auto-installed one dependency (`superpowers`).

---

## 2026-05-04 — v2.2.2 — remove standing "no backward-compat / hard-cut renames" rule

**Scope:** Documentation-only removal of the project-encoded prohibition on backward-compatibility aliases and dual-load shims when renaming. Going forward, decisions about migration aliases for breaking renames are made case-by-case.

**Changes:**

- **`CLAUDE.md`:** deleted the "Top anti-patterns" #2 bullet ("Don't add backward-compatibility shims when renaming things…"); renumbered prior #3–#6 to #2–#5.
- **`docs/internals.md`:** deleted the `### Why hard-cut name changes` subsection and the corresponding bulleted anti-pattern (`Adding backward-compat shims for breaking name changes.`) under "Architectural anti-patterns".
- **`WORKLOG.md` v2.2.0 entry:** scrubbed two policy-framing references — Phase 1 doc-revisionism narrative no longer mentions the (now-deleted) `Why hard-cut renames` → `Why hard-cut name changes` heading rewrite; Phase 2 verb-rename narrative no longer prefaces with "Hard-cut, no alias." Functional record of what was changed in v2.2.0 stays.
- **`~/.claude/projects/-home-ras-dev-superpowers-masterplan/memory/feedback_no_backward_compat_aliases.md`:** deleted entirely.
- **`MEMORY.md` index:** deleted the line linking to the removed feedback file.
- **CHANGELOG `[Unreleased]`:** new `### Removed` entry; promoted to `[2.2.2] — 2026-05-04`.
- **`plugin.json` 2.2.1 → 2.2.2** + description tweak.
- **README "Current release"** bumped to v2.2.2.

**Key decisions (the why):**

- **No replacement rule.** The user explicitly removed the rule, not "softened" or "qualified" it. Adding a successor like "always confirm before adding aliases" would defeat the point. The absence of a standing rule is the new state; case-by-case judgement applies.
- **Past breaking renames stay shipped.** `/masterplan new` stays renamed to `/masterplan full`. `claude-superflow` stays rebranded to `superpowers-masterplan`. The v2.2.0 CHANGELOG `(breaking — no alias)` framing for the released `new → full` rename is kept verbatim — release notes are an append-only historical record.
- **Scrub WORKLOG narratives, not CHANGELOG entries.** The user picked this scope explicitly via AskUserQuestion. WORKLOG framings encoded the rule prescriptively in past-tense narrative ("Hard-cut, no alias"); CHANGELOG entries describe released behavior factually ("(breaking — no alias)"). The latter stays as documentation of what shipped.
- **Patch-level version bump (2.2.1 → 2.2.2).** Doc-only deletion with no orchestrator behavior change. No CLI surface change. No config schema change.

**Verification:**

- Negative grep: `hard-cut`, `dual-load`, `no backward-compat` return zero hits in `CLAUDE.md`, `docs/internals.md`, `commands/`, `skills/`, `hooks/`, `.claude-plugin/`, `README.md`. WORKLOG/CHANGELOG hits are limited to the new WORKLOG entry (this one) and the v2.2.0 CHANGELOG release-notes lines.
- CLAUDE.md anti-pattern numbering: 5 sequential bullets (1–5) confirmed.
- MEMORY.md index: 5 entries (was 6); `feedback_no_backward_compat_aliases.md` no longer present.
- Standard pre-commit gates: `bash -n hooks/masterplan-telemetry.sh` clean, `plugin.json` JSON-valid.

**Branch state at end of v2.2.2:**

- 1 commit ahead of v2.2.1 on `main`.
- Tag `v2.2.2` to be created locally; pushed alongside the commit.
- Working tree clean.
- plugin.json: 2.2.2.

---

## 2026-05-04 — v2.2.0 — doc revisionism + verb rename + no-args picker

**Scope:** Three threads:

1. **Doc revisionism.** Pre-v1.0.0 (v0.x) release-history references removed everywhere. CHANGELOG older blocks deleted entirely + remaining v0.x mentions in v1.0.0/v2.0.0 entries scrubbed; v2.0.0 entry's rename framing dropped (Renamed section + rename-step migration notes deleted; mechanical `/superflow → /masterplan` and `claude-superflow → superpowers-masterplan` substitutions throughout v1.0.0 + v2.0.0 entries). README "Path to v2.0.0" → "Releases since v1.0.0" with v0.x bullets removed; Project status reworded to drop rename framing; `/superflow` alias non-feature item removed from Roadmap. `docs/internals.md` v0.x parentheticals dropped from "Why" section headings; remaining "rename"/"renamed" mentions reworded throughout. `docs/design/intra-plan-parallelism.md` + the v1.1.0 spec: "v0.1 → v0.2 → v0.3 → v0.4 → v1.0.0" deferral-chain framing rewritten as "deferred prior to v1.0.0". WORKLOG v2.0.0 entry's rename narrative trimmed (was 6 threads → now 5); functional deliverables (parallelism Slice α, Codex defaults, internal docs) preserved.
2. **Verb rename `new` → `full`.** All sync'd locations updated: frontmatter description, Step 0 verb routing table rows ("full" no-topic + "full <topic>"), reserved-verbs warning, argument-parse precedence list, Step P "no candidates" example, README verb table + quick-start examples + reserved-verb prose + Aliases-and-shortcuts table, `docs/internals.md` Step 0 mirror.
3. **Two-tier no-args picker (Step M).** New section before Step A. `/masterplan` (no args) surfaces `AskUserQuestion("What kind of work?")` with 4 options (Phase work / Operations / Resume in-flight / Cancel). Tier 2a picks a phase verb + topic prompt; Tier 2b picks an operation verb. "Resume in-flight" delegates to Step A's existing list+pick. "Cancel" exits cleanly. Step 0 routing table's `_(empty)_` row updated to point at Step M; `execute (no path)` row reworded to no longer cross-reference "bare empty" (they diverge now).

**Key decisions (the why):**

- **Doc revisionism over git-history rewrite.** Git log + tag history persist showing the v2.0.0 rename release. The user-facing docs are scrubbed; the git-history mismatch is accepted by user instruction. Mitigation: anyone curious can `git log v2.0.0..HEAD` for the actual rename history.
- **v2.2.0 minor bump for breaking verb rename.** `new` → `full` is technically a breaking change (would normally warrant a major bump), but v2.0.0 shipped today (2026-05-04) and no users have memorized the verb yet — functionally no-impact. Doc revisionism + picker are additive. Net minor bump per semver judgment call. CHANGELOG migration notes flag the breaking change explicitly.
- **Two-tier picker not one-tier (8 verbs).** CD-9's 2-4 option cap on `AskUserQuestion`. Two-tier respects the cap while still surfacing all 8 verbs cleanly. Tier 1 separates "Phase work" (the common case for new tasks) from "Operations" (the housekeeping verbs). "Resume in-flight" gets its own top-level option because it's frequent and would otherwise be hidden under Operations.
- **Picker delegates to Step A for "Resume in-flight" rather than re-implementing the worktree scan.** One canonical site for the in-progress-plans logic; Step M routes there with no further prompt. Keeps Step M small and Step A unchanged.
- **Local dir + memory dir rename deferred to AFTER push (Phase 5).** Mid-session rename would invalidate the Bash cwd and Claude's memory-dir hashing. Push first → rename last → resume in a new session in the renamed dir.
- **`retro` skill check confirmed Step R implementation.** The `superflow-retro` skill was deleted in v1.0.0 (consolidated into Step R inside `commands/masterplan.md`). The picker can offer `retro` confidently because Step R is functional.

**Operational notes:**

- 9 commits this release (Phase 1 = 5 commits scrub passes, Phase 2 = 3 commits verb rename, Phase 3 = 1 commit picker; this Phase 4 commit + Phase 5 rename happens after push).
- Halt-mode discriminator suite: 32 unique lines (was 26 pre-Phase 3). The +6 are from Step M's halt_mode mentions in Tier 2a routing (brainstorm/plan/full set the appropriate halt_mode) + Notes section. Not orphans; intentional new mentions.
- Verification spec from the plan: pre-v1.0.0 reference audit (`grep -rn -i 'v0\.|claude-superflow|/superflow' commands/ README.md CHANGELOG.md WORKLOG.md docs/ .claude-plugin/`) returns 0 lines. Verb rename completeness: `/masterplan new\b` returns 0; `/masterplan full\b` returns 6+ across the three sync'd files.
- Doctor table size: 18 rows (unchanged from v2.0.0+v2.1.0).

**Verification gaps (carried as v2.x followups):**

- **Two-tier picker behavior not yet first-user-tested.** Markdown logic; runtime behavior depends on a real bare-`/masterplan` invocation. First-user verification is the smoke test.
- **Verb rename `new` → `full` not yet smoke-tested with $ARGUMENTS.** Step 0's argument-parse precedence (line 84) has been updated to match `full` instead of `new` in the verb set. A canned `$ARGUMENTS` self-test would catch any drift; deferred.
- All v2.1.0 followups still apply (canned $ARGUMENTS self-tests, macOS hook smoke, etc.).

**Known followups (post-v2.2.0):**

- **Telemetry signal for picker option chosen** — informs whether the tier-1 ordering is well-calibrated. If users overwhelmingly pick "Resume in-flight" the order should change; if they pick "Phase work → full" we should consider promoting `full` as the bare-default. Defer until usage data exists.
- **Doctor check for "old verb tokens in user-authored plans"** — flag plans where activity log mentions `/masterplan new` (suggests user is on outdated muscle memory). Niche; defer.
- All v2.0.0/v2.1.0 followups still apply.

**Branch state at end of v2.2.0 (pre-push):**

- 10 commits ahead of v2.1.0 on `main` (Phase 1: 5 commits, Phase 2: 3 commits, Phase 3: 1 commit, this Phase 4 release commit: 1).
- Tag `v2.2.0` to be created locally; push deferred to user-approval gate.
- Working tree clean.
- plugin.json: 2.2.0; description mentions the v2.2.0 surface (picker + verb rename).
- Phase 5 (local dir rename + memory dir migration) runs AFTER push.

---

## 2026-05-04 — v2.0.0 — Slice α intra-plan parallelism + Codex defaults on + internal docs

**Scope:** Single coherent v2.0.0 release bundling five threads:

1. **Slice α of intra-plan task parallelism:** read-only parallel waves only — verification, inference, lint, type-check, doc-generation. Wave assembly + parallel SDD dispatch via Agent + wave-completion barrier in Step C step 2; single-writer status funnel + wave-aware activity log rotation in Step C 4d; files-filter union in Step C 4c; per-member outcome reconciliation (completed | blocked | protocol_violation) in Step C step 3; wave-count threshold in Step C step 5. Implementation tasks remain serial; Slice β/γ deferred per `docs/design/intra-plan-parallelism.md`.
2. **Codex defaults flipped to on:** `codex.routing: auto` (already default) + `codex.review: on` (was `off`). Step 0 codex-availability detection auto-degrades both to `off` for the run with one-line warning when codex plugin not installed. Doctor check #18 surfaces persistent misconfiguration as a Warning during lint.
3. **Codex documentation:** new top-level `## Codex integration` section in README (~490 words: why/how/defaults/install/disable/cross-references). Plugin.json description tweaked.
4. **Internal documentation:** `CLAUDE.md` (always-loaded ~620 words) + `docs/internals.md` (~8000 words, 15 sections: architecture, dispatch model, status format, CD rules, operational rules, wave dispatch, Codex integration, telemetry, doctor, verb routing, design history, recipes, anti-patterns, cross-references). Migrates institutional knowledge from earlier WORKLOG entries that were pruned.
5. **History pruning:** deleted 5 older spec/plan files (small-fixes, subcommands); trimmed WORKLOG to drop earlier entries; per-version release detail lives in CHANGELOG.

**Key decisions (the why):**

- **Slice α (read-only waves) over Slice β/γ.** Depth-pass on candidate mitigations found that the SDD-wrapper alone doesn't solve the central git-index-race for committing work — concurrent commits to the same branch race the index regardless of wrapper. Read-only work sidesteps it entirely. Slice β (~8-10d serialized commit) and Slice γ (~10-15d full per-task worktree subsystem) inherit the unsolved committing-work problem; deferred with sharpened, measurable revisit trigger in `docs/design/intra-plan-parallelism.md`.
- **Codex defaults flipped to on (auto routing + on review).** Most users who install Codex want adversarial review by default but had to explicitly enable it under v1.0.0. Graceful-degrade on missing-codex makes default-on safe (one-line warning, run continues with both off). New doctor check #18 catches persistent on-but-missing misconfiguration.
- **Internal docs written BEFORE history pruning.** The user's instruction to delete earlier plans/worklogs would have lost the WORKLOG decisions/rationale captured across earlier entries. A migration step moved that knowledge into `docs/internals.md` §12 (Design decisions + deferred items) before the source was deleted.
- **Aggressive WORKLOG trim.** User picked to keep only the v2.0.0 entry. CHANGELOG retains the full release history.
- **Wave dispatch implementation order:** Tasks 1 (cache extension) → 2-4 (wave assembly + 4-series + failure handling) — bundled into one commit since tightly coupled. Tasks 5-8 (doctor checks + B2 brief + flag + config) — bundled. Tasks 9 (hook), 10 (telemetry-signals), 11 (intra-plan-parallelism design doc), 12 (README) — separate commits. Smoke verification deferred — markdown-only project; documented as manual verification step in `docs/internals.md` §13 (Common dev recipes).
- **Hook portability sub-bug found and fixed during smoke test.** Original wave_groups extraction used gawk's `match()` with array argument (third arg) — gawk-only. Replaced with portable awk + grep + sed pipeline. Linux smoke-tested; first-turn caveat verified (tasks=0 when no prior record), wave detection verified (tasks=3 + waves=["verify-v2"] after delta).

**Operational notes:**

- 16 commits on the v2.0.0 release path; tasks tracked via `TaskCreate` / `TaskUpdate` (5 phase-level tasks).
- Halt-mode discriminator suite re-grepped after each Step C / B-section edit — 25 references throughout, no orphans.
- Doctor table size: 18 rows. Step D parallelization brief says "all 18 checks" — matches.
- README structure: 16 top-level sections after v2.0.0 (added `## Codex integration` between `## Design philosophy` and `## Install`).
- Internal docs (`docs/internals.md`) committed BEFORE history pruning — institutional knowledge migrated successfully.
- `pwd` recovery needed once during a smoke test that `rm -rf`'d its tmpdir without `cd` first; no harm.

**Verification gaps (carried as v2.x followups):**

- **macOS hook smoke verification.** Linux smoke-tested only. Hook code is portable-by-construction (no GNU-only flags in the new wave_groups path; uses portable `head -n1` for find result, `stat -c '%Y' || stat -f '%m'` dual form for transcript-resolution fallback). README Option C softens the cross-platform claim.
- **Wave dispatch end-to-end smoke test.** Markdown-only project; the orchestrator IS the prompt. Dispatching a real wave requires a real `/masterplan execute` invocation against a hand-crafted parallel-group plan. Documented as manual verification step in `docs/internals.md` §13. Acceptance criterion #16 from the spec marked deferred until first user runs a real wave.
- **Codex graceful-degrade smoke.** Step 0 codex-availability detection logic landed as markdown logic (heuristic: scan system-reminder skills list for `codex:` prefix). Behavior in practice depends on Claude Code's actual context delivery. Worth verifying on first user run.

**Known followups (post-v2.0.0):**

- **Slice β/γ revisit trigger** — telemetry-derived; doctor check candidate (deferred to v2.0.x): scan recent plans for the trigger condition (≥3 parallel-grouped committing tasks, wall-clock >10 min). Telemetry fields `tasks_completed_this_turn` + `wave_groups` provide the data.
- **Codex CLI/API concurrency model verification** — affects whether a future slice could allow Codex tasks in waves (FM-4 is currently conservative).
- **Canned-`$ARGUMENTS` self-test specs** for routing-table drift detection. Markdown spec exercising every verb's branch.
- **Doctor check for the three-place verb-list invariant** (frontmatter description, reserved-verbs warning, routing table).
- **macOS smoke verification** of the telemetry hook (gated on access to a macOS env).

**Branch state at end of v2.0.0:**

- 16 commits ahead of v1.0.0 on `main`.
- Tag `v2.0.0` created locally (Phase 4.4); push deferred to user-approval gate (Phase 4.5).
- Working tree clean.
- README Project status section updated to v2.0.0 framing.

---

## 2026-05-04 — v2.1.0 — README polish + gated→loose offer + Roadmap

**Scope:** Additive release on the v2.x track; no breaking changes. Three threads:

1. **README polish:** reordered `## Why this exists` to precede `## What you get`; appended a 6-bullet benefits paragraph to "Why this exists" (long-term complex planning, aggressive context discipline, dramatic token reduction, parallelism for faster operation, cross-session resume, cross-model review); added `### Defaults at a glance` sub-section under `## Configuration` (~50-line compact YAML block); added `## Roadmap` top-level section before `## Author` (6 deferred items + 4 documented non-features, each with measurable revisit trigger).
2. **Gated→loose switch offer:** new AskUserQuestion at Step C step 1 (after telemetry inline snapshot, before per-task autonomy loop). When `autonomy == gated` AND `config.gated_switch_offer_at_tasks > 0` AND plan task count ≥ threshold (default 15) AND not already dismissed, offer 4-option switch. Two new status frontmatter fields handle suppression: `gated_switch_offer_dismissed` (permanent per-plan) + `gated_switch_offer_shown` (per-session; re-fires on cross-session resume by design).
3. **Release bookkeeping:** plugin.json 2.0.0 → 2.1.0; CHANGELOG `[2.1.0]` block; this WORKLOG entry; tag + push.

**Key decisions (the why):**

- **Top-level `gated_switch_offer_at_tasks` config key, not nested under `autonomy:`.** Initial plan suggested `autonomy.gated_switch_offer_at_tasks` but YAML doesn't allow `autonomy: gated` (scalar) AND `autonomy: { gated_switch_offer_at_tasks: 15 }` (block) to coexist. Renaming the existing scalar would be a breaking change. Top-level key avoids the conflict. Cost: less namespaced, but unambiguous and additive.
- **Per-session `gated_switch_offer_shown` flag re-fires on cross-session resume by design.** Reasoning: when the user comes back to a paused plan after a break, they may have changed their mind about the gated friction. Asking once per session is the right cadence. Permanent dismissal is available via `gated_switch_offer_dismissed: true` for users who DON'T want re-prompting.
- **The orchestrator does NOT modify `.masterplan.yaml`** even when the user picks "Switch + don't ask again on any plan." It writes a `## Notes` entry recommending the change; user takes the action manually. Per CD-2 — config files are user-owned.
- **Default threshold = 15 tasks.** Educated guess. The audit-pass v1.0.0 plan was 12 tasks; the v1.1.0 wave-dispatch plan was 14 tasks; the v2.0.0 release plan was effectively 18+ tasks. 15 captures the "this is going to take a while" point. Easy to tune via config.
- **Reordered Why before What in README per user instruction.** Better reading flow: explain the value before the verb surface. The "thin orchestrator" framing in the tagline paragraph still introduces *what* it does at a high level, then "Why this exists" goes deep on value, then "What you get" enumerates the verbs.
- **Roadmap section frames deferred items as "decided NOT to ship yet, and why"** rather than "promised future work." Prevents the section from being read as a feature-request invitation. Each item has a measurable revisit trigger (per the existing `docs/design/intra-plan-parallelism.md` convention).
- **Defaults at a glance duplicates the Configuration section's schema.** Maintenance cost: when a default changes, both must update. Mitigation: keep the at-a-glance block deliberately compact (just `key: value` pairs, no inline explanation) so it's easy to eyeball-diff against the full schema.

**Operational notes:**

- 3 commits this release: Phase 1 README polish [9226037], Phase 2 gated→loose offer [9d22c5d], Phase 3 release (this commit). Smaller release than v2.0.0.
- Halt-mode discriminator suite: 26 unique lines (was 25 pre-Phase 2). The +1 is from the new Step C step 1 paragraph; not an orphan reference, just a contextually-correct mention.
- README structure verified post-changes: `## Why this exists` (line 9), `## What you get` (line 24), `## Configuration` with `### Defaults at a glance` + `### Full schema (with explanations)` sub-sections, `## Roadmap` (line 620) before `## Author`.
- Status file format docs updated in 3 places: README "Status file" section, commands/masterplan.md "Status file format" section, docs/internals.md §4 "Status file format". All three include both new optional fields with v2.1.0+ annotation.

**Verification gaps (carried as v2.x followups):**

- **Gated→loose offer behavior not yet first-user-tested.** Markdown logic; runtime behavior depends on a real `/masterplan execute` against a 15+ task plan under autonomy=gated. First-user verification is the smoke test. Documented in WORKLOG (this entry) + plan file's "Risks" section.
- Same gaps as v2.0.0 carried forward: macOS hook smoke verification, canned `$ARGUMENTS` self-test specs.

**Known followups (post-v2.1.0):**

- **Telemetry signal for "user dismissed the offer"** — would help inform whether threshold default 15 is well-calibrated. Add `gated_switch_offer_outcome: switched|stayed|never_offered` field to the Stop hook's record. Defer until a real signal exists from a few users.
- **Doctor check for unusually-long-but-still-gated plans** — flag plans where `task_count > 20` AND `autonomy: gated` AND `gated_switch_offer_dismissed: true` for >7 days as candidates for re-revisit. Niche.
- All v2.0.0 followups still apply (Slice β/γ trigger doctor check, Codex concurrency verification, $ARGUMENTS self-tests, macOS smoke).

**Branch state at end of v2.1.0:**

- 3 commits ahead of v2.0.0 on `main`.
- Tag `v2.1.0` created locally (Phase 3); push deferred to user-approval gate.
- Working tree clean.
- plugin.json: 2.1.0; description mentions the v2.1.0 surface.

---

## 2026-05-04 — post-v2.2.0 documentation sync

**Scope:** README, `docs/internals.md`, `CHANGELOG.md`.

**Why:** A post-release review found stale public and internal docs after v2.2.0:

- README still described bare `/masterplan` as direct list+pick instead of the Step M two-tier picker.
- README still marked Project status as v2.1.0 and omitted the v2.2.0 release bullet.
- README's full schema block still had `codex.review: off` and omitted v2.x keys already present in the defaults-at-a-glance block.
- `docs/internals.md` Step 11 still mirrored empty `$ARGUMENTS` as Step A instead of Step M.

**Decisions:** Kept this as a docs-only sync. Did not alter `commands/masterplan.md`; the command source already has Step M and v2.2.0 routing. README's `--no-parallelism` note now points users to `parallelism.enabled: false` for durable config instead of claiming a nonexistent `parallelism:` status-frontmatter field.

**Verification:** Re-run the standard doc drift greps before commit: README no-args wording, v2.2.0/current status, config defaults (`codex.review: on`, `parallelism:`, `gated_switch_offer_at_tasks`), internals Step M mirror, `bash -n hooks/masterplan-telemetry.sh`, and JSON validation for `.claude-plugin/plugin.json` + `.claude/settings.local.json`.

---

## 2026-05-04 — README simplification + project audit

**Scope:** README cleanup plus a prompt/docs consistency audit before commit.

**Changes:**

- Rewrote README as a compact user guide (install, quick start, command reference, flags, config, advanced features, project status) and removed duplicated internals/history content now covered by `docs/internals.md` and `CHANGELOG.md`.
- Fixed command prompt flag docs to include `--no-codex-review`, to stop claiming `--parallelism` persists to status frontmatter, and to remove stale future-only wording about Slice α parallelism. Durable parallelism defaults live in `.masterplan.yaml`.
- Updated this handoff plus CHANGELOG Unreleased so the docs cleanup and audit findings are durable.

**Verification:** Run README structure/stale-reference checks, local-link existence checks, doctor-check table count, `git diff --check`, hook `bash -n`, and JSON validation before commit.

---

## 2026-05-04 — Step M0 inline status + doctor-tripwire preamble

**Scope:** Bare `/masterplan` now emits a structured orientation block before the Tier-1 picker fires.

**Changes:**

- **`commands/masterplan.md`:** new `### Step M0 — Inline status orientation` section inserted before `### Tier 1`. M0 enumerates plans across worktrees (parallel Bash glob over `git_state.worktrees`, bounded at 20), reads frontmatter inline (no Haiku — bounded count), runs 7 cheap tripwire checks against data already in memory (#2/#3/#4/#5/#6/#9/#10), emits a 1-line headline + up-to-3 plan bullets + optional `… and N more` tail + optional `· <K> issue(s) detected` flag, then fires Tier-1. Caches the parsed list in `step_m_plans_cache` for Step A reuse. Updated the verb routing table at line 60 to read "Step M0 → Step M". Updated the "Stay on script" guardrail at line 292 to acknowledge the structured preamble while reaffirming the no-tangents rule and adding "do NOT enumerate which doctor checks tripped — that's `/masterplan doctor`'s job."
- **Step A short-circuit:** new step 0 — if `step_m_plans_cache` is populated (Resume in-flight pick), skip the worktree scan + Haiku dispatch and use the cache directly. Avoids redundant scanning.
- **`docs/internals.md` Step 11 mirror:** routing table cell updated to "Step M0 → Step M"; new paragraph under "Bare `/masterplan` picker" describing M0's behavior + tripwire scope + cache.
- **CHANGELOG.md `[Unreleased]`:** added an `### Added` block describing M0 and the tripwire scope.

**Key decisions (the why):**

- **7 cheap checks, not all 18.** M0 runs on every bare `/masterplan` invocation; the latency budget is small. The 7 chosen checks (#2/#3/#4/#5/#6/#9/#10) are all derivable from data already in memory after the worktree scan + frontmatter parse — no extra subprocesses beyond one parallel `test -f` batch. The other 11 doctor checks (orphan plan files, archive files, telemetry growth, parallel-group invariants, codex-config) need separate enumeration that doesn't pay back at the bare-invocation latency budget. M0 is deliberately a tripwire, not a lint pass — it counts, doesn't enumerate. `/masterplan doctor` is still the canonical lint surface.
- **Headline + top-3 bullets, not headline-only.** The user explicitly asked for "where we're at in them" — a per-plan `current_task` + age payload satisfies that. Capping at 3 bullets keeps the preamble under one screen even on busy repos; truncation tail (`… and N more`) routes the user to "Resume in-flight" for the full list.
- **`No active plans.` empty-state line.** Always emit a one-liner so the user knows the scan ran. Slightly chattier than skipping but reassures the user that "no plans" is a real fact, not a missed scan. Picked over "skip preamble entirely" by user choice.
- **`step_m_plans_cache` reused by Step A, not parallel re-scan.** The cache writes once in M0; Step A reads-only. Keeps a single canonical scan per turn. Cache is transient — discarded at end-of-turn — so it doesn't pollute later cross-session resumes.
- **Updated "Stay on script" instead of inventing a new guardrail.** The line-292 paragraph already permitted "a one-line orientation"; M0's structured preamble is a permitted extension, not a new license. Reframing the existing guardrail (and explicitly forbidding per-check enumeration in the preamble) prevents future model drift more reliably than a separate guardrail elsewhere.
- **No new doctor check #19.** M0's tripwire reuses existing checks #2/#3/#4/#5/#6/#9/#10 by name + semantics. Doctor table size stays at 18; Step D's parallelization brief still says "all 18 checks." Per CLAUDE.md anti-pattern #4, the three sync'd verb-routing locations remain aligned (M0 is a sub-step of M, not a new verb — frontmatter `description:` and reserved-verbs warning unchanged).

**Verification:**

- `grep -n "Step M0" commands/masterplan.md docs/internals.md` should return ≥3 matches (M0 section header, Stay-on-script reference, internals mirror paragraph).
- `grep -n "step_m_plans_cache" commands/masterplan.md` should return ≥2 (M0 step 6, Step A step 0).
- Doctor table count unchanged: 18 rows. Step D parallelization brief still says "all 18 checks."
- `bash -n hooks/masterplan-telemetry.sh` clean. JSON validation on `.claude-plugin/plugin.json` clean.
- Halt-mode discriminator suite re-grep: count should remain unchanged (M0 introduces zero `halt_mode` mentions).
- Manual smoke runtime test deferred to first user invocation — markdown-only project; the orchestrator IS the prompt. Documented in `docs/internals.md §13` recipe pattern.

**Verification gaps (carried as followups):**

- **First-user smoke not yet performed.** M0's runtime behavior depends on a real bare `/masterplan` invocation against a repo with various plan-state combinations (empty, 1 plan, 5 plans, 1 corrupted plan). Smoke verification deferred to next user-driven invocation.
- **Cache-hit path in Step A not yet smoke-tested.** The `step_m_plans_cache` short-circuit is markdown logic; runtime behavior depends on the same first-user invocation picking "Resume in-flight" after seeing the M0 preamble.
- All v2.2.0 followups still apply (canned `$ARGUMENTS` self-tests, macOS hook smoke).

**Branch state at end of this change:**

- 1 commit ahead of v2.2.0 on `main` (after the WORKLOG cleanup commit).
- Tag deferred — this is an `[Unreleased]` addition, will fold into the next minor bump.
- Working tree clean post-commit.
- plugin.json: 2.2.0 (unchanged).

---

## 2026-05-04 — pre-public-release branch + worktree cleanup

**Scope:** Deleted both stray feature branches and the worktrees holding them, leaving only `main` locally and on origin.

**Changes:**

- Removed `feat/simplify-code-and-docs` (fully merged into main): worktree at `.worktrees/simplify-code-and-docs` removed via `git worktree remove`; branch deleted local + origin.
- Removed `feat/codex-persona-integration` (18 unmerged commits of pre-rename `superflow-persona` exploratory work, abandoned in favor of the v2.0.0 Codex defaults-on direction): registered worktree at `/home/ras/dev/claude-superflow/.worktrees/codex-persona-integration` was already prunable (parent dir gone post-rename) — pruned via `git worktree prune`. Orphan checkout dir at `.worktrees/codex-persona-integration` (not in `git worktree list`, leftover from before the rename) deleted via `find -depth -delete`. Branch hard-deleted local-only (never had a remote ref).
- Confirmed v2.2.0 tag was already on origin; earlier impression that it was unpushed came from a truncated `tail -10` of the tag list.

**Why:** Pre-public-release tidy-up. Stray branches and an orphan worktree dir would confuse contributors browsing the GitHub repo or running `git worktree list` locally on a fresh clone. Reflog retains the deleted SHAs ~90 days if the codex-persona thread ever needs salvaging.

**Verification:** `git branch -a` shows only `main` + `origin/main`/`origin/HEAD`. `git worktree list` shows the single main worktree. `git ls-remote origin` shows only `refs/heads/main` and the four version tags. Working tree clean. `bash -n hooks/masterplan-telemetry.sh` and JSON validation of `.claude-plugin/plugin.json` both pass. Pre-rename grep audit (`claude-superflow|/superflow`) returns only the meta-references in WORKLOG.md describing the audit pattern itself, per the v2.2.0 verification spec.

---

## 2026-05-04 — Step M loop-safety guardrail

**Scope:** One-paragraph addition to Step M's Notes after observing a remote-control session where the model routed to Step M correctly but skipped surfacing the Tier-1 `AskUserQuestion` picker, instead pitching an unrelated browser-visualization feature and ending with a free-text "Want to try it?" — fatal in `/loop` and remote sessions where no human types between turns.

**Why:** The script already says "surface AskUserQuestion(...)" but didn't explicitly forbid prose tangents around it. The new "Stay on script." note makes the prohibition explicit: the picker IS the user-facing surface; no adjacent feature upsells; any `?` outside an `AskUserQuestion` is a bug. Complements the global `feedback_use_askuserquestion_consistently.md` rule with the loop-fatal-interaction angle.

**Verification:** `grep -n "Stay on script" commands/masterplan.md` confirms the note landed at Step M's Notes (line 292). No other steps changed; verb routing table and Step 0 logic untouched.

---

## 2026-05-04 — v2.3.0 — model-dispatch contract + per-subagent telemetry layer

**Scope:** Two threads bundled into one minor release.

1. **Cost-leak fix.** Every `Agent` tool dispatch site in `commands/masterplan.md` now structurally requires a `model:` parameter; previously the dispatch model was prose-only ("dispatch parallel Haiku agents") and the orchestrator-Claude (Opus 4.7) emitted Agent calls without `model:`, so spawned subagents inherited Opus silently.
2. **Per-subagent observability.** Stop hook now captures one record per Agent dispatch into `<plan>-subagents.jsonl`, with full token breakdown (`input/output/cache_creation/cache_read`), duration, dispatch-site attribution, subagent_type, model, and tool_stats. Six jq cookbook recipes added so finding the biggest token consumers is tractable instead of guessing.

**Trigger:** A real 2-day /masterplan-heavy session on a non-trivial codebase consumed $487 with **94% Opus** ($458) vs 5% Sonnet ($26) vs 1% Haiku ($2). Completely inverted from the design intent. Investigation found that ~15 dispatch sites all said the right thing in narrative prose but never told the orchestrator to pass `model:` as a structural Agent-tool parameter. The follow-on observation: even with the contract in place, finding the NEXT cost driver requires per-dispatch visibility — hence the hook upgrade.

**Changes:**

- **`commands/masterplan.md`:**
  - Added `### Agent dispatch contract` subsection (normative MUST + value-by-use table + Codex exemption + recursive-application clause + telemetry-capture clause + dispatch-site tag table) under `## Subagent and context-control architecture`, between the existing "Model selection guide" and "Briefing rules — the bounded brief".
  - 14 inline dispatch sites updated with explicit `model: "haiku"` / `model: "sonnet"` parameters: Step A status parse, Step B0 worktree scan, Step C step 1 eligibility cache, Step C step 2 wave dispatch, Step C step 2 SDD invocation (with model-passthrough override clause), Step C 3a Codex EXEC (exempt), Step C 4b Codex REVIEW (exempt), Step I1 discovery, Step I3.2 fetch (per-candidate haiku/sonnet/no-Agent), Step I3.4 conversion, Step S1 situation gather, Step R2 retro source, Step D doctor checks, Completion-state inference.
  - Step C blocker re-engagement gate's option 2 ("Re-dispatch with a stronger model") now structurally re-dispatches with `model: "opus"` — was a UI-only promise.
  - Doctor check #19 (orphan subagents file) added; #12 extended to also catch `<slug>-subagents.jsonl > 5 MB`.
  - Step D parallelization brief updated from "all 18 checks" to "all 19 checks".
  - DISPATCH-SITE tag value table embedded in the contract section enumerating the per-step tag values (Step A frontmatter parse / Step B0 related-plan scan / etc., 14 sites).
- **`hooks/masterplan-telemetry.sh`:** added ~120 lines — section 8 of the hook parses the parent transcript at end-of-turn, builds an in-memory tool_use index from all assistant Agent dispatches, joins with toolUseResult lines beyond cursor, and emits one JSONL record per dispatch to `<plan>-subagents.jsonl`. Cursor file `<plan>-subagents-cursor` stores the last-processed line count for incremental parsing. Records carry: ts, plan, session_id, tool_use_id, agent_id, subagent_type, model, description, dispatch_site (regex-extracted from prompt's first line), status, prompt_chars, prompt_first_line (truncated to 200 chars), duration_ms, total_tokens, input_tokens, output_tokens, cache_creation_tokens, cache_read_tokens, tool_uses_in_subagent, tool_stats: {bash, edit, read, search, other, lines_added, lines_removed}, result_chars, branch, cwd. Smoke-tested with a 3-dispatch fixture: produces correct records, advances cursor (6 lines → 8 lines after appending two more transcript lines), idempotent on re-run.
- **`docs/design/telemetry-signals.md`:** removed v2.2.4's `dispatch_models` field from the per-turn record schema; added new `## Subagent dispatch records (v2.3.0+)` section with full record shape + field semantics; added new `## Subagent dispatch jq recipes (v2.3.0+)` section with 6 jq cookbook recipes (top-N most expensive single dispatches, per-subagent_type aggregates, per-dispatch-site aggregates, per-model breakdown by site for §Agent dispatch contract verification, anomaly detection at >2σ, cost trend over 14 days).
- **`docs/internals.md`:** doctor table extended to 19 rows (#19 orphan subagents file); #12 entry extended; Step D parallelization brief count updated 18→19.
- **`.claude-plugin/plugin.json`** + **`marketplace.json`:** version 2.2.3 → 2.3.0.
- **`README.md`:** Current-release line bumped to v2.3.0.
- **`CHANGELOG.md`:** single `[2.3.0]` block describing both threads — added/fixed/migration-notes/verification per Keep a Changelog.

**Key decisions (the why):**

- **One release, not two.** v2.2.4 was prepared but never published (user picked "Don't commit — review first" at the prior plan's exit). Folding the per-subagent telemetry work into v2.3.0 collapses what would have been two consecutive releases into one minor bump. CHANGELOG narrative covers both threads under v2.3.0.
- **Deprecate `dispatch_models` (v2.2.4 was prepared but never published).** The new `<plan>-subagents.jsonl` is more granular and authoritative than the per-turn aggregate counter. Single source of truth. Nobody had built on `dispatch_models` yet (it never shipped). The contract section's "Telemetry counter" paragraph is replaced with a "Telemetry capture" paragraph pointing at the new file.
- **Central DISPATCH-SITE tag table over 14 inline brief edits.** The contract section is already the single source of truth for dispatch decisions. Adding the per-step tag values to that section's table avoids duplicating the requirement at every dispatch site. Risk: orchestrator might miss the convention at some sites; mitigation: smoke test reveals null `dispatch_site` records, and we can add per-site reminders later if usage data shows drift.
- **Cursor-based incremental parsing.** A long-running session's transcript can grow to 80 MB. Re-parsing the entire file every Stop turn is wasteful. The cursor file (`<slug>-subagents-cursor`) stores the last-processed line count; subsequent turns only emit records for new toolUseResult lines beyond the cursor. The tool_use index still requires scanning the entire file (a tool_use earlier than cursor can pair with a toolUseResult after cursor for long-running subagents), but jq slurps that fast.
- **Recursive override at SDD invocation, not wrap-SDD-in-outer-Agent.** SDD runs as a skill in the orchestrator's context; wrapping it in an outer Agent dispatch creates an additional indirection layer for marginal benefit. The override clause asks SDD to add `model: "sonnet"` to its inner Task calls — this is the cheapest mitigation that doesn't depend on modifying upstream `obra/superpowers`. Risk-mitigation branch documented: if SDD's template structure ignores the override on a future upstream change, fall back to wrapping.
- **Minor version bump (2.2.3 → 2.3.0).** Substantial new capability (per-subagent telemetry) + behavioral change (model: now structurally required) + new file `<plan>-subagents.jsonl` + new doctor check #19. Minor fits per semver. No status frontmatter change, no config schema change.

**Verification:**

- 10 grep discriminators per the v2.3.0 plan: contract section landed once; ≥14 `model: "..."` parameters in `commands/masterplan.md`; ≥2 Codex-exempt notes; opus-on-blocker structural wire-up; `<plan>-subagents.jsonl` referenced in hook; ≥14 DISPATCH-SITE values in the contract table; doctor table 19 rows + Step D brief at "all 19 checks"; `dispatch_models` references = 0 (deprecated); subagent dispatch schema + 6 jq recipes in `docs/design/telemetry-signals.md`; version 2.3.0 in CHANGELOG/README/plugin.json/marketplace.json.
- `claude plugin validate .` — clean.
- `bash -n hooks/masterplan-telemetry.sh` — clean.
- **First runtime smoke test for the project:** hand-crafted JSONL fixture with three Agent dispatches (haiku Explore, sonnet general-purpose, codex:codex-rescue) verifies the hook emits exactly 3 records with correct model/duration/tokens/dispatch_site, cursor advances to 6, idempotent on re-run, and 4th record only emerges after appending a new dispatch to the transcript (cursor advances to 8). Documented in §13 recipe pattern.

**Verification gaps (carried as v2.3.x followups):**

- **First-user smoke not yet performed against a real plan.** Markdown-only project; the orchestrator IS the prompt. The `<plan>-subagents.jsonl` file on the first user-driven `/masterplan execute` will confirm whether the contract + DISPATCH-SITE tagging fire correctly. Acceptance criteria: ratio of `opus_tokens / total_tokens` < 0.1 over ≥ 5 turns; `dispatch_site` populated on ≥ 90% of records (null only for non-/masterplan dispatches that bypassed the contract).
- **SDD override clause may not propagate.** If `superpowers:subagent-driven-development` upstream evolves in a way that ignores or overrides the orchestrator's brief at sub-step boundaries, the override has no effect. Mitigation: smoke-verify on first user run via per-model breakdown by site (recipe #4); fall back to wrapping SDD in an outer `Agent(subagent_type: "general-purpose", model: "sonnet", ...)` if needed.
- **Hard-pinned short names** (`"haiku"` / `"sonnet"` / `"opus"`). Confirmed valid by plugin-dev SKILL docs at v2.3.0 release time. If Anthropic deprecates these aliases, the contract section is the single place to update.
- **macOS hook smoke.** Linux-only smoke verified. Hook is portable-by-construction (no GNU-only flags; uses `wc -l`, `cat`, `awk`, `jq`, portable `stat -c '%Y' || stat -f '%m'`); deferred until a macOS env is available.
- **JSONL schema fragility.** The hook depends on the exact shape of `toolUseResult.usage`, `agentId`, `agentType`, etc. If Claude Code's transcript format changes upstream, the hook breaks. The smoke-test fixture is the canary — re-run after any upstream Claude Code update.
- All v2.2.0 / v2.2.1 / v2.2.2 / v2.2.3 followups still apply.

**Known followups (post-v2.3.0):**

- **Doctor check candidate** — surface `<plan>-subagents.jsonl` records with `dispatch_site == null` count > N as a Warning (DISPATCH-SITE tag drift detector). Niche; defer until usage data shows the central-table approach is or isn't sufficient.
- **Cross-plan cost dashboard.** Each plan's `<plan>-subagents.jsonl` is per-plan. A cross-plan aggregator (e.g., `/masterplan status --cost`) would let the user compare token spend across in-flight plans. Defer; depends on first-user runtime data.
- **Per-token cost calculation.** Cookbook recipes report token COUNTS; converting to dollars requires a rate-card mapping that drifts with Anthropic's pricing. Defer until cost reporting is a clearer requirement; tokens are a reliable proxy.
- **PreToolUse hook for real-time Agent capture.** Stop-hook-based capture is at end-of-turn. Real-time would require PreToolUse + PostToolUse hooks per Agent dispatch. End-of-turn is fine for analysis; defer.
- **Auto-archive when plan completes.** When `status: complete`, the subagents file could move to `<archive_path>/<date>/<slug>-subagents-final.jsonl`. Niche; defer.
- All v2.2.x followups still apply (SessionStart hook redundancy, blocker-gate option-pick telemetry, etc.).

**Branch state at end of v2.3.0:**

- 1 commit ahead of v2.2.3 on `main` (this release commit, folding the prepared-but-unpublished v2.2.4 work into v2.3.0).
- Tag `v2.3.0` to be created locally; pushed alongside the commit per release convention.
- Working tree clean post-commit.
- plugin.json: 2.3.0. marketplace.json: 2.3.0 (both nested + top-level).

## 2026-05-07 - v2.13.1 - marketplace install self-healing

**Scope:** `/masterplan` slash command vanished after marketplace install despite plugin content being correct and at v2.13.0.

**Root cause:** Claude Code's marketplace installer deploys `commands/masterplan.md` to `~/.claude/plugins/marketplaces/rasatpetabit-superpowers-masterplan/commands/` but slash-command discovery only scans `~/.claude/commands/`. The installer backed up the prior direct-install copy to `masterplan.md.bak-pre-v2.9.0-*` and did not create a replacement link. Plugin content and hook were untouched and correct throughout.

**Why this fix:** Created `~/.claude/commands/masterplan.md` as a symlink to the marketplace path (immediate fix, survives in-place upgrades). Added `hooks/hooks.json` (mirrors `obra/superpowers` convention) with a SessionStart hook that silently recreates the symlink if missing or dangling — making the install self-healing for future reinstalls.

## 2026-05-07 - v2.14.0 - issues #1 and #3 closed

**Scope:** Two GitHub issues fixed in one release.

**#3 (Step I1 false negative on remote-only branches).** Root cause: the source class 2 brief said "`git branch -avv`, then filter…" — Haiku occasionally downgraded that to `git branch -v` (or to local-only iteration) and silently missed remote refs. Replaced with explicit `git for-each-ref refs/heads/ refs/remotes/ --format='%(refname:short)'` and clarified that the check is topology-based (SHA reachability) so rebased-equivalent branches are still flagged. Reproducer: petabit-os-mgmt origin/phase-5-southbound-ipc.

**#1 (`doctor --fix` near-no-op).** Three separate gaps closed: (1) `--fix` extends to checks #20/#21 — eligibility cache rebuild, deterministic from plan annotations, mirrors Step C step 1's Build path. (2) check #1 gains sub-classification #1a — stray-duplicate-orphan (cross-worktree dedup), `--fix` runs `git rm`. (3) Output gets a top-line "0 of N findings match the auto-fix action set" warning when `--fix` ran but produced no file changes — closes the historical UX failure where the buried "0 files changed/moved" line made `--fix` look broken.

**Why these fixes specifically:** Each addresses an observed real-world failure (petabit-os-mgmt, optoe-ng) where existing behavior produced false negatives or false UX. Each is bounded and deterministic — no judgment calls on the orchestrator's part.

**Fresh-eyes review caught:** phantom `**Tests:**` annotation reference in the #20 --fix description (Section 2 eligibility rules only mention `**Codex:**` + `**Files:**`); removed before commit. Also added `(v2.14.0+)` version marker to #1a sub-classification per project's existing version-tag convention.

## 2026-05-07 - v2.14.1 - Step I1 brief tightening (symbolic-HEAD short-form gotcha)

**Scope:** Smoke-test of v2.14.0 against `petabit-os-mgmt` exposed a residual ambiguity in the Step I1 source class 2 brief. Original issue #3 reproducer (`origin/phase-5-southbound-ipc`) was already deleted from the remote, so end-to-end test had to be against current state.

**Finding:** A Haiku dispatched with the v2.14.0 brief self-reported: "the brief says to exclude 'HEAD' but doesn't specify whether to filter on the literal substring 'HEAD' in refname:short output (doesn't appear here), [or] the `refs/remotes/<remote>/HEAD` symbolic ref nature." Root cause: `git for-each-ref --format='%(refname:short)'` renders `refs/remotes/origin/HEAD` as the bare token `origin` — NOT caught by `grep -v HEAD`. The Haiku guessed right (interpretation), but a worse-luck run could pass it through.

**Fix:** Brief now uses `--format='%(refname)|%(refname:short)'` and instructs the agent to filter on the full refname (drop lines whose full path ends in `/HEAD`) and use the short name for display + topology + `gh` cross-reference. Validated with a second Haiku dispatch on the same repo — clean, explicit drop, no "I had to guess" language.

**Lesson:** the v2.14.0 release tested against the exact reproducer in the issue, but not against a fresh Haiku-running-the-new-brief end-to-end. End-to-end test caught a residual ambiguity that the unit-level edit review missed. Worth folding into the project's CD-9-style test rule: after orchestrator-prompt edits, smoke test with a Haiku running the updated brief against a real repo before declaring done.

## 2026-05-08 - fix: sentinel vUNKNOWN in non-dev-checkout repos

**Scope:** Two-line prompt fix to `commands/masterplan.md`.

**Root cause:** The sentinel's plugin.json path resolution said "resolve via `dirname(dirname(<this-prompt's-path>))`" but the model has no literal access to the prompt file path. In the dev checkout (`/home/ras/dev/superpowers-masterplan`), it happened to work because CWD contained `.claude-plugin/plugin.json`. In any other project (e.g., petabit-os), the CWD has no `.claude-plugin/` directory so the Read fails and the sentinel falls back to `vUNKNOWN`.

**Fix:** Both the sentinel and the §Stats `plugin-root` resolution now list three concrete candidate paths to try in order: (1) installed marketplace path `~/.claude/plugins/marketplaces/rasatpetabit-superpowers-masterplan/.claude-plugin/plugin.json`, (2) `<cwd>/.claude-plugin/plugin.json` for dev-checkout, (3) highest-semver cache path. First readable path wins; only falls back to `vUNKNOWN` if all three fail.

**Confirmed by:** doctor --fix run showed clean repo (6 archived plans, all artifacts present). Fix was observed after user reported `vUNKNOWN` in petabit-os session.

## 2026-05-09 - Codex masterplan prompt exposure bridge

**Scope:** New Codex sessions could not call or recognize masterplan even though the Codex marketplace was registered.

**Root cause:** The prior packaging check proved static manifests and marketplace registration, not prompt exposure. `codex debug prompt-input` showed no `superpowers-masterplan` or `masterplan-detect` content; `~/.codex/config.toml` registered `[marketplaces.rasatpetabit-superpowers-masterplan]` but had no enabled plugin entry, and no plugin cache existed under `~/.codex/plugins/cache/`. The package also only shipped `masterplan-detect`, not a Codex-native `masterplan` skill.

**Fix:** Added `skills/masterplan/SKILL.md` as the Codex-visible entrypoint that loads `commands/masterplan.md`, maps `/masterplan` and natural-language masterplan requests, and requires scanning existing Claude-created `docs/masterplan/*/state.yml` bundles before starting fresh work. Installed the same bridge at `~/.codex/skills/masterplan/SKILL.md` and enabled `[plugins."superpowers-masterplan@rasatpetabit-superpowers-masterplan"]` locally for new sessions.

**Confirmed by:** `codex debug prompt-input 'Run /masterplan status'` from `/home/ras/dev/meta-petabit` now includes `- masterplan: ... (file: r0/masterplan/SKILL.md)`. Repo checks passed: `bin/masterplan-self-host-audit.sh`, `bin/masterplan-self-host-audit.sh --codex`, `jq empty .claude-plugin/plugin.json .codex-plugin/plugin.json .agents/plugins/marketplace.json`, `git diff --check`, and `bash -n` for all shell entrypoints.

## 2026-05-15 - fix: Codex-degraded false positive (working branch, deferred release)

**Scope:** Branch `fix/codex-degraded-false-positive` off main. Five files touched: `commands/masterplan.md` (CC-2 Step 3), `parts/doctor.md` (Check #39 table row + impl, Check #41 `auth_healthy` probe), `lib/masterplan_session_audit.py` (watcher retirement), `bin/masterplan-findings-to-issues.sh` (hard-codes CSV), `docs/internals.md` (Codex-routing-visibility family note).

**Root cause:** `/masterplan v5.2.x` boot banner and doctor Check #39/#41 read `.id_token`/`.access_token` at the top level of `~/.codex/auth.json`, but the actual schema nests them at `.tokens.*`. Even with the bash bug, the user-visible degraded line still fired because the Read tool surfaces the JSON to the LLM, which then inferred and reported the expiry. The *deeper* misjudgment: under `auth_mode=chatgpt` the JWTs are short-lived (~1h) and auto-refreshed via the persistent `refresh_token` on every Codex call, so a past-due `exp` is steady-state, not degradation. The v5.2.1 policy-regression watcher `codex_health_check_jwt_only` (`lib/masterplan_session_audit.py`) had been silently logging this false-positive shape since release.

**Fix:** Both the boot-banner Step 3 logic and Doctor #39 now early-exit silently when `auth_mode == "chatgpt"` AND `tokens.refresh_token` is present AND `last_refresh` is within 7 days — JWT `exp` arithmetic is skipped entirely. Sub-fire (c) (`last_refresh > 30d`) remains active in #39 since stale refresh is real trouble even in chatgpt mode. jq paths fixed to `.tokens.$field // .$field // empty` (both new and old schemas accepted). Check #41's `auth_healthy` probe got the same fixes so its sub-fire (a) doesn't misjudge chatgpt-healthy auth as unhealthy. The `codex_health_check_jwt_only` watcher was retired (function deleted, registry entry removed, `WarningItem` emission dropped, `findings-to-issues.sh` CSV trimmed) — the user-visible boot banner is the regression detector going forward.

**Confirmed by:** Extracted Check #39 bash run against live `~/.codex/auth.json` reports `Check #39: PASS (auth_mode=chatgpt; JWT auto-refresh healthy; last_refresh 0d ago)`. Extracted Step 3 bash run against same file emits the silent cosmetic-skip path. `python3 -m py_compile lib/masterplan_session_audit.py` + `bash -n bin/masterplan-findings-to-issues.sh` clean. Router byte size 11233/20480.

**Out of scope by user choice:** no version bump, no CHANGELOG entry, no marketplace push, no tag. Release-shape decision deferred — branch is the deliverable.

## 2026-05-16 - v5.7.1→v5.7.2: doctor fixes, bundle cleanup, epyc1 mystery resolved, grojas setup

**Scope:** Multi-part session. Doctor check #41 false-positive fix (v5.7.1 released earlier), then full regression pass, then doctor cleanup and v5.7.2 release.

**Key decisions:**
- Check #41(a) false-positive: gated on `codex_ever_active` — zero codex_ping/degraded/routing events means Codex was intentionally off from bundle creation; no degrade-loudly evidence expected. Both `concurrency-guards` and `p4-suppression-smoke` were false-firing because they had codex_routing=off with zero codex events.
- State field normalization: `phase: complete`/`ready_for_retro` and `status: complete` → `completed`/`archived` per run-bundle schema (doctor check #9 WARNs). Pre-existing values were written by older orchestrator code before canonical values were standardized.
- Bundle cleanup: `p4-suppression-smoke` stale .lock removed, anomalies.jsonl deleted (plan complete + retro done). `concurrency-guards` archived (work was already shipped in v5.7.0; state.yml just hadn't been transitioned).

**epyc1 mystery:** ras@epyc1 was at v3.2.9 (not v5.7.x as WORKLOG claimed). Root cause: grojas@epyc1 IS a real user and had the plugin at 5.7.0 (independent install), but WORKLOG entries saying "grojas@epyc1 rollout success" were tracking grojas not ras. ras@epyc1 was stuck at 3.2.9 from initial install, never updated. Both now at 5.7.2.

**grojas@epyc1 setup:** Claude Code was NOT installed for grojas; installed via `npm install -g --prefix ~/.npm-global @anthropic-ai/claude-code`. PATH added to ~/.bashrc + ~/.profile. Plugin at 5.7.2.

**v5.7.2 release:** Maintenance commits (state normalization + archive) bundled as a patch release. Tagged, pushed, deployed to ras@epyc2 + ras@epyc1 + grojas@epyc1.

## 2026-05-16 — v5.7.3: fix parent_turn duplication (telemetry audit)

**Scope:** 8-hour telemetry + transcript audit (Opus agent). Filed GH issues #8 and #9. Shipped Fix 2 as v5.7.3.

**Root cause (Bug #8):** `emit_parent_turns()` in the Stop hook rescanned the full transcript on every Stop fire with no seen-set → ~2× parent_turn inflation. Fix: build seen-set keyed by `ts|session_id` from existing subagents.jsonl before scanning; filter jq output against it. Mirrors the existing `agent_id` dedup in `_do_append_subagents()`. Commit: 501e9ec.

**Bug #9 (routing_class:codex not recorded) — open:** 0 codex routing_class records despite a confirmed codex dispatch. Hypothesis: codex results may lack `toolUseResult.agentId`, which the filter at line 426 requires. Root cause needs transcript-level confirmation before a fix can be written. Held per advisor guidance.

**Bug #1 (slug misattribution) — invalidated:** all p4-suppression-smoke records have cwd=/home/ras/dev/superpowers-masterplan; these are sessions in this repo dispatching subagents to yanos work. Working as designed.
