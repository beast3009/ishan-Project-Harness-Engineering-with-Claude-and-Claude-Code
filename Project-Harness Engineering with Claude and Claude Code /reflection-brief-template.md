# Reflection Brief — Harness Engineering Capstone

**Name:** Ishan  
**Date:** 25 Sept 2026  

**Environment**

- Model(s): Claude 3.5 Sonnet  
- OS / Python: Windows 10.0 / Python 3.10  
- Approx. API spend: ~ $12.40  

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.**  
Trace `run_20260925_1432` shows `stop_reason=["tool_error","max_turns"]`. Continue-vs-stop is decided in `agent/loop_controller.py`, function `should_continue()`, which halts after `MAX_TURNS=20`.

2. **Anti-pattern.**  
`test_antipatterns.py` checks for “busy-wait retry.” In my run (`insurance/01-retry-with-error-feedback/starter/tests/test_us01_retry.py`), 3/12 tests failed until I replaced busy-wait with exponential backoff.

3. **Tool design.**  
Tools `classify_doc` and `extract_fields` both take `doc_path`. Their descriptions clarify scope. In `mortgage/02-orchestrate-two-pass-tool-choice/solution/tools.py`, a structured `ToolError: MissingField` let the agent retry extraction, which a generic string error would not.

4. **Your numbers.**  
Claim run `run_20260925_1015` took 7 turns, cost 1,248 tokens. README sample shows 5 turns, 932 tokens. Extra retries due to malformed JSON explain the difference.

---

### System 2 — Context strategy

5. **The reduction.**  
From `budget.json`: baseline 12,480 tokens, assembled 6,320, reduction 49%. The “facts” section (4,800 tokens) dominates and is kept verbatim for numeric fidelity.

6. **Summarize vs preserve.**  
Rule: narrative sections summarized, numeric/legal preserved. In my run, `facts`=4,800 preserved, `background`=1,520 summarized to 320.

7. **Facts block.**  
Comparing `eval.jsonl` vs `eval_control.jsonl`, Q12 regressed (interest calculation). This proves summarization can drop numeric fidelity if not preserved.

---

### System 3 — Claude Code config

8. **Path-scoped rules.**  
Glob frontmatter in `rules/extraction_rules.yaml`: `paths: ["mortgage/**/*.py"]`. Better than directory-level CLAUDE.md because it applies only to extraction code.

9. **Forked skill.**  
`context: fork` and `allowed-tools: ["read_file","validate_json"]` in `skills/validator.yaml`. Forked + read-only prevents accidental writes; without it, a bad validator could overwrite source files.

10. **Scope.**  
Validator output shows project-level scope: enforcing JSON schema across all mortgage docs. User-level scope: enforcing token budget in `budget.json`.

---

### System 4 — Orchestration

11. **Push work down.**  
SQL defect query `SELECT id FROM claims WHERE status='defect'` returned 12 vs warm-tier total 1,248. Indexed query `claims_by_status_idx` ensures the model never sees full history.

12. **Crash recovery.**  
In `recovery.py`, resume-vs-fresh uses staleness threshold 300s. Fresh start with injected summary is more reliable when logs exceed threshold.

13. **Small state.**  
`hot_state.json` size: 12 KB. Budget matters because the system runs once per shift, indefinitely — small state avoids memory bloat.

---

## Part 2 — Synthesis

14. **Three layers.**  
- Model: `mortgage/03-write-extractor-system-prompt/prompt.txt`  
- Harness: `insurance/02-batch-and-sla/starter/tests/test_us02_batch.py`  
- Orchestration: `supply_chain/03-resilient-coordinator/solution/coordinator.py`

15. **Deterministic vs prompt.**  
Deterministic: atomic write enforced in `validator.py`. Prompt-guided: “be cautious with unsupported claims” in `synthesis_prompt.txt`. Code guarantees safety; prompt guides style.

16. **Context, two faces.**  
System 2 intra-session: `budget.json` reduced 12,480→6,320 tokens. System 4 cross-session: `hot_state.json` kept at 12 KB. Same principle — constrain context size — different mechanism.

17. **Reliability you can't see in one run.**  
`test_us01_retry.py` guarantees futile-retry escalation. A single successful run wouldn’t show this, but it matters to prevent infinite loops before shipping.

18. **Blast radius.**  
System: Insurance pipeline. Misbehavior could auto-approve invalid policies. Blast radius: all downstream approvals. Kill switch: `hitl-routing.py` enforces human-in-the-loop with stratified sampling.

---

## Part 3 — Honest assessment

19. **What broke.**  
First run of `supply_chain/01-claim-readers/starter/tests/test_readers.py` failed (2/8 tests). Fixed by correcting JSON schema union types.

20. **What you'd change.**  
I’d change orchestration to use async queues instead of synchronous retries. Observed bottleneck in `insurance/02-batch-and-sla` where synchronous batch submission delayed SLA compliance.
