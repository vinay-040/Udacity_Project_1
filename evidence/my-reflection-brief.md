# Reflection Brief — Harness Engineering Capstone

**Name: M S Vinay**

**Date: 27/09/2026**

**Environment**

- **Model(s):** `claude-haiku-4-5-20251001`
- **OS / Python:** Linux workspace / Python virtual environments
- **Approx. API spend:** **$0.1481 USD**
---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

   → In `claim_01_kitchen_fire.jsonl`, the observed sequence is **`tool_use → end_turn`**. The control logic is in `claims_intake/loop.py`, function `run()`: it returns when `response.stop_reason == "end_turn"`, continues when it equals `"tool_use"`, and raises `UnexpectedStopReason` for any other value. This makes the model's API stop reason the primary loop-control signal rather than a fixed turn count. *(Artifact: `runs/20260925_170833/traces/claim_01_kitchen_fire.jsonl`; source: `claims_intake/loop.py`.)*

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

   → `tests/test_antipatterns.py` checks that the loop does **not use an integer-literal iteration cap as its primary stopping mechanism**, such as `for _ in range(5)` or `while turn < 5`. If the loop used a fixed cap, a claim that needed more turns could terminate before the model returned `end_turn` or before the required terminal tool action. The actual S1 implementation instead exits according to `stop_reason`, with the separate budget checks acting as safety limits. *(Artifacts: `tests/test_antipatterns.py`, `claims_intake/loop.py`, and `pytest_S1.log` — 29 tests passed.)*

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

   → `classify_claim` and `assess_severity` both accept a **rationale**, but their descriptions clearly separate their responsibilities: `classify_claim` commits to the claim type and confidence, while `assess_severity` commits to the severity bucket after classification. Their schemas also constrain the categorical values with enums, reducing ambiguous tool selection. Tool failures are returned as structured JSON containing `is_error`, `error_category`, `is_retryable`, and `message`, so the model can distinguish a retryable/transient problem from a permanent error instead of treating an arbitrary error string as normal successful tool output. *(Artifact: `claims_intake/tools.py` and `tests/test_tools.py`.)*

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

   → `claim_05_auto_collision` completed in **4 turns** at an estimated cost of **$0.0195**, using 14,867 input tokens and 918 output tokens. The README sample describes the expected run as **7 routed, 1 escalated** with an estimated cost of about **$0.05** and 29 tests. My actual run processed eight fixtures with different model-generated token usage and produced a total estimated cost of **$0.1481**, so the sample is a reference estimate rather than a fixed runtime result. *(Artifacts: `runs/20260925_170833/summary.md` and the S1 README.)*

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

   → `budget.json` reports a **38,708-token baseline**, **16,839 assembled tokens**, and a **56.5% reduction**. The `active` section dominates the assembled context at **15,789 tokens**, compared with 204 for `case_facts`, 352 for `resolved_refund`, and 512 for `resolved_subscription`. The active section is kept verbatim because it represents the currently active issue and its exact conversational details may be needed for the next response; summarizing it could remove important information.

   *(Artifact: S2 `budget.json`.)*

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

   → The strategy is to **compress resolved historical segments while preserving the active segment byte-exact**. The assembled sections were `case_facts` **204 tokens**, `resolved_refund` **352**, `resolved_subscription` **512**, and `active` **15,789**. The compression API reduced the refund input from **12,334** tokens to **339** output tokens and the subscription input from **11,475** to **499**, while the active segment remained detailed. *(Artifact: S2 `budget.json`.)*

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

   → The normal `eval.jsonl` passed **all 6/6 questions**, including Q6, which required the exact structured token `in_progress`. In `eval_control.jsonl`, **Q6 regressed and failed** because the model incorrectly said that no structured status token existed. This proves that critical structured facts need to be preserved explicitly: without the required fact in the control context, the model produced a plausible but incorrect answer. *(Artifacts: S2 `eval.jsonl` and `eval_control.jsonl`.)*

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

   → The frontmatter in `.claude/rules/react.md` is:

   ```yaml
   paths:
     - "src/components/**/*"
     - "src/pages/**/*"
   ```

   This activates the React conventions for matching component and page files wherever they occur in the repository. A path-scoped rule is better for a cross-cutting convention because it follows the file pattern rather than depending on a particular directory hierarchy or nested `CLAUDE.md`. *(Artifact: `.claude/rules/react.md`; validator result: `validator_output.txt` = `OK`.)*

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

   → `.claude/skills/deploy-check/SKILL.md` contains:

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

   The fork keeps verbose discovery output out of the main session and returns only the structured deployment summary. The read-only allowlist prevents the skill from modifying files, pushing, or deploying. Without the fork, intermediate discovery output would pollute the main context; without the read-only allowlist, the validation skill would have a larger blast radius and could perform changes instead of only checking the repository. *(Artifact: `.claude/skills/deploy-check/SKILL.md`.)*

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.

    → The validator returned **`OK`**, confirming the project configuration passed its checks. A project-level example is `./CLAUDE.md` and the `.claude/rules/` and `.claude/standards/` directories; these are repository-scoped and shared with the team through version control. A user-level example is `~/.claude/CLAUDE.md`, `~/.claude/commands/`, or `~/.claude/skills/`, which are personal and are not shared through version control. *(Artifacts: `validator_output.txt` and `.claude/` structure.)*

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

    → The warm-tier fixture contains **40 defects**, while the Shift C run returned **0 new defects** because the query's timestamp cutoff was later than the available defect records. The indexed query is:

   ```sql
   SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?
   ```

   The warm database defines indexes `idx_defects_shift_ts ON defects(shift, ts)` and `idx_defects_ts ON defects(ts)`. SQL performs the timestamp filtering and limit before results reach the model, so the model receives only the relevant new-defect set instead of the full historical database. *(Artifacts: `fixtures/defects.json`, `shift_run_output.txt`; source: `shift_monitor/warm.py` and `shift_monitor/pipeline.py`.)*

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

    → `shift_monitor/recovery.py` defines `STALE_RESUME_THRESHOLD_MINUTES = 30`. A run with no steps is `fresh`; a completed run is also `fresh`; an incomplete run resumes when its last step is **within 30 minutes**, and becomes `fresh` when it is older than 30 minutes. A fresh start can be more reliable when the partial state is stale because the system can restart cleanly while injecting the findings already captured in the manifest as a summary, rather than continuing from an outdated working state. *(Artifact: `shift_monitor/recovery.py`.)*

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

    → My `data/hot_state.json` measured **643 bytes**. The hot state contains only compact state needed for subsequent shifts rather than the full defect history. Keeping this persistent state bounded matters because the orchestrator may run once per shift indefinitely; otherwise state would grow continuously and eventually increase context size, cost, and processing time. *(Artifact: `hot_state_size.txt`.)*

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.

    → **Model:** `runs/20260925_170833/traces/claim_01_kitchen_fire.jsonl` — this shows the model's `stop_reason` values and tool calls, including `tool_use → end_turn`.

    → **Harness:** S2 `budget.json` — this shows deterministic context accounting and assembly: **38,708 baseline → 16,839 assembled**, a **56.5% reduction**.

    → **Orchestration:** S4 `shift_run_output.txt`, `hot_state_size.txt`, and `scratchpad_line.jsonl` — these show the shift-level pipeline, persistent state, and the hypothesis/evidence/conclusion record.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?

    → A deterministic behavior is S1's loop control in `claims_intake/loop.py`: `end_turn` returns, `tool_use` continues, and unexpected stop reasons raise an exception. A prompt-guided behavior is in `claims_intake/system_prompt.py`, which tells the model to ask one clarification when a claim is genuinely ambiguous and to route when confidence is at least 0.6 or escalate otherwise. Deterministic enforcement is appropriate when violating a rule could corrupt state or create uncontrolled execution; prompt guidance is appropriate when the model must reason about changing facts and choose among valid actions.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?

    → System 2 manages context **inside a long conversation**, reducing **38,708 baseline tokens to 16,839 assembled tokens (56.5%)** while preserving the active segment. System 4 manages context **across shifts**, keeping the persistent hot state to only **643 bytes** and using the warm database plus scratchpad to recover relevant history. Both follow the same principle—keep information needed for the next decision while avoiding unnecessary historical bulk—but S2 uses summarization/context assembly, whereas S4 uses persistent state, SQL-side filtering, and crash recovery. *(Artifacts: S2 `budget.json`; S4 `hot_state_size.txt`, `shift_scratchpad.jsonl`, and `warm.py`.)*

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

    → `tests/test_antipatterns.py` statically checks that the agentic loop does not use string-membership tests against assistant text or an integer-literal iteration cap as its primary stopping mechanism. A successful runtime trace alone would not reveal whether the implementation happened to contain one of these fragile control-flow patterns. The test matters before shipping because it protects the architectural invariant across future code changes, not just the behavior of today's successful run. *(Artifact: `tests/test_antipatterns.py`; S1 `pytest_S1.log` shows **29 passed**.)*

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.

    → In S1, the blast radius is primarily the **individual claim session** because claims are processed separately and terminal actions are explicitly controlled. `route_to_adjuster` and `escalate_to_human` are terminal tools, and the dispatcher prevents a second terminal action through `session.terminal_called`; the `Budget` also stops execution when input-token or wall-clock limits are exceeded. Therefore the practical kill switches are the terminal-action enforcement and `BudgetExceeded`, which prevent an uncontrolled claim loop from continuing indefinitely. *(Artifacts: `claims_intake/tools.py`, `claims_intake/loop.py`, and `claims_intake/budget.py`.)*

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)

    → One early environment problem was command/path handling when repository paths containing spaces were entered across multiple lines, causing the shell to interpret the path incorrectly. I fixed it by using the complete repository path as a single quoted command path and activating the correct repository-specific virtual environment before running the tests. I then verified the saved test artifacts: S1 **29 passed**, S2 **28 passed, 2 skipped**, S3 **35 passed**, and S4 **33 passed**. *(Artifacts: `evidence/system1_agentic_loop/pytest_S1.log`, `system2_context_strategy/pytest_S2.log`, `system3_claude_config/pytest_S3.log`, and `system4_orchestration/pytest_S4.log`.)*

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.

    → I would make the boundary between **structured state and conversational context** even more explicit from the beginning. S2 reduced the context from **38,708 to 16,839 tokens**, but the control evaluation showed that Q6 could fail when the exact structured status `in_progress` was not preserved. That suggests critical identifiers, status tokens, and decision state should remain deterministic structured data, while narrative history can be summarized more aggressively. *(Artifacts: S2 `budget.json`, `eval.jsonl`, and `eval_control.jsonl`.)*
