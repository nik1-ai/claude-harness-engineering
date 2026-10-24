
# Reflection Brief — Harness Engineering Capstone

**Name:** Nikhil Gupta
**Date:** 22/10/2026

**Environment**

- Model(s): claude-haiku-4-5-20251001 for System 1 (runs/20261021_201310/summary.md) and System 2 (DEFAULT_MODEL in retail_context/client.py, line 28; also recorded in case_facts_call.json). Systems 3 and 4 made no API calls.
- OS / Python: Linux (Vocareum code-server workspace), Python 3.13.0 (from the pytest header in test-output.txt).
- Approx. API spend: System 1 was $0.1037 (summary.md). System 2 did not print a cost, so this is my estimate from its token counts (about 200k input tokens across 9 Haiku calls): well under $0.50. Systems 3 and 4 cost $0 (validator and recorded response).

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** In runs/20261021_201310/traces/claim_05_auto_collision.jsonl the stop_reason sequence is "tool_use", "tool_use", "tool_use", "end_turn", which matches turns=4 for that claim in summary.md. The decision is made in claims_intake/loop.py, function run (line 46): line 113 continues when `response.stop_reason == "tool_use"` (it executes the tools and loops), line 103 returns when `stop_reason == "end_turn"`, and line 131 raises UnexpectedStopReason for any other value. The module docstring (lines 4-7) states the loop is driven only by stop_reason.

2. **Anti-pattern.** tests/test_antipatterns.py contains test_no_integer_literal_iteration_cap_in_loop, which rejects a hard-coded turn cap such as `range(5)`. In my run the claims needed different numbers of turns: claim_03_water_damage took 5, claim_05 and claim_08 took 4, and the other five took 2 (summary.md). A small fixed cap would have cut some of these off before end_turn, so they would have ended without a classification or terminal action. The sibling tests (test_no_string_membership_against_text_in_loop and test_stop_reason_is_loop_control) guard the same idea: stop_reason, not the model's text, controls the loop.


3. **Tool design.** I compared route_to_adjuster and escalate_to_human (claims_intake/tools.py). Both are marked "TERMINAL TOOL" and both take a summary or reason, so their inputs overlap. The descriptions separate them by when to use them: escalate_to_human says to use it when "classification confidence is below 0.6 even after clarification", while route_to_adjuster says to call it exactly once when confidence is "at least" the threshold. escalate_to_human also takes a reason from a fixed list (unresolved_ambiguity | low_confidence | missing_facts | policy_dispute | other). Structured output let the harness act on the failure: each record in runs/20261021_201310/escalations.jsonl has a root_cause ("no terminal tool called before end_turn"), candidate_claim_types, the case_facts collected so far, a recommended_action ("human review of the full transcript") and confidence 0.0. A generic error string could not tell a human reviewer what was already known about the claim.
  

4. **Your numbers.** claim_05_auto_collision took 4 turns, 15,557 input and 1,074 output tokens, estimated cost $0.0209 (summary.md). The README (line 65) says a good run shows 7 routed and 1 escalated at about $0.05 total on Haiku 4.5. My run showed 1 routed and 7 escalated at $0.1037 total, so it is about twice the README cost and the outcomes are almost reversed. Seven claims ended without calling route_to_adjuster or escalate_to_human, so the deterministic backstop escalated them. Six of these had expected=routed (claim_01, 02, 03, 04, 07, 08). I did not find out why the model stopped early, so I can only say the difference comes from model behaviour in my run, not from a crash: all 8 claims reached a terminal outcome, all 30 tests passed and the process produced runs/20261021_201310. The higher cost is probably because multi-turn claims resend the growing history each turn (claim_03 used 17,189 input tokens), but I have not verified this.
  

### System 2 — Context strategy

5. **The reduction.** In run 20261021-203300, budget.json shows baseline_tokens 38,708, assembled_tokens 16,843 and reduction_pct 56.49 (target was at least 50%). The "active" section dominates with 15,789 tokens, which is about 93.7% of the assembled context. It is kept verbatim because the open payment-method issue needs exact values: the eval questions ask for the failure code AVS_MISMATCH (Q3), the new card last-4 7782 (Q4) and the exact status token in_progress (Q6). A paraphrase would lose these.
   

6. **Summarize vs preserve.** The rule is that resolved segments are summarized, while the case-facts block and the active, unresolved segment are kept exact. In budget.json the per-section tokens are case_facts 204, resolved_refund 394, resolved_subscription 474 and active 15,789. The compression calls took in 12,334 tokens (refund) and 11,475 tokens (subscription) and produced 381 and 461 output tokens, so the resolved history is cut heavily. Resolved issues only need their outcome (for example the refund of 22.14 for ORD-77310 and the cancellation reason duplicate_charge). The active issue still needs exact tokens, so it is not compressed.


7. **Facts block.** eval.jsonl passed 6/6. In eval_control.jsonl (case facts removed) Q1 still passed, but Q6 regressed: the answer began "I cannot provide a status token because this issue has not been resolved...". The run printed "1/2 control questions regressed (≥1 required)". This shows that the exact structured token (in_progress) survives only because of the case-facts block, since the summaries and the context alone did not preserve it. The facts block is therefore carrying exact values, not the summary.

### System 3 — Claude Code config

8. **Path-scoped rules.** submission/system3/dot-claude/rules/tests.md has this frontmatter: `paths:` followed by `"**/*.test.tsx"` and `"**/*.test.ts"`. The other rules are scoped the same way: react.md uses "src/components/**/*" and "src/pages/**/*", and api.md uses "src/api/**/*". A glob rule applies to matching files wherever they sit in the repo, so one rule covers every co-located test file. A directory-level CLAUDE.md only applies under its own folder and would need to be repeated in each one. The project's own CLAUDE.md says it prefers `.claude/rules/` with glob `paths:` for cross-cutting conventions. The validator confirmed the config conforms (validator-output.txt: OK, exit code: 0), and 35 tests passed.


9. **Forked skill.** In .claude/skills/deploy-check/SKILL.md, line 4 is `context: fork` and lines 6-16 are `allowed-tools:` with Read, Grep, Glob, `Bash(git status:*)`, `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(git rev-parse:*)`, `Bash(git ls-files:*)`, `Bash(gh pr view:*)` and `Bash(gh pr checks:*)`. Forking runs the check in a separate sub-agent context, so its file reading does not fill the main session and only one pass/fail summary comes back. The allowlist has no Write or Edit tool and only read-only git and gh commands, so the check cannot change files. Without the fork the main context would absorb all the check output, and without the allowlist a pre-deploy check could modify the repo it is checking.
   
10. **Scope.** validator-output.txt only says "OK" and "exit code: 0", so I take the scope evidence from CLAUDE.md, which the validator checks. Project-level (CLAUDE.md line 11): `./CLAUDE.md`, `.claude/standards/` and `.claude/rules/`, which live in git and are shared with the team. For example, rules/api.md is shared this way, and commands/review.md (line 130) and the deploy-check skill (line 79) are described as project-scoped. User-level (line 12): `~/.claude/CLAUDE.md`, `~/.claude/commands/` and `~/.claude/skills/`, which stay on one person's machine and are not shared via version control, for example a personal commit-message style.
   

### System 4 — Orchestration

11. **Push work down.** Shift C printed "shift C: 17 new defects" (system4/run-output.txt) from a warm tier that holds 40 defects in total (fixtures/defects.json has 40 and the `defects` table in warm.sqlite has 40 rows). The query is `defects_since` in shift_monitor/warm.py: `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, called from pipeline.py (line 95, limit=50). It is backed by the indexes idx_defects_ts and idx_defects_shift_ts. SQL filters the rows before the model is called, so the model only sees the 17 new defects and never the full history. That matters because the table grows every shift while the model's context does not.
   

12. **Crash recovery.** shift_monitor/recovery.py sets `STALE_RESUME_THRESHOLD_MINUTES = 30` and `decide(state, now)` returns "fresh" if there is no state or no last step, "resume" if the last step is within 30 minutes, and "fresh" otherwise. The docstring explains that 30 minutes is about 1/16 of an 8-hour shift cycle. I did not trigger a crash in my run, so this comes from reading the code, not from an artifact. A fresh start with an injected summary can be more reliable because an old partial session may carry stale or half-finished context, while a fresh session starts clean from the small hot state.
    

13. **Small state.** My hot_state.json is 965 bytes (system4/hot-state-size.txt), about 19% of the roughly 5 KB (5,120 byte) budget. Before the shift neither hot_state.json nor shift_scratchpad.jsonl existed (system4/before-state.txt), and after it shift_scratchpad.jsonl had 1 line (323 bytes). Because this system starts a fresh session every shift indefinitely, the hot state is loaded into every session. If it grew each shift, cost would rise every shift and it would eventually crowd out the context window.
   

---

## Part 2 — Synthesis

14. **Three layers.**
    → Model: claude-haiku-4-5-20251001, which chose the tool calls in System 1 (summary.md) and answered the evals in System 2 (eval.jsonl).
    → Harness: claims_intake/loop.py and tools.py (System 1), with the deterministic backstop in run.py, and the .claude/ rules and skill in System 3 (dot-claude/rules, skills/deploy-check/SKILL.md). These decide how the model runs and what it may touch.
    → Orchestration: shift_monitor/pipeline.py and recovery.py with hot_state.json (965 bytes) and shift_scratchpad.jsonl (System 4). They manage state, filtering and sessions across shifts, outside any single conversation.

15. **Deterministic vs prompt.** Code-guaranteed: the backstop in claims_intake/run.py (line 97) escalated 7 of 8 claims with reason "deterministic backstop" when the model stopped without a terminal tool. The deploy-check skill's read-only allowlist and the hot_state.json size budget are also enforced in code or config. Prompt-guided: the tool descriptions in tools.py ask the model to call route_to_adjuster or escalate_to_human, and in my run the model skipped this on 7 of 8 claims. Use code where failure is unacceptable (termination, permissions, size limits) and prompts where judgement is needed (classification and confidence).
    →

16. **Context, two faces.** System 2 manages context inside one conversation: 38,708 tokens became 16,843 (56.49%), and exact facts stayed exact (case_facts 204 tokens, active 15,789). System 4 manages it across sessions: a new session gets a 965-byte hot_state.json and only the 17 SQL-filtered defects out of 40 stored. The principle is the same: give the model only what it needs and keep exact facts exact. System 2 does it by summarizing resolved text and keeping a facts block; System 4 does it by tiered storage and pre-filtering in SQL.
    →

17. **Reliability you can't see in one run.** test_no_integer_literal_iteration_cap_in_loop guarantees that no fixed turn cap exists in the loop. My 8 claims all finished in 5 turns or fewer, so a cap of 5 or 6 might never have shown up in my single run. A longer real claim would hit it and be cut off without a terminal action. The test catches this before shipping.
    →

18. **Blast radius.** For System 1, a misbehaving agent could mis-route a claim into queues/*.jsonl or leave it with no outcome. The enforcement points are the two terminal tools, the UnexpectedStopReason raise in loop.py (line 131) and the backstop in run.py. In my run the backstop turned 7 unfinished claims into escalations for human review (escalations.jsonl) instead of letting them disappear. The kill switch is the max_wall_clock_s limit passed through run.py (lines 52, 64, 244) and the escalation path to a human. The state is limited to one timestamped run folder (runs/20261021_201310), so a bad run does not touch other runs.
    →

---

## Part 3 — Honest assessment

19. **What broke.** My first System 1 run failed with a 401 "invalid x-api-key" (request_id req_011CgFxFJLVFKy1HxXWoS2Hk). The Vocareum voc- key was being sent to api.anthropic.com because ANTHROPIC_BASE_URL was not set. A second attempt also failed because the pasted key had the wrong length (79 instead of 49). I fixed it by setting ANTHROPIC_BASE_URL=https://claude.vocareum.com and pasting the key again, and a small curl call returned stop_reason end_turn before I re-ran. Other issues were a leftover ANTHROPIC_MODEL variable, which I unset before System 2, and a missing data/ folder, which I created before seeding System 4. System 2 also first reported 28 passed, 2 skipped, and after its run it reported 30 passed.
    →

20. **What you'd change.** In runs/20261021_201310, 7 of 8 claims ended through the deterministic backstop (root_cause "no terminal tool called before end_turn"), including 6 where expected=routed. I would make the loop give the model one more chance, for example a short nudge or forcing a terminal tool call, before falling back to escalation. That keeps the backstop as a safety net instead of the normal path. I would also read the traces to find why the model stopped early before changing the prompt.
    →






_______________________________________________________________________________
# Reflection Brief — Harness Engineering Capstone

**Name:**
**Date:**

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s):
- OS / Python:
- Approx. API spend:

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → 

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → 

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → 

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → 

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → 

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → 

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → 

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → 

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → 

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → 

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → 

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → 

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → 

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model:
    → Harness:
    → Orchestration:

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → 

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → 

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → 

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → 

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → 

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → 
