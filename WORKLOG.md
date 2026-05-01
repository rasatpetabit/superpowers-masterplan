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
