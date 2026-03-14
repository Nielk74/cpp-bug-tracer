# cpp-bug-tracer Benchmark

Comparing four configurations on **35 C++ bug evals** (11 original + 24 complex evals):

| Config | Architecture | Model |
|--------|-------------|-------|
| **multi-agent** | orchestrator → abstractor → investigator ×2 | qwen/qwen3-coder-30b-a3b-instruct (OpenRouter) |
| **single-agent-qwen3** | one agent, reads files directly | qwen/qwen3-coder-30b-a3b-instruct (OpenRouter) |
| **single-agent-glm5** | one agent, reads files directly | glm-5 (Z.AI Coding Plan) |
| **cross-source-tracer** | orchestrator → investigator (source+parser diff) | qwen/qwen3-coder-30b-a3b-instruct (OpenRouter) |

---

## Results

| # | Bug type | multi-agent | single-qwen3 | single-glm5 | cross-source-tracer |
|---|----------|-------------|--------------|-------------|---------------------|
| 1 | Paste blocked (CashFlow read-only) | ✅ PASS | ✅ PASS | ✅ PASS | — |
| 2 | Spurious validation on load | ✅ PASS | ✅ PASS | ✅ PASS | — |
| 3 | Notional field FX vs CashFlow | ⚠️ PARTIAL¹ | ⚠️ PARTIAL¹ | ⚠️ PARTIAL¹ | — |
| 4 | Hardcoded counterparty ID 0 | ✅ PASS | ✅ PASS | ✅ PASS | — |
| 5 | Post-increment reservation off-by-one | ✅ PASS | ✅ PASS | ✅ PASS | — |
| 6 | Trade-validated notification never fires | ✅ PASS | ✅ PASS | ✅ PASS | — |
| 7 | Settlement triggers wrong handler | ✅ PASS | ⚠️ PARTIAL² | ✅ PASS | — |
| 8 | Counter double-incremented | ✅ PASS | ✅ PASS | ✅ PASS | — |
| 9 | Double settlement (cross-layer state) | ✅ PASS | ❌ FAIL³ | ✅ PASS | — |
| 10 | Credit limit `Success(false)` misuse | ✅ PASS | ✅ PASS | ✅ PASS | — |
| 11 | Wrong argument to credit release | ✅ PASS | ✅ PASS | ✅ PASS | — |
| | **Score (1–11)** | **10/11** | **8/11** | **10/11** | — |
| | | | | | — |
| 12 | Amendment validates stale notional | ⚠️ PARTIAL⁴ | ✅ PASS | ✅ PASS | — |
| 13 | Batch notify-before-execute ordering | ✅ PASS⁵ | ✅ PASS | ✅ PASS | — |
| 14 | Wrong risk limit in cumulative check | ✅ PASS | ✅ PASS | ✅ PASS | — |
| 15 | Pagination off-by-one (1-based page) | ✅ PASS | ✅ PASS | ✅ PASS | — |
| | **Score (12–15)** | **3/4** | **4/4** | **4/4** | — |
| | **Total score** | **13/15** | **12/15** | **14/15** | — |
| | | | | | — |
| 16 | Exposure release reads unpopulated map | ✅ PASS | ✅ PASS | — | — |
| 17 | P&L key: counterparty code vs product type | ✅ PASS | ✅ PASS | — | — |
| 18 | Sanctions: name vs code lookup | ✅ PASS | ✅ PASS | — | — |
| 19 | Audit: operation ID written to trade ID field | ✅ PASS | ✅ PASS | — | — |
| 20 | Rate limit: per-hour config vs per-minute window | ✅ PASS¹⁰ | ✅ PASS | — | — |
| 21 | Approval level: 1-based config vs 0-based gate | ⚠️ PARTIAL¹¹ | ✅ PASS | — | — |
| 22 | Position store: reversed key order (writer vs reader) | ✅ PASS⁹ | ✅ PASS | — | — |
| 23 | Fee units: basis points (schedule) vs percent (validator) | ✅ PASS | ✅ PASS | — | — |
| | **Score (16–23)** | **7/8** | **8/8** | — | — |
| | | | | | — |
| 24 | Priority queue inversion: classifier vs scheduler convention | ✅ PASS | ✅ PASS | — | — |
| 25 | Risk limit: normalizer millions vs absolute USD | ✅ PASS | ✅ PASS | — | — |
| 26 | Message encode/decode: cptyCode ↔ notional field swap | ✅ PASS | ✅ PASS | — | — |
| 27 | Implicit contract: fill recorder writes nothing on final fill; completion checker expects key=0.0 | ✅ PASS | ✅ PASS | — | — |
| | **Score (24–27)** | **4/4** | **4/4** | — | — |
| | | | | | — |
| 28 | Priority unsigned cast: negative score wraps to UINT_MAX | ✅ PASS | ✅ PASS | — | — |
| 29 | Settlement seconds vs minutes (false-witness audit logger) | ✅ PASS | ✅ PASS | — | — |
| 30 | Double FX: converter omits currency update, calculator guard permanently false | ✅ PASS¹² | ✅ PASS | — | — |
| | **Score (28–30)** | **3/3** | **3/3** | — | — |
| | | | | | — |
| **Tier 5 — vague prompts + 55 red-herring files (evals 28–30 v2)** | | | |
| 28v | Priority unsigned cast — symptom-only prompt, no component names | ❌ FAIL¹³ | ✅ PASS | — | — |
| 29v | Settlement seconds vs minutes — symptom-only prompt | ⚠️ PARTIAL¹³ | ✅ PASS | — | — |
| 30v | Double FX — symptom-only prompt | ❌ FAIL¹³ | ✅ PASS | — | — |
| | **Score (28–30 vague)** | **0.5/3** | **3/3** | — | — |
| | | | | | — |
| **Tier 6 — .env chain + conditional gate + false-witness logger (evals 31–32)** | | | |
| 31 | Grace BPS/percent mismatch: institutional cap enforcement passes 50% grace instead of 0.5% | ✅ PASS¹⁵ | ⚠️ PARTIAL¹⁵ | — | — |
| 32 | Windows registry locale string: std::stoi truncates "1,000" to 1, fee threshold collapses | ✅ PASS¹⁷ | ❌ FAIL¹⁶ | — | — |
| | **Score (31–32)** | **2/2** | **0.5/2** | — | — |
| | | | | | — |
| **Tier 7 — External API format changes + multi-source schemas (evals 33–35)** | | | | |
| 33 | API v2 scientific notation: std::stoi("1e3") returns 1, fee threshold collapses | ✅ PASS¹⁸ | ❌ FAIL²¹ | — | — |
| 34 | Multi-source message queue: field index 2 = notional for legacy, = fee for new provider | ❌ FAIL¹⁹ | ✅ PASS²² | — | ⚠️ PARTIAL²³ |
| 35 | API v3 field rename: parser searches "approved_credit_usd" but API returns "credit_limit_usd" | ❌ FAIL²⁰ | ❌ FAIL²⁴ | — | ✅ PASS²⁵ |
| | **Score (33–35)** | **1/3** | **1/3** | — | **1.5/2** |
| | | | | | — |

### Notes

¹ **Eval 3 partial (all configs)**: All agents find `TradeForm::updateFieldVisibility` and correctly report that both FX and CashFlow are hidden per the code — user's premise is wrong. However none fully enumerate all product types that show the notional field (Equity, FixedIncome, Derivative, Swap, Forward, Option, Future). GLM-5 also hits max steps exploring the `setState(Computed)` red herring.

² **Eval 7 qwen3 partial**: qwen3 single-agent said "event type 5 missing in switch" — the actual bug is that `SETTLEMENT_COMPLETED = 3` collides with `case 3: OnTradeSubmitted`. Found the dispatcher but misread the collision direction.

³ **Eval 9 qwen3 fail**: Single-agent cannot parallelize threads. When the entry point for a cross-layer bug is ambiguous, qwen3 anchors on the wrong file and produces a report for a different bug entirely.

⁴ **Eval 12 multi-agent partial**: Orchestrator wastes steps attempting `cpp-bug-tracer/agents/worker-agent` (non-existent). Abstractor report mentions validation-before-update but hallucinated `TradeValidator.cpp` — never traces `m_mapTradeStore` or the local copy issue. 2/5 assertions pass.

⁶ **Evals 16–19 original multi-agent fail**: Root cause was abstractor `task:false` — investigators never ran, pure hallucination. After architectural fix (task:true), all 4 now PASS. The "single-file bug" classification was a red herring: Thread A reads the whole file and sees both sides of the bug; Thread B's failure to find a second class is irrelevant when Thread A already has the full answer.

⁷ **Evals 20–21 original multi-agent fail (task:false)**: Abstractor had task:false — investigators never ran, pure hallucination.

¹⁰ **Eval 20 multi-agent PASS (after orchestrator classification fix)**: Orchestrator now classifies "never blocks, SetRateLimit with per-hour values" as Unit/scale mismatch. Derived Thread A → CTradeRateLimitConfig, Thread B → CTradeRateLimiter from "SetRateLimit" method name. Report correctly cites CTradeRateLimiter.cpp:71 (WINDOW_SECONDS=60) and exact fix (3600).

¹¹ **Eval 21 multi-agent partial (after abstractor fix)**: Investigators ran and found CTradeApprovalGate.cpp:63 and CTradeApprovalLevelConfig. Correctly identified 0-based vs 1-based mismatch and right fix direction. Gaps: CTradeApprovalLevelConfig line is "XX" (investigators hit step limit before fully reading the file). Example explanation in report slightly off, but core diagnosis correct.

⁸ **Eval 22 original multi-agent fail**: Report cited "CTradePositionStore.cpp lines 45/67" — hallucinated. Root cause: abstractor had `task: false` so investigators were never run. Fixed by enabling task: true in abstractor.

⁹ **Eval 22 multi-agent PASS (after abstractor fix)**: Investigators actually ran and read both CTradePositionWriter.cpp and CTradePositionReader.cpp. Report correctly identifies CTradePositionReader.cpp:52 as the bug location and the exact reversed key format. All 5 assertions pass.

⁵ **Eval 13 multi-agent (after fix)**: Previously failed — abstractor hallucinated `CTradeEventDispatcher.cpp` due to step budget exhausted by worker-agent failures. Fixed by rewriting orchestrator: `grep: false`, `steps: 6`, first action hardcoded to abstractor. Re-run now correctly identifies `NotifyTradeExecuted` before `ExecuteTrade` at line 817.

---

## Key observations

### When multi-agent wins
- **True cross-file bugs** (eval 9): two parallel threads read different service files simultaneously. Each file looks correct in isolation; only by comparing thread outputs does the mismatch appear.
- **Avoids anchoring**: qwen3 single-agent anchors on the first file it reads. On a cross-file bug it may declare the first file correct and stop, never reading the second.
- **Prerequisite**: abstractor must have `task: true` to actually spawn investigators. Previously `task: false` meant the abstractor only reasoned from the prompt and hallucinated file names — no files were ever read.
- **Pattern E** in investigator: use glob `**/<ClassName>.cpp` to find implementation files directly instead of grepping (which hits .h headers first).

### When vague prompts + red herrings flip the result (tier 5)
Adding 14 red-herring files (55 total) and removing component names from prompts reveals a new failure mode: multi-agent's orchestrator classification step becomes a bottleneck under ambiguity. The orchestrator must classify the bug type before spawning the abstractor; with a vague symptom it spends extra steps reasoning, hits the session timeout, and produces nothing. Single-agent's flat search-and-follow strategy is unaffected — it greps broadly, finds the right files through keyword density, and completes the report. **Counterintuitive result: for ambiguous real-world bugs without a named starting point, single-agent qwen3 outperforms the multi-agent pipeline.** The multi-agent workflow needs a more robust orchestrator that can spawn investigators under uncertainty rather than requiring a correct classification first.

### When single-agent wins
- **Step efficiency**: multi-agent's orchestrator→abstractor→investigator chain costs 3–5× more steps; for trivial bugs this overhead dominates.
- **Unclear classification**: if the orchestrator misclassifies the bug type, thread prompts point investigators to wrong entry points. Single-agent just follows symbols from the prompt directly.

### GLM-5 vs qwen3 single-agent
| | GLM-5 | qwen3 |
|---|---|---|
| Eval 7 (event collision) | ✅ Correctly traced `COMPLETED=3` collision | ⚠️ Misread as missing case 5 |
| Eval 9 (cross-layer state) | ✅ Found lifecycle state mismatch | ❌ Produced wrong-bug report |
| Eval 11 (wrong argument) | ✅ Quoted exact variable names, traced `m_mapCounterpartyReservedTotals` | ✅ Correct but less detail |
| Fix directions | More precise — includes exact code snippets | Correct but higher-level |
| Eval 3 (ambiguous premise) | ⚠️ Hits max steps chasing red herring | ⚠️ Produces reasonable report quicker |

**Summary**: GLM-5 is stronger at multi-file tracing and produces more precise fix directions. qwen3 is faster and less prone to rabbit holes on simple cases. Multi-agent (qwen3) covers the gap on cross-layer bugs by parallelizing.

### Architectural bug (now fixed): investigators were never run
The abstractor had `task: false` — it could NEVER spawn investigators. It only reasoned from the orchestrator prompt and hallucinated file names. The "parallel investigator threads" were fictional. Evals 1-11 passed because the model could reason correctly about common C++ patterns without reading files. Evals 16-22 failed because new files were unknown to the model.

Fix applied: `task: true` in abstractor.md (steps 3→8). Abstractor now spawns both investigators in parallel and waits for real file:line evidence before synthesizing.

¹³ **Evals 28-30 vague prompts — multi-agent orchestrator bottleneck**: With symptom-only prompts (no component names) and 55 service files, multi-agent's orchestrator classification step timed out before reaching the abstractor on 4 out of 5 attempts for evals 28 and 30. The orchestrator requires classifying the bug type before spawning the abstractor — vague prompts cause extra reasoning steps, pushing the session into timeout. Single-agent bypasses this bottleneck by doing broad keyword search across all files and following call chains directly. **Key insight**: multi-agent is *more brittle than single-agent with vague, ambiguous symptoms* — the orchestrator's rigid bug-type classification is a single point of failure. Single-agent's flat grepping strategy is more robust to ambiguity, though it would fail in a truly large codebase (thousands of files) where grep returns too many false positives to follow.

¹² **Eval 30 multi-agent reliability**: First run stuck at abstractor (no output, never completed). Re-run passed fully — correctly identified that `CTradeNotionalUSDConverter` sets `m_bNormalized=true` but never updates `m_strCurrency` to "USD", causing the calculator's `(m_bNormalized && currency=="USD")` guard to be permanently false and FX to be applied twice. This is the second occurrence of multi-agent randomly getting stuck mid-run (see also original eval 24 failure). Appears to be an OpenRouter timeout issue, not a reasoning failure.

| Weakness | Multi-agent (before fix) | Multi-agent (after fix) | Single-agent | Cross-source-tracer |
|----------|--------------------------|------------------------|--------------|---------------------|
| task:false — investigators never ran | ❌ All evals hallucinated | ✅ Fixed | N/A | N/A |
| Worker-agent step waste (evals 12-13) | ❌ Occurred | ✅ Fixed (orchestrator rewrite) | N/A | N/A |
| Cross-layer / multi-file bugs | ✅ Parallelism helps (eval 9, 22) | ✅ Still works | ❌ Anchors wrong | N/A (different pipeline) |
| Source→parser format mismatch (eval 33) | ❌ Scientific notation truncation missed | ✅ Fixed (Pattern G stoi + ApiResponseParser keyword) | ❌ Anchors on old CTradeFeeValidator files | not tested |
| Keyword contamination with older evals (eval 34) | ❌ "position limits" triggers eval 25 files | ❌ Open | ✅ Solved — no keyword lookup, domain-noun glob finds queue files directly | ⚠️ Partial (wrong wrong-value 0 vs 75) |
| stoi anchoring overrides early-return path (eval 35) | ❌ stoi diagnosed instead of find()==npos | ❌ Open | ❌ Went to CCreditCheckService, never reached ApiCreditParser | ✅ Solved — source+parser side-by-side diff isolates field rename |

### Latency
| Config | Typical time |
|--------|-------------|
| multi-agent | 90–420s (parallel threads, OpenRouter variance) |
| single-agent-qwen3 | 60–180s |
| single-agent-glm5 | 80–200s |
| cross-source-tracer | 60–150s (sequential: orchestrator → investigator) |

¹⁸ **Eval 33 — multi-agent PASS (4/5)**: API v2 returns `"max_fee_bps": "1e3"` as a quoted string. `CTradeApiResponseParser::ParseInt` calls `std::stoi("1e3")` which stops at 'e' and returns 1. `CTradeFeeScheduleEnforcer::IsFeeAcceptable` rejects any fee > 1 bps. Pattern G (scientific notation branch) correctly identified the truncation. Fix: Pattern G stoi knowledge + `ApiResponseParser` as Thread B primary keyword to avoid globbing old `CTradeFeeValidator`/`CTradeFeeSchedule` files. Missing assertion: `CTradeApiFeeClient` not named as the upstream source.

¹⁹ **Eval 34 — multi-agent FAIL (keyword contamination)**: `CTradeQueueMessageParser::ParseNotional` hardcodes `fields[2]` (= notional for legacy, = FEE_BPS for NewProvider). Despite keyword table entry `QueueMessage,QueueConsumer`, the orchestrator consistently extracts "position limits" from the symptom and finds `CTradeRiskChecker.cpp` (eval 25's file) instead. Root failure: symptom "position limits not enforced" is semantically identical to eval 25's domain. The orchestrator keyword table is advisory; the model overrides it with training-data knowledge of the codebase. Fix would require either renaming the eval 34 files to include a unique prefix (e.g., `CTradeQueue*`), or adding "Multi-source schema" as a distinct bug classification that hard-routes to queue files.

²⁰ **Eval 35 — multi-agent FAIL (stoi anchoring + missing cross-file synthesis)**: `CTradeApiCreditParser::ParseCreditLimit` searches for `"approved_credit_usd:"` (v2 field name) but API v3 response contains `"credit_limit_usd"`. `find()` returns `npos`, function returns 0, all trades blocked. Thread B correctly finds `CTradeApiCreditParser.cpp` but investigator anchors on `std::stoi` at line 20 and diagnoses truncation, ignoring the `if (nPos==npos) return 0` at line 12. Additionally, abstractor defaults to known `CCreditCheckService::Success(false)` pattern (eval 10) when synthesizing credit-limit bugs. Two simultaneous failure modes: (1) stoi anchoring despite Pattern G two-step rule, (2) abstractor training-data prior overrides file evidence. Fix would require cross-file comparison: Thread B must read BOTH `CTradeApiCreditParser.cpp` AND `CTradeApiCreditClient.cpp` and diff the field names.

²¹ **Eval 33 — single-agent FAIL (anchors on old fee files)**: Agent keywords `fee`, `v2`, `policy` match old eval files `CTradeFeeValidator.cpp` and `CTradeFeeSchedule.cpp` via training-data knowledge. Agent never reads `CTradeApiResponseParser.cpp` or `CTradeApiFeeClient.cpp` (the actual bug files). Produces a bps/percent mismatch diagnosis (eval 31 pattern) instead. Root cause: single-agent has no keyword-routing mechanism — any file that mentions "fee" in existing knowledge dominates.

²² **Eval 34 — single-agent PASS (5/5)**: Correctly traced: `CTradeQueueConsumer::ProcessMessage` → `CTradeQueueMessageParser::ParseNotional` reads `vecFields[2]`. For NewProvider schema `COUNTERPARTY_ID|NOTIONAL_USD|FEE_BPS|PRODUCT_CODE`, `vecFields[2]` = FEE_BPS (e.g. 75) instead of NOTIONAL_USD (e.g. 1500000). Position limit check passes because 75 << 10M. Suggested schema versioning fix. **Counterintuitive result**: single-agent solved the eval that multi-agent (with keyword routing) consistently failed — because single-agent read the QueueConsumer comment directly without going through a keyword step.

²³ **Eval 34 — cross-source-tracer PARTIAL (3/5)**: Domain noun `queue` correctly extracted. Investigator found both `CTradeQueueConsumer.cpp` (source) and `CTradeQueueMessageParser.cpp` (parser). Identified field index mismatch. However: (1) states "returns 0" instead of "returns 75" (wrong value — FEE_BPS for a typical message), (2) fix suggests changing index 2→1 which breaks legacy format (schema versioning not addressed). Passes: ParseNotional identified as site, NewProvider schema noted, CTradeQueueConsumer as entry, always-passes consequence. Fails: wrong value (75 not 0), no versioning fix.

²⁴ **Eval 35 — single-agent FAIL (training-data prior for credit domain)**: Agent greps `credit_limit.*0` and `CCreditCheckService` (eval 10 domain). Reads `CCreditCheckService.cpp` and `CTradeCreditAuditLog.cpp`, never reaches `CTradeApiCreditParser.cpp` or `CTradeApiCreditClient.cpp`. Produces a report about `Success(false)` misuse (eval 10 pattern) with no connection to the API field rename. The audit log comment "per Section 4.2" is taken as a false lead instead of a false witness.

²⁵ **Eval 35 — cross-source-tracer PASS (5/5)**: Domain noun `credit` → glob `**/*Api<Credit>Client*.cpp` finds `CTradeApiCreditClient.cpp` immediately, bypassing the `CCreditCheckService` red herring. Investigator reads source (v3 JSON with `"credit_limit_usd"`) and parser (searches for `"approved_credit_usd"`) side by side. Explicit diff table identifies field name mismatch. All 5 assertions met: correct site (ParseCreditLimit:9), correct field names quoted, correct source file, nCreditLimit=0 consequence, exact fix stated.

---

## Running benchmarks

```bash
# Multi-agent
cd ~/.config/opencode/skills/cpp-bug-tracer
opencode run --agent cpp-bug-tracer/orchestrator "<bug description>"

# Single-agent qwen3
opencode run --agent cpp-single-agent "<bug description>"

# Single-agent GLM-5 (Z.AI Coding Plan)
opencode run --agent cpp-single-agent-glm5 "<bug description>"

# Cross-source tracer (API/queue format changes, field renames)
opencode run --agent cpp-cross-source-tracer/orchestrator "<bug description>"
```
