# Reflection Brief - Harness Engineering Capstone

**Name:** Neeharika Nanjarapalli
**Date:** 1st Sept 2026

**Environment**

- Model(s): `claude-haiku-4-5-20251001` (System 1, per `summary.md`); recorded-response client, no live model call (System 4)
- OS / Python: Linux, Python 3.13.0 (all four `pytest` headers)
- Approx. API spend: ~$0.1181 for System 1's 8-claim run (per `summary.md`); System 2's compression calls cost ~12,334+347 and ~11,475+503 input/output tokens for the refund and subscription summaries respectively (per `budget.json`, dollar figure not itemized); System 4 ran fully offline via `--recorded-response`, $0 spend.

---

## Part 1 - Per-system

### System 1 - Agentic loop

**1. Loop control.** The trace for `claim_01_kitchen_fire` shows: turn 1 `stop_reason: tool_use` (calls `lookup_policy`), turn 2 `stop_reason: tool_use` (six `record_claim_fact` calls), turn 3 `stop_reason: end_turn`. Termination is decided in `claims_intake/loop.py`, in the `run()` function: after each `client.messages.create(...)` call, it checks `if response.stop_reason == "end_turn": ... return FinalState(...)` to stop, and `if response.stop_reason == "tool_use": ...` to append tool results and continue the `while True:` loop. Nothing else drives the loop condition - no turn counter, no string match on the assistant's text.

**2. Anti-pattern.** `test_antipatterns.py::test_no_integer_literal_iteration_cap_in_loop` checks (via AST analysis) that `loop.py` contains no `for _ in range(<int literal>)` or `while <var> < <int literal>` construct as its stopping mechanism. If the loop instead used a hardcoded cap like `for turn in range(3)`, claims like `claim_02_stolen_bike` and `claim_03_water_damage` - which needed 5 turns each to reach `end_turn` - would have been silently truncated mid-tool-call at turn 3, before `classify_claim`/`route_to_adjuster` ever ran, instead of terminating cleanly.

**3. Tool design.** `route_to_adjuster` and `escalate_to_human` have overlapping shape: both are marked "TERMINAL TOOL," both are meant to be called exactly once, and both carry a summary field (`claim_summary` vs. `structured_summary`) describing the same claim. Misrouting is prevented because both descriptions encode the identical numeric threshold: `route_to_adjuster` requires "classification confidence is at least 0.6," `escalate_to_human` fires when "confidence is below 0.6... or... cannot be routed safely." Since `classify_claim` already commits a single confidence number earlier in the loop, the two descriptions partition cleanly on that value with no ambiguous middle ground. Separately, `_t_lookup_policy` returns a structured error - `_err("permanent", False, f"policy_id {pid!r} not found")`, i.e. `{"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "..."}` - instead of a bare string. The `is_retryable: false` field lets the agent immediately infer that retrying the same tool call is pointless and it should ask the claimant for a corrected ID or escalate, rather than looping on retries as it might if it only received an unstructured error string with no retry signal.

**4. Your numbers.** `claim_01_kitchen_fire`: 3 turns, 9,494 input / 566 output tokens, est. cost $0.0123, 5.9s elapsed (per `summary.md`). Compared against `claim_03_water_damage` in the same run - 5 turns, $0.0207, 11.8s - the difference is one `request_clarification` round trip (`clarifications_asked: 1`): each extra clarification adds a full turn (question out, claimant reply in) plus whatever prior context has already accumulated, which is why the more ambiguous claim costs roughly 1.7× as much as the straightforward one despite processing a comparably-sized incident.

### System 2 - Context strategy

**5. The reduction.** From `budget.json`: baseline 38,708 tokens → assembled 16,851 tokens, a 56.47% reduction. Per-section: `case_facts` 204, `resolved_refund` 360, `resolved_subscription` 516, `active` 15,789. The `active` section dominates (15,789 of 16,851 total, ~94%). It's kept verbatim per `assemble.py`'s design: the active issue sits at the bottom boundary, immediately before the model's next reply, and that's the position models attend to most reliably - any lossy compression there risks the very next turn hallucinating a detail from the just-happened exchange.

**6. Summarize vs preserve.** The rule is enforced in `compressor.py::summarize_segment`: `if segment.status != "resolved": raise ValueError(...)` - only `resolved` segments are ever sent through the compression call; the active segment is structurally exempt. In numbers: the two resolved issues compressed down to 360 and 516 tokens each (they're finished, so only their factual conclusions matter going forward), while the active issue stayed at its full 15,789 tokens because the model must still reason over its turn-by-turn detail live.

**7. Facts block.** Comparing `eval.jsonl` to `eval_control.jsonl`: Q6 ("What is the structured status of the payment-method update issue?", expected fragment `in_progress`) passed in the full run - the model answered `payment_update_status: in_progress` - but failed in the control (case-facts block removed), where the model instead claimed "there is no case record with a structured status token for the payment-method update issue... remains in active conversation without a formal case token." All other 5 questions passed in both versions. This proves the persistent case-facts block is load-bearing specifically for that one structured field: the raw active-issue text is still present verbatim in the control, but without the redundant top-of-context facts block, the model can't reliably surface that particular structured token.

### System 3 - Claude Code config

**8. Path-scoped rules.** `.claude/rules/tests.md`'s frontmatter:
```yaml
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
```
This is better than a directory-level `CLAUDE.md` for a cross-cutting convention like test-file rules, because co-located test files live scattered across `src/components/`, `src/pages/`, `src/api/`, and `src/db/` simultaneously (per the repo layout in `CLAUDE.md`). A directory-level file would need to be duplicated into every one of those directories, or hoisted to a shared ancestor that also picks up irrelevant files. One glob-scoped rule activates precisely when Claude touches a matching file, regardless of where in the tree it lives - confirmed by `test_ac_02_06_test_file_matches_react_and_tests`, which checks that a test file under `src/components/` correctly matches *both* the React rule and the tests rule at once.

**9. Forked skill.** `.claude/skills/deploy-check/SKILL.md` frontmatter:
```yaml
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(git status:*)
  - Bash(git diff:*)
  - Bash(git log:*)
  - Bash(git rev-parse:*)
  - Bash(git ls-files:*)
  - Bash(gh pr view:*)
  - Bash(gh pr checks:*)
```
Running forked keeps the verbose intermediate output - file enumeration, diff parsing, `git status` noise - out of the main session; only the structured pass/fail summary returns. Read-only `allowed-tools` means the check structurally cannot push, deploy, or modify files, even if the model tried. Without `context: fork`, every deploy check would dump raw git output into the calling session's context on every invocation, burning tokens and drowning the actual conversation. Without the read-only allowlist, a future edit to the skill file could silently grant `Write` and the check would gain the ability to modify state during what's supposed to be pure verification - the skill file itself says exactly this: "even if a future maintainer adds `Write` here, the team's `/review` command should catch the change."

**10. Scope.** Validator output: `OK` (exit 0), confirming the hierarchy resolves cleanly. Project-level example: `./CLAUDE.md` plus `.claude/rules/*.md` (e.g. `api.md`, `react.md`, `tests.md`) - shared via git, loaded into every teammate's session automatically. User-level example: `~/.claude/skills/deploy-check-strict/` - the skill file itself documents this: "If you want a stricter personal variant... create a parallel skill in `~/.claude/skills/deploy-check-strict/`... Your variant will not affect teammates and will not be committed to the repo."

### System 4 - Orchestration

**11. Push work down.** The shift run printed `shift C: 0 new defects` against a warm tier seeded with 40 total defect rows (`fixtures/defects.json`). The named indexed query is `WarmStore.defects_since()` in `shift_monitor/warm.py`: `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, run against an indexed `ts` column. The model never sees the full 40-row history because `gather_new_defects` is a pure passthrough to this SQL call with no Python-side filtering (confirmed by `test_gather_new_defects_has_no_python_side_filtering`) - the `WHERE`/`LIMIT` happen inside SQLite before any row reaches the prompt, so only whatever handful of rows matched the time window ever gets built into context, regardless of how large the warm tier grows over months of shifts.

**12. Crash recovery.** `recovery.py` sets `STALE_RESUME_THRESHOLD_MINUTES = 30`. `decide()` returns `"resume"` only when the manifest has incomplete steps and `now - last_step.ts <= 30 minutes`; otherwise `"fresh"`. The test suite pins the boundary exactly: `test_recovery_decide_truth_table[30-False-resume]` and `[31-False-fresh]` confirm the cutoff is inclusive at 30 minutes. A fresh restart with an injected summary is more reliable past that window because the working set has likely moved on - a new shift's data may have arrived, or a root cause may already be understood - so blindly resuming risks reasoning over context that no longer matches reality, whereas a fresh start with a compact summary of what the crashed run already found discards only the stale partial reasoning, not the actual findings.

**13. Small state.** `data/hot_state.json` is 643 bytes, well under the ~5 KB budget. This matters because `shift_monitor` runs once per shift indefinitely - the same file is read and rewritten on every future invocation forever. `HotState` structurally caps this (`test_hotstate_rejects_more_than_20_hashes`): without that cap, `recent_defect_hashes` would grow without bound as more shifts ran, and every future shift's prompt would get proportionally larger, slower, and more expensive. Keeping it near-constant (643 B today, regardless of how many shifts preceded it) is what keeps the system's per-invocation cost flat over its entire operational lifetime instead of growing linearly with shift count.

---

## Part 2 - Synthesis

**14. Three layers.**
- **Model:** `claims_intake/tools.py`'s `TOOL_SCHEMAS` (e.g. `classify_claim`'s enum-constrained `input_schema`) - this is literally what Claude is told it can do; the model layer is the reasoning inside Claude, exposed only through what the tools and prompts describe.
- **Harness:** `claims_intake/loop.py`'s `run()` function - the `stop_reason`-driven `while True:` loop, budget checks, and tool-result feedback - decides when to call the model again and when to stop.
- **Orchestration:** `shift_monitor/recovery.py` + `manifest.py` + `fork.py` - session-level concerns spanning multiple invocations over time (crash recovery, fork-and-merge, tiered warm/cold state) that exist above any single agentic loop.

**15. Deterministic vs prompt.** Deterministic (enforced in code): System 4's `HotState` size cap (`test_hotstate_rejects_more_than_20_hashes`, `write_atomic`) guarantees `hot_state.json` never exceeds budget no matter what the model outputs; System 3's `deploy-check` `allowed-tools` allowlist (no `Write`) makes it structurally impossible for the skill to push or deploy. Prompt-guided: System 1's `confidence >= 0.6` threshold between `classify_claim`/`route_to_adjuster` and `escalate_to_human` - nothing in `loop.py` enforces that number in code; the model is trusted to follow it. Enforce in code when a failure is catastrophic or invisible until too late (state bloat, an unauthorized write); use prompt guidance when the decision is genuinely judgment-based over ambiguous, case-specific facts, where a hard rule would misfire on edge cases.

**16. Context, two faces.** System 2 compresses *within* one continuous conversation: resolved segments shrink to 516/360 tokens, `case_facts` stays a fixed 204-token block, the active segment stays byte-exact at 15,789 tokens - 38,708 → 16,851 total (56.47% reduction), all inside one transcript. System 4 compresses *across* invocations that don't even share a process: each shift is a fresh CLI call, so instead of summarizing a transcript, it filters at the SQL layer before anything reaches a prompt (`defects_since()`'s indexed `WHERE ts > ?`) and persists only a fixed ~5 KB snapshot (643 bytes actual) between runs - no conversation history at all survives between shifts. Same principle - never make the model re-read more than it needs right now - different mechanism: summarize-and-position text within one window vs. filter-at-the-source and persist a tiny structured snapshot between windows that don't overlap in time.

**17. Reliability you can't see in one run.** `test_recovery_decide_truth_table`'s boundary cases (`[30-False-resume]`, `[31-False-fresh]`) and `test_hotstate_rejects_more_than_20_hashes` guarantee behavior at exact edges that a single successful run would never exercise - a lucky shift with only 3 defect hashes, or a crash recovered at minute 5, looks identical whether or not the 20-hash cap or the 30-minute boundary is implemented correctly at all. This matters before shipping because production shifts will eventually land exactly on these edges (a bad week with >20 distinct defects; a crash recovered at minute 31), and you want confidence those paths work before the first time they happen for real.

**18. Blast radius.** System 3's `deploy-check`: its `allowed-tools` are entirely read-only (`Read`/`Grep`/`Glob`/`Bash(git status|diff|log|rev-parse|ls-files:*)`/`Bash(gh pr view|checks:*)`), so it structurally cannot push, deploy, write files, or run migrations even if the model tried - worst case is a human trusts a wrong pass/fail verdict and ships anyway. The kill switch is the `allowed-tools` list itself: since it's config, not code the skill can rewrite, tightening or removing it (or deleting the skill file from `.claude/skills/`) instantly and completely disables any capability beyond read-only inspection - no runtime flag needed, because the enforcement point is the tool-availability boundary Claude Code checks before any tool call is dispatched.

**19. What broke.** System 1's venv installed `httpx>=0.28`, which dropped the `proxies` kwarg that `anthropic==0.39.0`'s client init relies on, raising `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'` inside `client.py`'s `make_client()`. Fixed by pinning `httpx==0.27.2`/`httpcore==1.0.5` in that venv specifically. Applying that same fix inside System 2's venv then broke it a second way: `anthropic==0.39.0` has no `messages.count_tokens` method, so `retail_context/tokens.py`'s `count()` raised `AttributeError: 'Messages' object has no attribute 'count_tokens'`. Fixed by reinstalling `anthropic==0.69.0` specifically in System 2's venv - underscoring that each system's SDK pin is load-bearing and not interchangeable across environments.

**20. What you'd change.** I'd pin exact `httpx`/`httpcore` versions (or ship a lockfile) in each system's own `pyproject.toml` instead of leaving them as unpinned transitive dependencies. Leaving pip free to resolve "whatever's newest" is exactly what caused the `proxies` `TypeError`, and cost two separate debugging cycles across two different systems before landing on the right combination for each.