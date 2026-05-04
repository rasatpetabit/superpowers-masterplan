# WORKLOG

Append-only handoff notes for collaboration with other LLMs (Codex, future Claude sessions). Read at the start of substantive work; append a brief dated entry before ending substantive work. Diff shows _what_; this captures _why_.

---

## 2026-05-01 — `/superflow` v0.2.0 small-fixes pass (`feat/superflow-small-fixes`)

**Scope:** Bundled six findings from a `/superflow` analysis session into one improvement pass. Spec: `docs/superpowers/specs/2026-05-01-superflow-small-fixes-design.md`. Plan: `docs/superpowers/plans/2026-05-01-superflow-small-fixes.md` (8 tasks). Status: `docs/superpowers/plans/2026-05-01-superflow-small-fixes-status.md`. All 8 tasks complete; v0.2.0 released.

**Key decisions (the why):**

- **Bundled six small fixes into one pass instead of one per finding.** Each fix is independently small and several share the same files (`commands/superflow.md`, `README.md`, `CHANGELOG.md`). Six separate spec/plan/execute cycles would have been pure overhead. Six-in-one preserves a coherent "v0.2.0" release rather than a stream of patch versions. The advisor explicitly recommended this framing during analysis ("present the findings, don't brainstorm" — then user picked the bundled-spec option).
- **Three larger threads were deliberately deferred to dedicated specs:** SDD × Codex routing per-task loop boundary (analysis finding #3), 4-review pile-up under default + codex-review (#5), intra-plan task parallelism (#12). The boundary one is the highest-impact orchestration ambiguity; tackle it before broadening Codex use further.
- **`codex_routing: off` for THIS execution run** — the SDD × Codex boundary is unresolved at the time of this run, and this pass doesn't fix it. Setting `off` sidesteps the ambiguity. Future plans (after the boundary is resolved) can use `auto`. The plan's per-task `**Codex:** ok|no` annotations are valid documentation regardless — they'll take effect once the boundary is settled.
- **Inline execution instead of subagent dispatch** — plan tasks are mechanical text edits with explicit Edit + grep verification. Spinning up 8 subagents would re-load the same context per task. The "Subagents do the work" pillar applies to LONG runs where orchestrator context bloats; this pass fits comfortably in one session.
- **Behavior change made the default rather than opt-in.** The user's permissiveness ask drove this: default `gated` mode no longer prompts on pre-configured Codex automation. Users who want the old chatty behavior set `codex.confirm_auto_routing: true` and `codex.review_prompt_at: "low"`. Documented as a behavior change in CHANGELOG `[0.2.0]` Changed.
- **Step 4b's SHA fallback bug was real**, not theoretical. Verified empirically: `git merge-base HEAD master` returns the HEAD SHA when on master tip. Fix removes the fallback entirely; `task_start_sha` is now required in implementer return digest, blocks recoverably if missing.

**Operational lessons (worth keeping in mind):**

- Multiple Edits to the same file in one session work fine sequentially — the Edit tool's "must read before write" check holds within a session. But moving across worktrees (e.g., the .gitignore on main vs. files in the worktree) requires re-reading per worktree.
- The advisor (when applicable) is especially good at re-framing: it caught that the user wanted "the analysis as deliverable" rather than "let's brainstorm together," which would have wasted an hour of one-question-at-a-time refinement.
- Gated checkpoints between tasks in a small pass like this are noise — the user said "go" once and that was standing approval. Long autonomous runs (`/loop`) need different tradeoffs.
- `git status --porcelain` is correctly never cached in `git_state` (per CD-2). Confirmed in this pass: every Step C entry that reads dirty state goes live.

**Open questions / followups:**

- The SDD × Codex boundary (analysis finding #3) needs its own spec. Without it, `codex_routing: auto` under SDD has undefined semantics — superflow inlines tasks via SDD, but Codex routing is per-task and superflow-decided in Step C 3a, with no documented mechanism for the orchestrator to intercept before SDD dispatches.
- `superpowers:writing-plans` skill upstream doesn't know about `**Codex:** ok|no` annotations. We documented the convention in superflow's Step B2 brief — plans authored via `/superflow` will get annotations. Plans authored elsewhere won't. Worth proposing an upstream PR to writing-plans at some point.
- Telemetry per-task model usage (analysis finding #11) wasn't included in this pass. Small but isolated change; would inform tuning of `codex.max_files_for_auto` and the eligibility heuristic.
- `finishing-a-development-branch` is mandatorily interactive even under `--autonomy=full` (analysis finding #10). Not fixed in this pass.

**Branch state at end of pass:**

- 11 commits ahead of `main` on `feat/superflow-small-fixes`.
- Linear history, no merge commits, no rebase needed.
- `.worktrees/` ignored on main; .worktrees/superflow-small-fixes is the active worktree.
- Suggested next: invoke `superpowers:finishing-a-development-branch` to merge to main or open a PR.

---

## 2026-05-02 — `/superflow` v0.3.0 explicit phase verbs

**Scope:** Added `new`, `brainstorm`, `plan`, `execute` as explicit first-token verbs in `/superflow`. Spec: `docs/superpowers/specs/2026-05-02-superflow-subcommands-design.md`. Plan: `docs/superpowers/plans/2026-05-02-superflow-subcommands.md`.

**Key decisions (the why):**

- **Discoverability over phase-control framing.** User picked "Discoverability — make verbs visible at a glance" as the motivation. The phase-control verbs (`brainstorm`, `plan`) fall out for free once the verbs are addressable, but they aren't the headline.
- **Additive, no deprecation.** Bare-topic catch-all and `--resume=<path>` keep working forever. Existing `/loop /superflow <topic> ...` invocations and any cron / docs that use the bare-topic form continue unchanged. Cost: routing logic remains "verb match OR catch-all."
- **`halt_mode` as a tiny internal state machine instead of a per-step flag.** Set once in Step 0 from the verb match, read by B1/B2/B3/C. Cleaner than threading four boolean flags through every dispatch site, and the in-session "Continue to plan now / Start execution now" overrides become a simple `halt_mode` flip.
- **`plan --from-spec=<path>` skips Step B0 — spec's location is authoritative.** B0a covers the trunk-branch foot-gun (relocate spec to a feature worktree if it lives on main/master/trunk). Caught during spec self-review; without it, we'd silently inherit the trunk branch and only discover SDD's refusal at execute time.
- **`/superflow plan` (no args) does a Step P picker, not an error.** User flagged "list recent specs without a plan, let user pick" as the desired behavior. One filesystem scan beats forcing the user to remember/type the path; consistent with how `/superflow` (empty) lists in-progress plans.
- **Verb tokens reserved.** Topics literally named `new`, `brainstorm`, `plan`, `execute` need a leading word. Documented in the README. Concrete cost is small; alternatives (escape character, `--topic=` flag) would have introduced more grammar than they saved.

**Operational notes:**

- All edits are markdown-only. No code, no automated test suite for the prompt. Verification is grep-based per-task plus a final smoke-read of the modified `commands/superflow.md`.
- The README's existing `## Subcommand reference` got a new `### Verbs` subsection at its top; the original table is now `### Invocation forms (back-compat detail)`. README structure preserved otherwise.
- v0.3.0 is a minor version bump because the externally-visible grammar grew (new verbs) without breaking anything that already worked.

**Open questions / followups:**

- The `/loop /superflow brainstorm <topic>` foot-gun is mitigated by a one-line warning at Step 0; a stricter "auto-disable loop under halt_mode" could be considered later if telemetry shows users still hit it.
- `/superflow execute` with zero in-progress plans currently routes to Step A which offers "Start fresh" → kickoff. That's slightly indirect under explicit-verb framing; a future polish could reword the option to "No in-progress plans. Run /superflow new <topic>?" so the verb model stays coherent.

---

## 2026-05-03 — README + plugin.json catch-up to v0.3.0

**Scope:** Doc-only follow-up to v0.3.0 — the verbs landed in `commands/superflow.md` and CHANGELOG but `README.md`'s top-level "What you get" / Quick start / Project status / Recent improvements still framed `/superflow` as topic-only, and `.claude-plugin/plugin.json` was still pinned at `0.2.2`.

**Key decisions (the why):**

- **Two-tier "What you get" list.** Split into "Phase verbs" (the new v0.3.0 surface) on top, then "Other subcommands" below. Front-loads the discoverability win and keeps the established invocations findable. The README's existing `## Subcommand reference` already has the canonical Verbs table — this section is the elevator pitch.
- **Quick start gained a "Brainstorm or plan only" example.** The brainstorm-only and plan-only flows are the headline behavior change; without an example they'd only surface in the reference table. Added the `/loop /superflow brainstorm` foot-gun callout inline (matches the orchestrator's Step 0 warning) so readers see it before they hit it.
- **Resume example shows all three forms.** `/superflow execute <path>`, `--resume=<path>`, and the bare-`/superflow` listing — explicit-verb form first, back-compat alias second. Mirrors the README's broader policy of "verb form headline, alias detail."
- **Project status rewritten, not patched.** The previous "v0.2 release (current: v0.2.2)" paragraph carried four versions of release notes that were already in the CHANGELOG. Replaced with a one-paragraph v0.3 framing; consolidated the v0.2.x bullets into the Recent improvements section so the CHANGELOG stays the canonical detail.
- **Recent improvements compressed v0.2 down to two bullets and added v0.3.0.** The original two-bullet structure (speed, context use) was a v0.2.0-only summary that had grown stale once .1/.2 landed. Re-grouped by version (v0.2.0 / v0.2.1+v0.2.2 / v0.3.0) so each release reads as one unit.
- **plugin.json description tweaked, not just version-bumped.** Added "with explicit phase verbs (new / brainstorm / plan / execute)" so plugin browsers reading the description see the v0.3.0 surface without having to open the README.

**Operational notes:**

- All edits are markdown / JSON. No code changes. Verification was grep-based (`v0.` references in README, version pin in plugin.json) plus a smoke-read of changed sections.
- Skills (`superflow-detect`, `superflow-retro`) and design docs deliberately untouched — the detect skill only ever suggests `/superflow import` (correct), and retro doesn't reference verbs at all.

**Open questions / followups:**

- None this pass. The CHANGELOG-as-canonical-detail / README-as-elevator-pitch split is the convention going forward; future minor releases should follow the same pattern (compress prior version into Recent improvements, rewrite Project status paragraph).

---

## 2026-05-03 — v0.4.0 — `retro` verb + terminology cleanup (same day)

**Scope:** Same-day follow-up to the v0.3.0 doc pass. Two user requests, addressed together: (1) "clean up superflow-retro, just make retro a verb instead of a separate command"; (2) "look for other consistency issues like that." Tagged v0.4.0 because the verb surface grew (new verb token reserved) and a shipped skill was removed — externally visible.

**Key decisions (the why):**

- **Retro skill deleted entirely instead of being shrunk to a suggester.** User picked the "Delete skill entirely" option from a 3-option fork (delete / shrink-to-suggester-parallel-to-detect / keep-both). Reasoning the user agreed with: one trigger surface beats two. Auto-fire-on-completion was clever but discoverable-via-verb-table is more legible; `superflow-detect` stays because the artifacts it suggests on (PLAN.md / orphan plans / draft PRs) are not invocations the user would otherwise know about, whereas `/superflow retro` is now in the verb table and surfaces in `/superflow status`.
- **Step R port preserves the skill's structure verbatim.** R0 (resolve target) → R1 (pre-write guard) → R2 (gather, parallel) → R3 (synthesize + write) → R4 (offer follow-ups). The skill body's section template (Outcomes / Timeline / What went well / What blocked / Deviations / Codex routing observations / Follow-ups / Lessons) is reproduced inline in R3. Future readers don't need to know there used to be a skill.
- **Pre-write guard from the skill (`*-<slug>-retro.md` glob) carried over to R1.** This was the bugfix in v0.2.0 that prevented silent duplicate retros. Re-implementing it as part of R1 instead of relying on R3-time discovery keeps the user's intent visible as a named step.
- **Terminology standardization stayed minimal.** User picked only "Standardize terminology on 'verbs'" from a 4-option multi-select. The other three (split verbs table by phase/op, drop back-compat framing for bare-topic, sample-topic consistency) deferred — three separate decisions for another pass. Concretely landed: `## Subcommand reference` → `## Verb reference`; `### Subcommand routing` → `### Verb routing`; "Other subcommands" → "Operation verbs"; `### Invocation forms (back-compat detail)` → `### Aliases and shortcuts` (the back-compat label was misleading — these are documented aliases, not legacy forms).
- **Verbs table column rename ("Phases" → "Effect").** The original "Phases" column had `(unchanged)` placeholders for import/doctor/status/retro because they're not pipeline phases. Renaming to "Effect" + filling each row with a one-line description is honest about what each verb does. (This is technically beyond the user's "standardize on verbs" scope but it falls out of having to add `retro [<slug>]` to the table — the row needed an effect description, and `(unchanged)` for the others would have been jarringly inconsistent.)
- **Reserved-verb list in Step 0 expanded to all 8 verbs.** Pre-existing bug: the warning listed only the four phase verbs but the routing matched on 7 (now 8). Fixed because the warning should reflect what's actually consumed.
- **Pre-existing `## Step D — Doctor` header restoration.** The doctor section had been silently un-headed for at least two releases — section started with `### Scope` directly after Step S4. Restored as part of inserting `## Step R` between S and D. Tracked in CHANGELOG `[0.4.0] Fixed`.
- **Plugin description tweaked to enumerate all 8 verbs**, not just the 4 phase verbs from v0.3.0. Plugin browsers reading the description see the full surface.

**Operational notes:**

- Used `git rm -r skills/superflow-retro/` so the deletion is part of the staged set; no untracked-file ambiguity at commit time.
- Two same-day WORKLOG entries (v0.3.0 doc pass + v0.4.0 verb consolidation). Kept separate because they were independent decisions made at different points in the same conversation; merging would lose the chronology.
- The Codex adversarial review of the v0.3.0 doc pass returned `verdict: approve, no findings`. Triggered between the two passes, so it does not cover the v0.4.0 changes.

**Open questions / followups:**

- The three deferred consistency items (split verbs table by phase/op, drop back-compat framing for bare-topic, sample-topic consistency) are small enough to bundle into a future doc-only pass if other small items accumulate.
- `commands/superflow.md`'s top-of-file `description:` is the canonical source for the `/superflow` verb list shown in autocomplete. Any future verb addition needs to update three places: routing table at Step 0 line ~46, reserved-verbs warning at line 70, and the description frontmatter line. Worth a doctor check at some point.
- No regression test for the verb routing — the orchestrator prompt is markdown, not code. Future addition: a `docs/superpowers/specs/` self-test that exercises every verb's branch via canned `$ARGUMENTS` strings would catch routing-table drift.

---

## 2026-05-03 — v1.0.0 — pre-release audit fix pass + first stable release (same day)

**Scope:** Pre-public-release audit of the v0.4.0 in-flight diff (uncommitted retro-verb consolidation + terminology cleanup). Three parallel fresh-eyes Explore agents audited (1) `commands/superflow.md`, (2) human-facing docs (README, CHANGELOG, design docs, historical specs/plans), (3) `hooks/superflow-telemetry.sh` + `skills/superflow-detect/SKILL.md` + `.claude-plugin/plugin.json`. Findings: 10 blockers + 14 polish items + ~24 informational notes. Applied 10 blocker fixes (B1–B10) and 13 polish items (P1–P14, P15 deferred). User chose to ship the bundled v0.4.0 + audit fixes as **v1.0.0** — the first stable public release — in one push, rather than tag v0.4.0 separately.

**Key decisions (the why):**

- **Bumped to v1.0.0 instead of v0.4.0 / v0.4.1.** User picked "Change version to v1.0.0 in one push" from the bundling fork. Reasoning: the orchestration logic has been stable in real Petabit Scale workflows since v0.1; the audit fixes were the last things blocking a "wide-audience-public-release-ready" framing. Renaming the staged `[0.4.0]` CHANGELOG block to `[1.0.0]` and extending it with the audit fixes is cleaner than tagging v0.4.0 separately + a v0.4.1 patch — public history shows one release commit, one date, one canonical changelog block.
- **B6 fix dropped option 3 specifically (not 1, 2, or 4).** The blocker re-engagement gate had 5 options; CD-9 caps at 2–4. Option 5 ("Set status: blocked and end the turn") MUST stay because resume-from-blocker depends on it being the only path to the legacy `status: blocked` end-turn state. Of the four "keep moving" options, option 3 ("Break this task into smaller pieces — pause so I can edit the plan to decompose, then continue") overlapped semantically with option 1 ("Provide context and re-dispatch") — both pause for plan/context editing. Dropping 3 collapses the gate to 4 options without losing a unique control-flow outcome.
- **README install Option A rewritten using `/plugin marketplace add` syntax.** The pre-audit README opened Option A with `# Once Claude Code's plugin install supports github.com URLs:`, gating the install on a future condition that never landed in the form expected. Verified the actual current syntax via `claude-code-guide` agent (citing https://code.claude.com/docs/en/discover-plugins.md): `/plugin marketplace add <owner>/<repo>` then `/plugin install <plugin-name>@<marketplace-name>`. Documented both the literal command and the interactive `/plugin` Discover-tab fallback so syntax drift in Claude Code doesn't strand first-time installers.
- **Hook portability rewrite landed for B9/B10 with Linux-only smoke verification.** `find -quit` and `find -printf` are GNU extensions; on macOS BSD `find`, `-quit` is ignored (returns all matches; first one wins by accident or not) and `-printf` produces no output (the most-common transcript-resolution fallback silently degrades). Replaced with `head -n1` and a `stat -c '%Y' || stat -f '%m'` dual form. Verified end-to-end on this Linux env via a synthetic worktree fixture (jq present, output JSONL has all 10 expected fields with correct values). The macOS path is portable-by-construction but not smoke-tested — README softens the "tested on Linux" claim and points readers at GitHub issues if telemetry doesn't land.
- **B5 fix went through two passes.** First pass clarified the Step C dispatch guard's composite-path description ("Continue to plan now → Start execution now" → spelled out as B1's flip + B3's explicit pick). The fresh-eyes Explore smoke-read after Wave 1 caught that the rewrite implied B3's gate was transparent, and that the related B1 option label used "B0a worktree check" when B0 (not B0a) is what ran during the brainstorm. Second pass tightened both, plus updated B3's `post-plan` gate description to mention the B1-via-flip arrival path and B2's dispatch guard prose to clarify the post-brainstorm vs flipped-to-post-plan distinction. The halt_mode flow is the highest-risk surface in the audit — it has tendrils across B1/B2/B3/C/P — and the second pass was specifically designed to catch new-introduced contradictions.
- **B8 fix verified empirically against a 3-record fixture.** The original `jq foreach` query returned `growth: 0` for every record (the UPDATE expression `$r` overwrote the accumulator each iteration, so `$r.transcript_bytes - $r.transcript_bytes = 0`). Confirmed broken behavior, then confirmed the rewritten `range`-based indexed-access query returns the expected `growth: 500, 700` for transcript bytes 1000 → 1500 → 2200. The snippet has been broken since shipped; nobody noticed because nobody had a reason to copy-paste it yet. Will be useful starting with v1.0.0 telemetry adoption.
- **P15 (self-referential `superflow-detect` skill) deferred to backlog.** The skill auto-fires on this repo because the repo ships `WORKLOG.md` and orphan plans (`docs/superpowers/plans/2026-05-02-superflow-subcommands.md` has no sibling status file). Anyone who installs the plugin and runs detection against this repo will be suggested to import the plugin's own implementation history. This is acceptable as a dogfooding signal for v1.0.0 — revisit post-release if it confuses real users.
- **Wave 1 verification used a fresh-eyes Explore subagent for end-to-end smoke read.** The orchestrator is a 1075-line markdown prompt with no test suite. Confirmation bias on the editor (me) doing 13 sequential Edits across one file is real. Dispatching a different-instance Explore agent with the prompt "find any contradictions, dangling references, broken cross-refs, or stale draft remnants" caught 5 second-order issues introduced by the first-pass edits — applied as a quick second pass before declaring Wave 1 done.

**Operational notes:**

- Single-session execution: orientation → 3 parallel Explore audits → 1 Plan agent for bundling/sequencing → 4 batched user decisions via `AskUserQuestion` → ExitPlanMode → 4 sequential waves of fixes (orchestrator → docs → hook+skill → release bookkeeping). One commit (still pending at WORKLOG-write time) covers all changes.
- TaskList used to track the four waves. All audit findings batched into 4 task items rather than 24 individual items — finer granularity wouldn't have helped, the work had natural wave boundaries.
- Used the user's preference (per memory) for `AskUserQuestion` over open-ended prose at the 4 decision points.
- Total Edits: 16 in commands/superflow.md (Wave 1 first + second pass), 6 in README.md, 1 in CHANGELOG.md (large), 1 in plugin.json, 1 in WORKLOG.md (this entry), 1 in docs/design/telemetry-signals.md, 1 in skills/superflow-detect/SKILL.md, 4 in hooks/superflow-telemetry.sh, 1 in docs/superpowers/plans/2026-05-01-superflow-small-fixes-status.md.

**Verification performed:**

- All 10 blocker discriminator greps return zero post-fix. All positive greps for new content return ≥1.
- Halt-mode flow re-grepped after each B1/B5/P1/P2/P4/P5 edit — no orphan references.
- Doctor checks table size confirmed at exactly 14 rows (matches "all 14 checks" claim post-B2).
- Codex annotation form check: 11 instances of `**Codex:**` (canonical), 0 instances of lowercase `codex: ok|no` (post-P3).
- Verb routing table has 14 rows post-P7 (added `new` (no topic) row).
- B8 jq fix verified against a 3-record heredoc fixture: returns `growth: 500` and `growth: 700` for the documented inputs (vs. the original query's `growth: 0` for all records).
- `bash -n hooks/superflow-telemetry.sh` passes.
- Hook smoke test on Linux: defensive bail confirmed (exit 0, no output) when on a branch with no matching status file. Success path confirmed in a synthetic worktree fixture: all 10 JSONL fields populated correctly, `activity_log_entries: 2` matches fixture, `wakeup_count_24h: 1` matches fixture, transcript bytes/lines populated from a real session jsonl.
- Fresh-eyes Explore smoke read of `commands/superflow.md` after Wave 1 caught 5 second-order issues; second pass applied, re-verified.

**Known gaps + followups:**

- **macOS portability for `hooks/superflow-telemetry.sh` is portable-by-construction but not smoke-tested.** Linux verification only. README points readers at GitHub issues.
- **No regression test for the orchestrator.** It's a markdown prompt; behavior emerges from a live agent reading it. Future v1.x: consider canned-`$ARGUMENTS` self-test specs in `docs/superpowers/specs/` that exercise every verb's branch — would catch routing-table drift.
- **P15 (self-referential `superflow-detect` on this repo)** deferred to post-release backlog.
- **The three deferred consistency items from v0.4.0 prep** (split verbs table by phase/op, drop back-compat framing for bare-topic, sample-topic consistency) — still small enough to bundle into a v1.0.x doc-only pass if other small items accumulate.
- **Doctor check for the three-place verb-list invariant** (commands/superflow.md frontmatter `description:`, reserved-verbs warning, routing table) — flagged at v0.4.0 close-out, still not implemented. Worth a v1.0.x add since adding/renaming a verb in the future requires updating three sync'd locations.
- **B0a vs. B0 reference in the B1 "Continue to plan now" gate option** was tightened in the second pass after the smoke-read agent caught the original wording said "B0a worktree check" but B0 is what actually ran. Worth re-reading after future B/B0/B0a refactors.
