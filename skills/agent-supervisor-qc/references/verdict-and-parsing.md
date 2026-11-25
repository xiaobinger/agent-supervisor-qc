# Verdict Protocol & Parsing Rules

Supervisors are LLMs — they will occasionally drift from the required output format.
The orchestrator must therefore be **strict in what it asks for, tolerant in what it accepts,
and conservative in what it defaults to**.

---

## 1. Verdict vocabularies

| Mode | Pass | Fail / Rework | Escalate |
|---|---|---|---|
| Chat response review | `approved` | `retry` | `flag_human` |
| Acceptance review | `【验收通过】` | `【需要返工】` | unparseable → pending human; max-rounds force-close is NOT acceptance |
| Risky-action gate | `{"approved": true}` | `{"approved": false}` | default-false on failure |
| Stuck-task watchdog | `【重试处理】` | `【放弃重试】` | abandon → human assignment |
| Pre-retry evaluation | `【已执行完成，需要验收】` | `【需要补充执行】` | undecidable → normal processing |

Aliases accepted during parsing: `flag` → `flag_human`; keyword fallbacks below.

## 2. Score extraction

Regex: `评分\s*[:：]\s*(-?\d{1,3})` → clamp to [0, 100]. Absent → `null` (do not invent a score).

## 3. Parsing algorithm (per mode)

### 3.0 The no-valid-verdict principle (applies to every mode)

A verdict counts ONLY when extracted from a complete, well-formed, enumerable output.
Empty output, truncated JSON, unknown prose, and malformed formats are **not verdicts**:

| Mode | No valid verdict resolves to |
|---|---|
| Chat response review | `flag_human` (pending human) |
| Risky-action gate | `approved=false` (default deny, human confirmation required) |
| Acceptance review | do not close / do not approve; attach the raw opinion for humans |
| Watchdog / pre-retry evaluation | skip this cycle, leave state untouched, retry next tick |

**No path may ever read an empty, truncated, unknown, or malformed output as an approval.**

### 3.1 First-JSON extraction (both JSON modes)

Do **not** use a greedy regex such as `\{[\s\S]*\}`. Starting at the first `{` it spans all the
way to the LAST `}` in the text, so brace-bearing prose after the JSON (or a second JSON object)
gets swallowed and `json.loads` fails — contradicting the "first JSON object" promise. A
non-greedy `\{[\s\S]*?\}` is equally wrong: it stops at the first `}`, breaking on nested objects.

Use a deterministic balanced-brace scan instead:

1. Find the first `{`. If none exists → no candidate → parse failure.
2. Scan forward with `depth = 1`. Track string literals: an unescaped `"` toggles in-string
   state; a `\` skips the next character. Count `{` / `}` only while outside strings.
3. When `depth` returns to 0, the candidate is the substring from the opening brace to here.
4. `json.loads(candidate)` — success → done. Failure → resume from the next `{` after the
   candidate's start and repeat.
5. If `depth` never returns to 0 (truncated output) or no candidate parses → **parse failure**.

### 3.2 Response review (chat)

1. Extract the first JSON object per 3.1; `json.loads`; normalize the `verdict` field
   (lowercase, strip, alias-map: `flag` → `flag_human`).
2. Well-formed JSON but missing/invalid `verdict` value → parse failure.
3. Parse failure → ordered keyword fallback (first match wins):
   1. `人工复核` / `转人工` / `需要人工` / `flag_human` → `flag_human`
   2. `retry` / `重新生成` / `重新回答` / `不通过` / `不合格` / `需要返工` → `retry`
   3. **anything else (including empty output) → `flag_human`**
4. The keyword table contains **no rule that can yield `approved`** — `approved` is reachable
   only from a well-formed JSON `verdict` field. An empty output means the supervisor produced
   no review at all; that is never evidence of quality (see failure-modes.md #8).

### 3.3 Risky-action gate

1. Extract the first JSON object per 3.1; `json.loads`; read the `approved` field.
2. Well-formed JSON with a boolean `approved` → use that value. This is the **only** path to `true`.
3. Everything else — truncated JSON (e.g. `{"approved": false`), missing or non-boolean field,
   malformed JSON, unknown prose, empty output — → `false` ("assessment failed / no valid
   verdict, human confirmation required").
4. Keyword matching may enrich the `feedback` text but **never flips the verdict**.
   Rationale: substring keywords are negation-blind — bare `approved` also occurs inside a
   truncated `{"approved": false`, `通过` inside `不通过`, `安全` inside `不安全`.

### 3.4 Marker modes (acceptance, watchdog, pre-retry)

Primary check: `startswith(marker)`.

For acceptance review, additionally apply **broad keyword matching** because models paraphrase:

```text
PASS   = 验收通过 | 通过验收 | 【通过】 | 验收结论：通过 | 同意关闭 | 可以关闭 | 应关闭
       | 同意结束 | 可以结束 | 应结束
REJECT = 验收不通过 | 不通过 | 【不通过】 | 不应结束 | 不应关闭 | 不同意关闭
       | 需要返工 | 补充处理 | 任务遗漏 | 执行不完整 | 结果不完整
```

**REJECT overrides PASS** when both appear — a sentence like "验收通过的标准未达到，验收不通过"
must resolve to rejection.

Unparseable output → conservative: do NOT close/approve. Record the raw opinion for humans.

## 4. Verdict input/output table (deterministic spec)

The conformance contract for §3: every listed input must resolve to exactly the stated output.
It doubles as the test matrix for implementations.

### 4.1 Chat response review

| Input (supervisor output) | Resolved verdict | Notes |
|---|---|---|
| `{"verdict": "approved", "feedback": "...", "suggestions": [...]}` | `approved` | Only pass path; suggestions kept (≤4) |
| `{"verdict": "retry", "feedback": "..."}` | `retry` | Triggers regeneration; suggestions forced `[]` |
| `{"verdict": "flag_human", ...}` or text with `人工复核` / `转人工` | `flag_human` | Route to human |
| Truncated pass: `{"verdict": "approved` | `flag_human` | Parse failure → pending human; truncation ≠ approval |
| Truncated rework: `{"verdict": "retry` | `retry` | Keyword hit, non-approving direction hence safe; no hit → `flag_human` |
| Empty output (empty / whitespace-only) | `flag_human` | Log "supervisor produced no output"; never `approved` |
| Unknown text (e.g. "我觉得还可以") | `flag_human` | No valid verdict → pending human |
| Malformed format (single-quoted JSON, bare `verdict: approved`, multiple broken objects) | `flag_human` | Parse failure → pending human |

### 4.2 Risky-action gate

| Input (supervisor output) | Resolved `approved` | Notes |
|---|---|---|
| `{"approved": true, "feedback": "..."}` | `true` | Only pass path: complete well-formed JSON with boolean true |
| `{"approved": false, "feedback": "..."}` | `false` | Reject with reasons |
| Truncated rejection: `{"approved": false` | `false` | Parse failure → default deny + human confirmation |
| Truncated approval: `{"approved": true` | `false` | Parse failure → default deny; approval requires complete JSON |
| Empty output | `false` | Silence ≠ consent |
| Unknown text / malformed format | `false` | Default deny |
| Prose rejection ("风险较高，不建议执行") | `false` | Same direction as explicit rejection |
| Prose approval ("可以执行") | `false` | No JSON boolean → no valid verdict → default deny + human confirmation |

## 5. Suggestions cleanup (response review)

- Only keep suggestions when verdict = `approved`; force `[]` otherwise.
- Strip leading bullets/numbering: `^\s*(?:[-•*]|\d+[.、)）])\s*`
- Max 4 items, each truncated to 30 chars.

## 6. Model-call resilience

- Call with `temperature` 0.1–0.2, small `max_tokens` (512 for gates, 1024 for reviews).
- Iterate an ordered list of model configs (failover); log each failure and try the next.
- All configs failed → apply the no-valid-verdict principle (§3.0): gates → `approved=false`;
  reviews → `flag_human` ("needs human verification"); acceptance → do not close, pending human;
  watchdog/pre-retry → skip this cycle, leave state untouched, retry on next tick.
  **Never auto-approve on outage.**

## 7. Payload hygiene

| Field | Limit |
|---|---|
| User question | 4,000 chars (2,000 for gates) |
| Executor reply | 6,000 chars |
| Tool log | 10 entries × 600 chars |
| Gate params JSON | 2,000 chars |
| Historical comments | last 5 × 200 chars |

Always append an explicit "truncated" note (`…（已截断）`) so the supervisor does not mistake
truncation for missing execution.
