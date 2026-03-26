# Project Coordinator Skill

A skill for structuring multi-agent project execution with isolated session architecture.

## What It Does

When you start a complex project, this skill spawns an isolated Project Coordinator session that owns the project context and spawns subagents for parallel execution. The main session stays lightweight and clean.

## Architecture

```
Main Session (coordinator only - no direct tool calls)
  └── Project Coordinator (spawned per project, isolated run)
        └── Subagents (spawned by Coordinator, run tasks in parallel)
```

## Key Benefits

- **Token isolation**: Each project runs in its own session — old projects don't accumulate tokens in the main session
- **Clean separation**: Main session only coordinates; Coordinators handle execution
- **Parallel execution**: Subagents run tasks concurrently under the Coordinator
- **Safe archiving**: Uses archive-project skill — credential sanitization and human approval before any file deletion

## What the Coordinator Can Do

The Coordinator can run shell commands, read/write workspace files, and spawn subagents. These are standard project execution capabilities. All archiving follows the archive-project skill, which sanitizes credentials and requires human approval before deleting files.

## For AI Agents

Install into workspace skills directory. The SKILL.md contains the full workflow, architecture rules, and Coordinator pattern.

## Language

All output is in English.
