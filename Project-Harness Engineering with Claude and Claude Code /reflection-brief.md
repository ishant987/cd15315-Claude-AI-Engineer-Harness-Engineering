# Reflection Brief — Harness Engineering Capstone

**Name:** Ishant  
**Date:** 2026-09-24  

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): `claude-haiku-4-5-20251001` (System 1 & System 2), `RecordedClaudeClient` (`fixtures/recorded_responses/shift_C_2026-04-30.json` for System 4)
- OS / Python: macOS (Darwin ARM64) / Python 3.11.14
- Approx. API spend: ~$0.04 USD total ($0.0381 USD for System 1 run `20260924_111500` across 8 claims; $0.00 USD for System 4 recorded offline shift execution)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In trace artifact `evidence/system-1-claims-intake/traces/claim_01_kitchen_fire.jsonl` (run `20260924_111500`), the 6-turn sequence of `stop_reason` values is:
   `["tool_use", "tool_use", "tool_use", "tool_use", "tool_use", "end_turn"]`.
   Loop termination is decided in `claims_intake/loop.py` inside the `run()` function (lines 103–132). When `response.stop_reason == "tool_use"`, the loop executes each requested tool via `tool_executor()`, appends `tool_result` blocks into the message history as a user turn, and continues to the next iteration. When `response.stop_reason == "end_turn"`, the function appends the assistant's final response and immediately returns the `FinalState` dataclass. Any other reason (such as `max_tokens`) raises `UnexpectedStopReason`, preventing silent failures.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `tests/test_antipatterns.py` checks `test_no_string_membership_against_text_in_loop` (AST check forbidding string-in-text checks) and `test_no_integer_literal_iteration_cap_in_loop` (forbidding hard-coded integer loop bounds). If the loop used a string-membership check like `"ROUTED" in response.text` to terminate, the loop could terminate prematurely if the model mentioned "routed" in an exploratory thought or clarification step before completing essential prerequisite tools like `classify_claim` and `assess_severity`. Furthermore, if a hardcoded iteration cap like `for _ in range(5)` were used, complex multi-turn claims such as `claim_03_water_damage` and `claim_07_tree_falls_on_car` (which required 7 turns to ask clarification, record facts, classify, assess severity, route, and confirm) would be prematurely aborted before reaching a terminal queue decision.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → In `claims_intake/tools.py:TOOL_SCHEMAS`, `route_to_adjuster` and `escalate_to_human` both accept claim context and policy details, but their descriptions establish clear mutually exclusive operational contracts. `route_to_adjuster` explicitly mandates that `classify_claim` has achieved confidence ≥ 0.6 and that `assess_severity` has been completed, whereas `escalate_to_human` specifies preconditions for confidence < 0.6, unresolved ambiguity between multiple candidate types, or missing facts. When tool execution fails, the harness returns a structured error object `{"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "structured_summary missing fields: ['root_cause']"}` instead of an unformatted string. This explicit schema allows the model to immediately identify and provide the exact missing parameters on its next turn rather than guessing or retrying blindly.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → For `claim_01_kitchen_fire` from `evidence/system-1-claims-intake/summary.md` (run `20260924_111500`), the claim completed in 6 turns with 3,350 input tokens, 195 output tokens, $0.0043 estimated cost, and 0.8s elapsed time. Across all 8 fixtures in run `20260924_111500`, total estimated cost was $0.0381 USD across 50 total turns. The turn count (6 turns) and token volume reflect standard dynamic decomposition (1 turn lookup policy + 1 turn record facts + 1 turn classify + 1 turn assess severity + 1 turn route + 1 turn final confirmation). Slight variations in input token count compared to the README sample stem from dynamic token accumulation in multi-tool user response blocks and specific system prompt formatting.

---

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → From `evidence/system-2-retail-context/budget.json` (run `20260924-111400`), baseline tokens was 47,144, assembled tokens was 19,813, and reduction was 57.97% (comfortably exceeding the ≥ 50% target). The active section (`active`: 19,538 tokens) dominates the assembled context, comprising ~98.6% of all assembled tokens. It is preserved byte-exact verbatim because the active issue (`payment_update`) is unresolved and in-progress; the assistant requires exact conversational turns, precise error responses (such as `AVS_MISMATCH`), and exact card update attempts to troubleshoot without hallucination or lost context.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The architectural rule in `retail_context/assemble.py` enforces a clear boundary: resolved past issues are aggressively compressed via LLM summarization, key invariant transactional metadata is extracted into a persistent structured facts block, and only the active, in-flight issue is preserved byte-for-byte.
   Citing per-section tokens from `budget.json`:
   - `# Case Facts`: `149` tokens (structured summary of 12 required fields across all 3 issues)
   - `# Resolved: Refund inquiry`: `60` tokens (compressed from ~14,200 input tokens)
   - `# Resolved: Subscription cancellation`: `65` tokens (compressed from ~13,800 input tokens)
   - `# Active issue: Payment-method update`: `19,538` tokens (verbatim raw turns 29–48)

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → In `evidence/system-2-retail-context/eval.jsonl`, all 6 evaluation questions passed (6/6, 100%). In `evidence/system-2-retail-context/eval_control.jsonl` (the control run where the `# Case Facts` block was stripped from the context), questions **Q1** ("What was the actual refund amount processed for order ORD-77310?") and **Q6** ("What is the structured status of the payment-method update issue...?") both regressed from `PASS` to `FAIL` (model returned `unknown`). This proves that the `# Case Facts` block at the top boundary is load-bearing: the model relies on this top-level structured scratchpad to answer precise transactional queries whose exact numeric values or machine-readable tokens were condensed away during resolved-issue summarization.

---

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → In `evidence/system-3-ecommerce-config/.claude/rules/react.md`, the frontmatter specifies:
   ```yaml
   ---
   description: Conventions for React components and pages
   paths:
     - "src/components/**/*"
     - "src/pages/**/*"
   ---
   ```
   Similarly, `.claude/rules/tests.md` matches `paths: ["src/**/*.test.ts", "src/**/*.test.tsx"]`. Path-scoped rules are superior to directory-level `CLAUDE.md` files because cross-cutting conventions (such as testing standards for co-located test files or React patterns across components and pages) span multiple directory subtrees. Placing rules in `.claude/rules/` with globs keeps configuration DRY in one central location and allows multiple rules to layer dynamically on matching files (e.g., `Cart.test.tsx` inherits both React and testing rules), eliminating redundant nested `CLAUDE.md` files.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → In `evidence/system-3-ecommerce-config/.claude/skills/deploy-check/SKILL.md`:
   ```yaml
   context: fork
   argument-hint: "[target-branch] (defaults to main)"
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
   Running forked + read-only isolates verbose validation outputs (such as large multi-file diffs, commit logs, and check traces) inside an ephemeral sub-agent, returning only a clean `verdict: pass|fail` summary to the main session. The read-only tool allowlist guarantees that pre-deployment checks cannot inadvertently modify source files, stage commits, or trigger deployments. Without `context: fork`, megabytes of diff output flood the main conversation context, driving up costs and degrading downstream model reasoning.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator `python -m ecommerce_team_config .` verified all 35 test assertions (`OK`, exit code 0; `evidence/system-3-ecommerce-config/validator_output.txt`).
    - **Project-level scope**: Located in the repository under `./CLAUDE.md`, `.claude/rules/react.md`, and `.claude/commands/review.md`. These are checked into version control so the entire engineering team shares the same conventions and PR review criteria.
    - **User-level scope**: Located on an individual developer's machine under `~/.claude/CLAUDE.md` or `~/.claude/skills/deploy-check-strict/`. These contain local personal workflows and preferences that are explicitly excluded from git so they do not enforce personal styles on teammates.

---

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → In `evidence/system-4-shift-monitor/shift_output.txt`, the shift run returned `0` new defects (for the current time window) out of `40` total historical defects seeded into `data/warm.sqlite` (`fixtures/defects.json`).
    - Indexed query: `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` in `shift_monitor/warm.py:defects_since()`, leveraging index `CREATE INDEX IF NOT EXISTS idx_defects_ts ON defects(ts);`.
    The model never sees the entire defect history because pushing temporal filtering down to SQLite keeps the context window constant O(1) per shift. If the full 40+ defect history were loaded on every 8-hour shift, token usage and latency would grow quadratically over weeks of plant operation.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → In `shift_monitor/recovery.py`, the `decide()` function uses a staleness threshold of `STALE_RESUME_THRESHOLD_MINUTES = 30`. If an interrupted shift crashed within the last 30 minutes, it resumes the in-flight session. If the crash is older than 30 minutes, it triggers a "fresh" start. A fresh start with an injected prior summary is more reliable than resuming an old session because physical defect data and plant conditions evolve over an 8-hour shift; attempting to resume an obsolete in-flight dialogue after hours of downtime risks acting on stale defect states and hallucinating context. A fresh session starts with a clean context while retaining essential historical conclusions via the hot state summary.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → `evidence/system-4-shift-monitor/hot_state.json` is `643` bytes on disk, well under the `HOT_STATE_BYTE_BUDGET = 5,120` bytes (5 KB) limit enforced in `shift_monitor/state.py`.
    ```json
    {"recent_defect_hashes":[],"current_shift_summary":"Shift C 2026-04-30: 3 high + 2 medium defects on capacitor-bank-C-7...","active_alerts":[...],"threshold_statuses":{...}}
    ```
    This strict byte budget matters because the shift monitor runs 3 times a day, 365 days a year. Without an enforced byte ceiling and capped list sizes (`MAX_RECENT_HASHES = 20`), the state file would accumulate unbounded historical alert strings over hundreds of shifts, eventually blowing past the prompt budget and causing orchestrator crashes.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → **Model layer:** `claims_intake/loop.py` (line 72: `client.messages.create(...)`) — where raw LLM inference, token generation, and tool selection take place.
    → **Harness layer:** `retail_context/assemble.py` (lines 52–87) & `claims_intake/tools.py` — where deterministic constraints, schema validation, prompt positional assembly, and tool execution error handling wrap around the model.
    → **Orchestration layer:** `shift_monitor/pipeline.py` (lines 84–137) & `shift_monitor/recovery.py` — where multi-session scheduling, tiered database state synchronization (Hot/Warm/Cold), crash recovery policies, and append-only scratchpads manage long-term system execution.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → **Deterministic in code:** `shift_monitor/state.py:write_atomic()` enforces state integrity by writing to a temporary file, performing `os.fsync()`, replacing atomically, and verifying the payload stays ≤ 5,120 bytes. Likewise, `claims_intake/tools.py:_t_route_to_adjuster()` programmatically forbids routing unless `classify_claim` and `assess_severity` have already been recorded.
    → **Guided by prompt:** In `claims_intake/system_prompt.py` and `.claude/commands/review.md`, prompt guidance instructs the model on how to assess ambiguous damage descriptions or conduct the "Phase 1 Interview pattern" when intent is unclear.
    → **When each is right:** Deterministic code is required for non-negotiable invariants: security permissions, tool allowlists, transactional writes, and schema compliance. Prompt guidance is appropriate for semantic reasoning, linguistic nuance, and subjective classification where rigid rules cannot capture natural language variability.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → **System 2 (Intra-session):** Compresses a single long multi-issue dialogue from `47,144` baseline tokens down to `19,813` tokens (`57.97%` reduction, `budget.json`) using position-aware in-memory layout: top `# Case Facts` (149 tokens), middle resolved summaries (125 tokens), and bottom active verbatim turns (19,538 tokens).
    → **System 4 (Cross-session):** Compresses history across days and shifts by keeping in-context state at `643` bytes (`hot_state.json`, < 5 KB budget) and delegating all 40 historical defects to SQLite (`WarmStore`), querying only the relevant time window slice into the prompt.
    → **Common principle vs mechanism:** Both systems enforce the core principle of **maximizing information density at the boundary while minimizing token waste**. System 2 achieves this within a single context window via structured extraction and hierarchical Markdown assembly; System 4 achieves this across sessions via persistent tiered storage (Hot RAM/JSON, Warm SQLite, Cold monthly archives).

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → In System 1, `tests/test_loop.py::test_budget_raises_when_token_cap_exceeded` and `test_loop_raises_on_unexpected_stop_reason` test behavior under abnormal model responses (e.g., token exhaustion or `stop_reason == "max_tokens"`). A single manual run only exercises the happy path where the model returns `tool_use` and `end_turn`. Automated tests guarantee that if the API produces truncated output or runs over budget, the system fails fast with structured exceptions (`BudgetExceeded`, `UnexpectedStopReason`) rather than looping infinitely, corrupting claim queues, or incurring uncontrolled billing.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → **System 1 (Claims Intake Agent):**
    - **Blast radius:** If the agent hallucinates or classifies incorrectly, the blast radius is strictly contained to appending a JSON record into an incorrect queue file (e.g., `queues/property_damage.jsonl` instead of `queues/liability.jsonl`) or creating an unnecessary entry in `escalations.jsonl`. The agent has no write access to customer policies (policy lookups in `data/policies.json` are read-only) and cannot authorize monetary payouts.
    - **Enforcement & Kill switch:** The `ClaimSession.terminal_called` flag in `claims_intake/tools.py` programmatically prevents multiple terminal tool executions per session. Furthermore, `claims_intake/budget.py` acts as an automated kill switch: if token count exceeds `max_input_tokens` (500,000) or elapsed time exceeds `max_wall_clock_s` (180.0s), `Budget.check()` immediately halts execution. Downstream, all routed claims are queued for human adjuster verification before financial disbursement.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → Two environment failures occurred and were resolved during setup:
    1. During initial package installation in the Python 3.11 virtual environment, pip failed with `OSError: [Errno 28] No space left on device` due to accumulated system build caches. We purged `pip cache purge` and cleared non-essential cached files to free 2.9 GiB of disk space.
    2. In System 1, `anthropic==0.39.0` conflicted with modern `httpx >= 0.28.0` where the deprecated `proxies` parameter was removed in httpx, producing `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'`. We resolved this by installing `httpx<0.28.0` (`httpx==0.27.2`), allowing the 0.39.0 client to construct cleanly and pass all 29 tests.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → In System 1 (`claims_intake/loop.py`), terminal tools (`route_to_adjuster` and `escalate_to_human`) currently require the model to take an additional turn just to output a closing acknowledgement and emit `stop_reason == "end_turn"`. In production, I would modify `loop.py` to immediately terminate execution upon detecting a terminal tool execution in `response.content`, synthesizing a standard confirmation message programmatically. This would eliminate one full LLM round-trip turn per claim, cutting latency by ~15–20% and saving token costs across high-volume intake pipelines.
