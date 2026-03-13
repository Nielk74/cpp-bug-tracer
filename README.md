# C++ Bug Tracer

An OpenCode skill for investigating bugs in C++ codebases using coordinated AI agents.

## Installation

Copy the files to your OpenCode configuration:

```bash
# Copy the main agents
cp orchestrator.md planner.md explorer.md abstractor.md investigator.md ~/.config/opencode/agents/cpp-bug-tracer/

# Copy the skill definition
cp SKILL.md ~/.config/opencode/skills/cpp-bug-tracer/

# Copy the workflow configuration
cp workflow.yml ~/.config/opencode/skills/cpp-bug-tracer/

# Copy optional enhanced agents
cp agents/*.md ~/.config/opencode/agents/
```

## Usage

```
opencode run --agent cpp-bug-tracer/orchestrator "BUG: <description>
CODEBASE: /path/to/codebase"
```

## Agents

| Agent | Description |
|-------|-------------|
| orchestrator | Main entry point - greps codebase and writes bug report |
| planner | Decides where to start investigation |
| explorer | Searches for symbols in codebase |
| abstractor | Analyzes grep results for bug causes |
| investigator | Standalone bug finder with grep |

## Requirements

- OpenCode CLI
- `ollama/qwen3-coder:30b` model (or configure alternative in agent files)

## Model Configuration

Add to `~/.config/opencode/opencode.json`:

```json
{
  "provider": {
    "ollama": {
      "models": {
        "qwen3-coder:30b": { "name": "Qwen 3 Coder 30B" }
      }
    }
  }
}
```
