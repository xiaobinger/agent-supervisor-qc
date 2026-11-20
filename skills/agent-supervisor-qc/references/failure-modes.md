# Production Failure Modes & Fixes

Hard-won lessons from running executor–supervisor loops in production. Each entry:
**symptom → root cause → fix**. Treat these as design requirements, not optional hardening.

---

## 1. Executor verbally complies but does nothing

**Symptom**: supervisor issues REWORK feedback; executor replies "已按反馈完成" without
calling any tool; next review round passes nothing real.

**Root cause**: feedback was injected as a soft suggestion; the executor optimized for a
plausible-sounding reply.

**Fix**:
- Inject feedback with an explicit **highest-priority enforcement block**: every item must be
  implemented; tool-requiring items must call tools; impossible items must be explained per
  item; the final reply must map item-by-item to the feedback.
- **Zero-tool forced retry**: if the executor's first rework response contains no tool calls,
  the orchestrator appends a system-enforcement message ("you have not called any tool to
  implement the feedback; call the appropriate tools NOW unless an item genuinely needs none")
  and runs one more round before accepting a text-only answer.

## 2. Dual-implementation drift (rework path is weaker than the main path)

**Symptom**: first-pass execution has full capabilities (tools, domain rules, memory), but
rework rounds run through a simplified code path — feedback is "handled" without the tools or
context needed to actually do it.

**Fix**: rework rounds must call the **same unified executor loop** as first-pass execution —
same tool set, same context injection, same multi-round tool calling. Never maintain a
separate "retry implementation".

## 3. Context wiped on restart

**Symptom**: task restarted/reassigned; executor starts from a blank slate, repeats work,
contradicts earlier results, or the supervisor re-litigates settled points.

**Fix**: on every rework/restart, inject the **full history**: last processing result,
collaboration log (executor + supervisor entries), and the supervisor's prior feedback.
Run **pre-retry state evaluation** (template 5) first: already-complete work jumps straight
to acceptance; only genuine gaps get supplementary-execution feedback.

## 4. Supervisor misjudges "tool not executed" from truncated logs

**Symptom**: supervisor sees an excerpted tool log and declares the executor never ran the
tool, demanding pointless rework.

**Fix**: state explicitly in the review prompt that tool results are excerpted and possibly
truncated, and that truncation must not be read as non-execution. Judge execution from call
records (name + params + status), not from output completeness.

## 5. Misleading success on empty results

**Symptom**: a query/export returns 0 rows; the executor reports "done, file generated" and
the supervisor passes it; the user gets a broken download or an empty file with no warning.

**Fix**: supervisor verifies **actual data**, not success flags. Zero-row results must be
surfaced to the user as "no data" — and never logged or announced as "file generated".
A success status code is not evidence of useful output.

## 6. Verdict format drift

**Symptom**: supervisor outputs prose ("我认为这个结果基本可以满足要求……") instead of the
required marker/JSON; the orchestrator can't parse it; the task hangs.

**Fix**: tolerant parsing with **conservative defaults** — see verdict-and-parsing.md:
first-JSON extraction, keyword fallback tables, REJECT-overrides-PASS, and
unparseable → do not approve, record raw output, escalate to human.

## 7. Infinite rework loops

**Symptom**: supervisor keeps finding nitpicks; executor keeps reworking; token spend explodes.

**Fix**: hard cap on collaboration rounds (default 3, configurable 1–20). At the cap:
force-close with the supervisor's final score and opinion attached, or escalate to a human.
Score formula includes a per-rework-round deduction (−5, floor 40) so endless loops visibly
degrade the score.

## 8. Supervisor service outage silently approves (or silently kills) work

**Symptom**: review call throws; naive code treats exception as pass (or drops the task).

**Fix**:
- Model **failover**: iterate an ordered config list, log each failure, try the next.
- All failed → fail-closed for gates (`approved=false`, human confirmation required);
  "needs human verification" for reviews. **Never auto-approve on outage.**
- Background supervisors (watchdog, auto-acceptance) must run in isolated threads with
  per-item try/except — one bad task must not kill the monitor loop.
- Beware hidden death paths: e.g. re-initializing the app inside background threads can
  race and silently kill completion callbacks — no logs, no notifications. Reuse the
  existing app context; wrap every callback chain.

## 9. Self-supervision

**Symptom**: executor and supervisor resolve to the same agent; reviews are meaningless
rubber stamps.

**Fix**: resolution explicitly excludes self — executor-bound supervisor → global default →
any active supervisor; if the winner equals the executor, supervision is disabled (degrade
gracefully to single-agent execution with a logged warning).

## 10. Ignored user interrupts

**Symptom**: user says "停，别查了" mid-execution; the final reply ignores it; supervisor
approves because the original question was answered.

**Fix**: inject the interrupt log into the review prompt with an explicit checklist: stop
commands honored? new instructions incorporated? Fully ignored interrupts → `retry`.

## 11. Standard mismatch between executor and supervisor

**Symptom**: supervisor rejects work for violating a domain rule the executor never saw
(or vice versa); the loop oscillates.

**Fix**: inject the **same** domain rules/memory/skills context into both agents' prompts.
The supervisor judges by exactly the standard the executor executed by. Custom per-task
review rules (`review_rules`) are appended to the supervisor prompt with "must strictly
comply" priority.

## 12. Chat-level reply quality regressions slip through

**Symptom**: task-level gates pass, but conversational replies are arrogant, fabricate
numbers, or misreport tool errors.

**Fix**: run a lightweight **per-turn response review** (template 1) with attitude and
consistency dimensions, capped review rounds (`max_supervisor_rounds`), and `flag_human`
for high-risk content — a hybrid of AI supervision with human backstop.
