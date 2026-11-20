# Verdict Protocol & Parsing Rules

Supervisors are LLMs — they will occasionally drift from the required output format.
The orchestrator must therefore be **strict in what it asks for, tolerant in what it accepts,
and conservative in what it defaults to**.

---

## 1. Verdict vocabularies

| Mode | Pass | Fail / Rework | Escalate |
|---|---|---|---|
| Chat response review | `approved` | `retry` | `flag_human` |
| Acceptance review | `【验收通过】` | `【需要返工】` | (human via max-rounds force-close) |
| Risky-action gate | `{"approved": true}` | `{"approved": false}` | default-false on failure |
| Stuck-task watchdog | `【重试处理】` | `【放弃重试】` | abandon → human assignment |
| Pre-retry evaluation | `【已执行完成，需要验收】` | `【需要补充执行】` | undecidable → normal processing |

Aliases accepted during parsing: `flag` → `flag_human`; keyword fallbacks below.

## 2. Score extraction

Regex: `评分\s*[:：]\s*(-?\d{1,3})` → clamp to [0, 100]. Absent → `null` (do not invent a score).

## 3. Parsing algorithm (per mode)

### JSON modes (response review, risky-action gate)

1. Extract the **first** `{...}` block via regex `\{[\s\S]*\}` (greedy to last brace).
2. `json.loads` it. On success, normalize the verdict field (lowercase, strip, alias-map).
3. On JSON failure → keyword fallback:
   - Response review: contains `retry` / `重新生成` / `不通过` / `不合格` → `retry`;
     contains `人工` / `人工复核` / `flag_human` → `flag_human`; else → `approved`.
   - Risky-action gate: contains `rejected` / `拒绝` / `风险` / `不建议` / `禁止` / `危险`
     → `false`; contains `approved` / `可以执行` / `安全` / `通过` / `允许` → `true`;
     else → `false` (default deny).
4. Empty output → response review: `approved` with empty feedback (don't block the user on a
   supervisor outage); risky-action gate: `false` (never auto-approve on silence).

### Marker modes (acceptance, watchdog, pre-retry)

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

## 4. Suggestions cleanup (response review)

- Only keep suggestions when verdict = `approved`; force `[]` otherwise.
- Strip leading bullets/numbering: `^\s*(?:[-•*]|\d+[.、)）])\s*`
- Max 4 items, each truncated to 30 chars.

## 5. Model-call resilience

- Call with `temperature` 0.1–0.2, small `max_tokens` (512 for gates, 1024 for reviews).
- Iterate an ordered list of model configs (failover); log each failure and try the next.
- All configs failed:
  - Acceptance/response review → verdict = fail-closed for gates, "needs human verification"
    for reviews. Never auto-approve.
  - Watchdog/pre-retry → skip this cycle, leave state untouched, retry on next tick.

## 6. Payload hygiene

| Field | Limit |
|---|---|
| User question | 4,000 chars (2,000 for gates) |
| Executor reply | 6,000 chars |
| Tool log | 10 entries × 600 chars |
| Gate params JSON | 2,000 chars |
| Historical comments | last 5 × 200 chars |

Always append an explicit "truncated" note (`…（已截断）`) so the supervisor does not mistake
truncation for missing execution.
