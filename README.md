# C++ Bug Tracer

An OpenCode skill for investigating bugs in C++ codebases using coordinated AI agents.

## Workflow

[View interactive diagram on Excalidraw](https://excalidraw.com/#json=yJWFQfoy0gNHyAkOZspn3,diIr7Re8xOTNaZT5MOOy3Q)

The orchestrator classifies the bug into one of 6 categories, then spawns parallel investigator threads. Each investigator follows a single execution path (read → grep → follow call chain). The orchestrator synthesizes results into a structured root cause report.

**Bug categories:**
| Category | Key signals | Strategy |
|---|---|---|
| Event/notification mismatch | "handler never fires", "wrong handler called" | Thread A: sender value · Thread B: receiver mapping |
| Counter/metric wrong | "count is double", "always N" | Grep all increment sites first |
| ID/key lookup failure | "not found", "always same result" | Thread A: generate+store · Thread B: lookup path |
| Conditional behavior | "only for product X", "only on first load" | Trace the branch condition and how state is set |
| Cross-layer state | "field wrong", "disabled unexpectedly" | Trace UI → signal → business logic |
| Should fail but doesn't | "can do X twice", "second call succeeds" | Thread A: guard path · Thread B: state update after success |

## Structure

```
cpp-bug-tracer/
├── README.md
├── workflow.yml              # Workflow config (topology + mock responses for DEV testing)
├── orchestrator.md           # Primary agent: classifies bug, spawns investigators, synthesizes
├── investigator.md           # Subagent: follows one execution thread (read + grep)
├── abstractor.md             # Subagent: text-only synthesis of multi-thread findings
├── explorer.md               # Subagent: maps codebase structure by layer
├── planner.md                # Subagent: investigation planning
└── evals/
    ├── evals.json            # 11 benchmark cases (evals 1–11)
    └── codebase/             # Synthetic C++ codebase for evaluation
        └── src/services/     # CTradeServiceFacade, CCreditCheckService, CNotificationService, ...
```

## Installation

```bash
# Copy agents to OpenCode agents directory
mkdir -p ~/.config/opencode/agents/cpp-bug-tracer
cp orchestrator.md investigator.md abstractor.md explorer.md planner.md \
   ~/.config/opencode/agents/cpp-bug-tracer/
```

## Usage

```bash
opencode run --agent cpp-bug-tracer/orchestrator \
  "Why does releasing a credit reservation always fail? Codebase: /path/to/project"
```

## Agents

| Agent | Mode | Tools | Role |
|-------|------|-------|------|
| orchestrator | primary | read, grep, glob, task | Classifies bug, spawns investigators, synthesizes report |
| investigator | subagent | read, grep, glob | Follows one call chain, reports file:line evidence |
| abstractor | subagent | none | Synthesizes findings from multiple threads (text only) |
| explorer | subagent | grep, glob | Maps codebase structure, locates relevant files |
| planner | subagent | read | Plans investigation strategy |

## Evals

11 benchmark cases covering:
- **Evals 1–3**: UI/form layer bugs (read-only field, spurious validation, field visibility)
- **Eval 4**: Hardcoded argument in adapter fallback
- **Eval 5**: Post-increment off-by-one as map key
- **Evals 6–7**: Notification/event type mismatch (wrong enum sent)
- **Eval 8**: Double-counted workflow metric (caller + callee both increment)
- **Eval 9**: Cross-layer state gap — facade updates local context, lifecycle service never notified (5 hops)
- **Eval 10**: ServiceResult misuse — `CheckCreditLimit` returns `Success(false)`, caller only checks `IsFailure()` (4 hops)
- **Eval 11**: Wrong argument — `ReleaseCreditForTrade` passes reservation ID where trade ID expected (4 hops)

Run evals with the workflow-creator skill's eval tooling or manually with:
```bash
opencode run --agent cpp-bug-tracer/orchestrator "<eval prompt>"
```
