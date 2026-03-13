# cpp-bug-tracer Benchmark

Comparing three configurations on the same 11 C++ bug evals:

| Config | Architecture | Model |
|--------|-------------|-------|
| **multi-agent** | orchestrator → abstractor → investigator ×2 | qwen/qwen3-coder-30b-a3b-instruct (OpenRouter) |
| **single-agent-qwen3** | one agent, reads files directly | qwen/qwen3-coder-30b-a3b-instruct (OpenRouter) |
| **single-agent-glm5** | one agent, reads files directly | thudm/glm-z1-32b-0414 (OpenRouter) |

---

## Results

| # | Bug type | multi-agent | single-qwen3 | single-glm5 |
|---|----------|-------------|--------------|-------------|
| 1 | Paste blocked (CashFlow read-only) | ✅ PASS | — | — |
| 2 | Spurious validation on load | ✅ PASS | — | — |
| 3 | Notional field FX vs CashFlow | ⚠️ PARTIAL¹ | ⚠️ PARTIAL¹ | — |
| 4 | Hardcoded counterparty ID 0 | ✅ PASS | — | — |
| 5 | Post-increment reservation off-by-one | ✅ PASS | — | — |
| 6 | Trade-validated notification never fires | ✅ PASS | — | — |
| 7 | Settlement triggers wrong handler | ✅ PASS | — | — |
| 8 | Counter double-incremented | ✅ PASS | — | — |
| 9 | Double settlement (cross-layer state) | ✅ PASS | ❌ FAIL² | — |
| 10 | Credit limit `Success(false)` misuse | ✅ PASS | ✅ PASS | — |
| 11 | Wrong argument to credit release | ✅ PASS | ✅ PASS | — |
| | **Score** | **10/11** | **~4/6 tested** | — |

> `—` = not yet benchmarked

### Notes

¹ **Eval 3 partial**: Both agents find `TradeForm::updateFieldVisibility` and correctly report that both FX and CashFlow are hidden per the code. However neither fully enumerates which product types DO show the notional field (Equity, FixedIncome, Derivative, Swap, Forward, Option, Future).

² **Eval 9 single-agent fail**: Single-agent cannot parallelize investigation threads. When the entry point for a cross-layer bug (facade updates local context, lifecycle service reads stale state) is ambiguous, the single agent wastes steps navigating uncertainty and produces a report for the wrong bug.

---

## Key observations

### When multi-agent wins
- **Cross-layer bugs** (eval 9): two independent threads cover the guard path and the state update path in parallel — neither thread alone has the full picture
- **Multi-file event bugs** (eval 7): Thread A traces the sender value, Thread B traces the receiver mapping — parallel coverage prevents the agent from anchoring on one file
- **Avoids anchoring**: single-agent anchors on the first file it reads and may miss the root cause in a second file

### When single-agent is comparable
- **Self-contained bugs** (evals 10, 11): bug fits in 1-2 files, single-agent finds it as well as multi-agent
- **Counter bugs** (eval 8): grep for the variable name surfaces both increment sites immediately

### Latency
| Config | Typical time |
|--------|-------------|
| multi-agent | 90–420s (parallel threads, OpenRouter variance) |
| single-agent | 60–200s |

---

## Running benchmarks

```bash
# Multi-agent
cd ~/.config/opencode/skills/cpp-bug-tracer
opencode run --agent cpp-bug-tracer/orchestrator "<bug description>"

# Single-agent qwen3
opencode run --agent cpp-single-agent "<bug description>"

# Single-agent GLM-Z1
opencode run --agent cpp-single-agent-glm5 "<bug description>"
```
