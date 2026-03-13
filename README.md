# C++ Bug Tracer

An OpenCode skill for investigating bugs in C++ codebases using coordinated AI agents.

## Structure

```
cpp-bug-tracer/
├── SKILL.md              # Skill definition - triggers automatically
├── workflow.yml          # Workflow config for maintenance/DEV testing
└── agents/
    ├── thinking-agent.md     # Pure reasoning planner (no tools)
    ├── abstraction-agent.md  # Investigation thread owner
    └── worker-agent.md       # Leaf code reader
```

## Installation

```bash
# Copy skill definition
mkdir -p ~/.config/opencode/skills/cpp-bug-tracer
cp SKILL.md ~/.config/opencode/skills/cpp-bug-tracer/

# Copy agents
mkdir -p ~/.config/opencode/agents
cp agents/*.md ~/.config/opencode/agents/

# Copy workflow config (for workflow-creator skill)
cp workflow.yml ~/.config/opencode/skills/cpp-bug-tracer/
```

## Usage

The skill triggers automatically when you describe a C++ bug:

```
opencode
> Why does my C++ app crash when validating unregistered product types?
```

Or explicitly:

```
opencode run --skill cpp-bug-tracer "BUG: crash in validation CODEBASE: /path"
```

## Agents

| Agent | Tools | Role |
|-------|-------|------|
| thinking-agent | None | Pure reasoning, decomposition, planning |
| abstraction-agent | read, grep, glob, task | Owns investigation threads, spawns workers |
| worker-agent | read, grep, glob | Reads specific code, answers targeted questions |

## How It Works

1. **thinking-agent** analyzes the bug and proposes investigation threads
2. **abstraction-agent** spawns workers in parallel for each thread
3. **worker-agent** reads code and returns findings with file:line refs
4. Results are synthesized into a root cause report

## Requirements

- OpenCode CLI
- Model supporting tool calls (configurable per agent)
