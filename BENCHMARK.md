# cpp-bug-tracer Benchmark

Comparing three configurations on **15 C++ bug evals** (11 original + 4 new complex evals):

| Config | Architecture | Model |
|--------|-------------|-------|
| **multi-agent** | orchestrator → abstractor → investigator ×2 | qwen/qwen3-coder-30b-a3b-instruct (OpenRouter) |
| **single-agent-qwen3** | one agent, reads files directly | qwen/qwen3-coder-30b-a3b-instruct (OpenRouter) |
| **single-agent-glm5** | one agent, reads files directly | glm-5 (Z.AI Coding Plan) |

---

## Results

| # | Bug type | multi-agent | single-qwen3 | single-glm5 |
|---|----------|-------------|--------------|-------------|
| 1 | Paste blocked (CashFlow read-only) | ✅ PASS | ✅ PASS | ✅ PASS |
| 2 | Spurious validation on load | ✅ PASS | ✅ PASS | ✅ PASS |
| 3 | Notional field FX vs CashFlow | ⚠️ PARTIAL¹ | ⚠️ PARTIAL¹ | ⚠️ PARTIAL¹ |
| 4 | Hardcoded counterparty ID 0 | ✅ PASS | ✅ PASS | ✅ PASS |
| 5 | Post-increment reservation off-by-one | ✅ PASS | ✅ PASS | ✅ PASS |
| 6 | Trade-validated notification never fires | ✅ PASS | ✅ PASS | ✅ PASS |
| 7 | Settlement triggers wrong handler | ✅ PASS | ⚠️ PARTIAL² | ✅ PASS |
| 8 | Counter double-incremented | ✅ PASS | ✅ PASS | ✅ PASS |
| 9 | Double settlement (cross-layer state) | ✅ PASS | ❌ FAIL³ | ✅ PASS |
| 10 | Credit limit `Success(false)` misuse | ✅ PASS | ✅ PASS | ✅ PASS |
| 11 | Wrong argument to credit release | ✅ PASS | ✅ PASS | ✅ PASS |
| | **Score (1–11)** | **10/11** | **8/11** | **10/11** |
| | | | | |
| 12 | Amendment validates stale notional | ⚠️ PARTIAL⁴ | ✅ PASS | ✅ PASS |
| 13 | Batch notify-before-execute ordering | ✅ PASS⁵ | ✅ PASS | ✅ PASS |
| 14 | Wrong risk limit in cumulative check | ✅ PASS | ✅ PASS | ✅ PASS |
| 15 | Pagination off-by-one (1-based page) | ✅ PASS | ✅ PASS | ✅ PASS |
| | **Score (12–15)** | **3/4** | **4/4** | **4/4** |
| | **Total score** | **13/15** | **12/15** | **14/15** |

### Notes

¹ **Eval 3 partial (all configs)**: All agents find `TradeForm::updateFieldVisibility` and correctly report that both FX and CashFlow are hidden per the code — user's premise is wrong. However none fully enumerate all product types that show the notional field (Equity, FixedIncome, Derivative, Swap, Forward, Option, Future). GLM-5 also hits max steps exploring the `setState(Computed)` red herring.

² **Eval 7 qwen3 partial**: qwen3 single-agent said "event type 5 missing in switch" — the actual bug is that `SETTLEMENT_COMPLETED = 3` collides with `case 3: OnTradeSubmitted`. Found the dispatcher but misread the collision direction.

³ **Eval 9 qwen3 fail**: Single-agent cannot parallelize threads. When the entry point for a cross-layer bug is ambiguous, qwen3 anchors on the wrong file and produces a report for a different bug entirely.

⁴ **Eval 12 multi-agent partial**: Orchestrator wastes steps attempting `cpp-bug-tracer/agents/worker-agent` (non-existent). Abstractor report mentions validation-before-update but hallucinated `TradeValidator.cpp` — never traces `m_mapTradeStore` or the local copy issue. 2/5 assertions pass.

⁵ **Eval 13 multi-agent (after fix)**: Previously failed — abstractor hallucinated `CTradeEventDispatcher.cpp` due to step budget exhausted by worker-agent failures. Fixed by rewriting orchestrator: `grep: false`, `steps: 6`, first action hardcoded to abstractor. Re-run now correctly identifies `NotifyTradeExecuted` before `ExecuteTrade` at line 817.

---

## Key observations

### When multi-agent wins
- **Cross-layer bugs** (eval 9): two parallel threads cover the guard path and the state update path simultaneously — neither thread alone has the full picture
- **Avoids anchoring**: qwen3 single-agent anchors on the first file it reads and may miss the root cause in a second file

### GLM-5 vs qwen3 single-agent
| | GLM-5 | qwen3 |
|---|---|---|
| Eval 7 (event collision) | ✅ Correctly traced `COMPLETED=3` collision | ⚠️ Misread as missing case 5 |
| Eval 9 (cross-layer state) | ✅ Found lifecycle state mismatch | ❌ Produced wrong-bug report |
| Eval 11 (wrong argument) | ✅ Quoted exact variable names, traced `m_mapCounterpartyReservedTotals` | ✅ Correct but less detail |
| Fix directions | More precise — includes exact code snippets | Correct but higher-level |
| Eval 3 (ambiguous premise) | ⚠️ Hits max steps chasing red herring | ⚠️ Produces reasonable report quicker |

**Summary**: GLM-5 is stronger at multi-file tracing and produces more precise fix directions. qwen3 is faster and less prone to rabbit holes on simple cases. Multi-agent (qwen3) covers the gap on cross-layer bugs by parallelizing.

### Complex evals reveal multi-agent weakness
The 4 new complex evals (12–15) expose a structural problem: the orchestrator still attempts the non-existent `worker-agent` before falling back to the abstractor, consuming a large portion of the step budget. On evals 12 and 13, the abstractor received a degraded investigation due to step starvation and produced hallucinated output. Both single-agent configs scored 4/4 on the same evals — demonstrating that for focused, single-file bugs the overhead of orchestration is a net negative.

| Weakness | Multi-agent | Single-agent-qwen3 | Single-agent-glm5 |
|----------|-------------|-------------------|------------------|
| Worker-agent step waste | ❌ Still occurs | N/A | N/A |
| Hallucination under step pressure | ❌ Evals 12, 13 | ✅ None | ✅ None |
| Cross-layer / multi-file bugs | ✅ Parallelism helps (eval 9) | ❌ Anchors wrong | ✅ Strong tracer |

### Latency
| Config | Typical time |
|--------|-------------|
| multi-agent | 90–420s (parallel threads, OpenRouter variance) |
| single-agent-qwen3 | 60–180s |
| single-agent-glm5 | 80–200s |

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
```
