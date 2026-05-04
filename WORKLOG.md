# WORKLOG

Append-only handoff notes for collaboration with other LLMs (Codex, future Claude sessions). Read at the start of substantive work; append a brief dated entry before ending substantive work. Diff shows _what_; this captures _why_.

Pre-v2.0.0 entries were pruned in the v2.0.0 release; institutional knowledge from those entries was migrated into `docs/internals.md` (deep-dive: architecture, dispatch model, failure modes, design history, recipes, anti-patterns). Per-version release detail lives in `CHANGELOG.md` (preserved verbatim).

---

## 2026-05-04 — v2.2.0 — doc revisionism + verb rename + no-args picker

**Scope:** Three threads:

1. **Doc revisionism.** Pre-v1.0.0 (v0.x) release-history references removed everywhere. CHANGELOG older blocks deleted entirely + remaining v0.x mentions in v1.0.0/v2.0.0 entries scrubbed; v2.0.0 entry's rename framing dropped (Renamed section + rename-step migration notes deleted; mechanical `/superflow → /masterplan` and `claude-superflow → superpowers-masterplan` substitutions throughout v1.0.0 + v2.0.0 entries). README "Path to v2.0.0" → "Releases since v1.0.0" with v0.x bullets removed; Project status reworded to drop rename framing; `/superflow` alias non-feature item removed from Roadmap. `docs/internals.md` v0.x parentheticals dropped from "Why" section headings; "Why hard-cut renames" → "Why hard-cut name changes"; remaining "rename"/"renamed" mentions reworded throughout. `docs/design/intra-plan-parallelism.md` + the v1.1.0 spec: "v0.1 → v0.2 → v0.3 → v0.4 → v1.0.0" deferral-chain framing rewritten as "deferred prior to v1.0.0". WORKLOG v2.0.0 entry's rename narrative trimmed (was 6 threads → now 5); functional deliverables (parallelism Slice α, Codex defaults, internal docs) preserved.
2. **Verb rename `new` → `full`.** Hard-cut, no alias. All sync'd locations updated: frontmatter description, Step 0 verb routing table rows ("full" no-topic + "full <topic>"), reserved-verbs warning, argument-parse precedence list, Step P "no candidates" example, README verb table + quick-start examples + reserved-verb prose + Aliases-and-shortcuts table, `docs/internals.md` Step 0 mirror.
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
- **No new doctor check #19.** M0's tripwire reuses existing checks #2/#3/#4/#5/#6/#9/#10 by name + semantics. Doctor table size stays at 18; Step D's parallelization brief still says "all 18 checks." Per CLAUDE.md anti-pattern #5, the three sync'd verb-routing locations remain aligned (M0 is a sub-step of M, not a new verb — frontmatter `description:` and reserved-verbs warning unchanged).

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
