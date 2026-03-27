---
name: project-coordinator
version: 1.0.7
description: |
  Spawns an isolated Project Coordinator session that owns a project's context,
  breaks work into tasks, and spawns subagents for parallel execution.
homepage: https://github.com/KaigeGao1110/Project-Coordinator
command-dispatch: tool
command-tool: project-coordinator-start
command-arg-mode: raw
permissions:
  - spawn: subagent sessions
  - read: workspace files
  - exec: shell commands via subagents
dataPolicy:
  archivedData: internal workspace only
  neverExternal: true
---

## Tools

### project-coordinator-start

**Input:** Project description following "//start "

**What it does:**
- Activates the Project Coordinator pattern for a new project
- Spawns an isolated coordinator session to manage the project
- Reports back to main session when done

**When to use:**
- User says "//start build a WhatsApp bot"
- User says "//start a new project: [description]"
- Any multi-step project needing subagent coordination

**Examples:**
- Input: "//start build a Chrome extension for Gmail"
- Output: "Starting new project: build a Chrome extension for Gmail. Spawning Project Coordinator..."

# Project Coordinator Skill

A skill for structuring multi-agent project execution with isolated session architecture.

---

## Manifest

```yaml
name: project-coordinator
version: 1.0.7
description: |
  Spawns an isolated Project Coordinator session that owns a project's context,
  breaks work into tasks, and spawns subagents for parallel execution.
  The Coordinator reports back to the main session when done.
permissions:
  - spawn: subagent sessions (via sessions_spawn)
  - read: workspace files
  - exec: shell commands via subagents
  - read: own session transcripts only
dataPolicy:
  archivedData: internal workspace only
  neverExternal: true
```

---

## Architecture

```
Main Session (never runs code directly)
  └── Project Coordinator (spawned per project, mode="run")
        └── Subagents (spawned by Coordinator, run tasks in parallel)
```

**Token efficiency**: Each project Coordinator is isolated. Old project sessions accumulate zero main-session tokens after completion.

**Session isolation**: By OpenClaw platform design, a Coordinator can only read its own session transcripts, not other sessions' transcripts.

---

## Trigger Conditions

Activated when:
- User says "start a new project" or "let's work on [project]"
- A task is complex and needs multiple subagents
- A task will take more than a few minutes

Do NOT activate for: quick questions, simple lookups, one-liner tasks.

### Trigger 3: Slash command
Type `//start ` followed by your project description to activate the Project Coordinator.
Example: "//start build a Chrome extension"

---

## Project Coordinator Pattern

### Step 1: Define the Project

Collect from the user:
- Project name (e.g., "cureforge-hr-assessment")
- Project description (1-2 sentences)
- Key deliverables
- Any known constraints

### Step 2: Spawn Project Coordinator

```python
sessions_spawn(
  task=f"""You are the Project Coordinator for {project_name}.

Project: {project_description}

Your job:
1. Understand the full scope of this project
2. Break it into independent tasks
3. Spawn subagents to execute tasks in parallel
4. Monitor subagent progress
5. Compile results and report back to main session

When done, write a summary and await further instructions.
""",
  label=f"coordinator-{project_name}",
  runtime="subagent",
  model="minimax-m2.7-highspeed",
  mode="run"
)
```

### Step 3: Monitor Progress

- Use `subagents(action="list")` to track subagent status
- Receive completion announcements via inter-session messages
- If a subagent fails, decide whether to retry or adapt

### Step 4: Compile and Report

When all subagents complete:
1. Review all outputs
2. Write a final summary
3. Save any deliverables before the Coordinator session ends

### Step 5: Archive (when project is done)

When the user says "archive this", archive the Coordinator session and all its subagent sessions:

**5a. Identify transcript files:**
- The Coordinator's own transcript, identified by its session key: `~/.openclaw/agents/<COORDINATOR_SESSION_KEY>/sessions/`
- Subagent transcripts are in the same directory

**5b. Sanitize before archiving:**
Before any commit, run sanitization on all transcript files. Replace sensitive strings:

```python
# Improved sanitize: catches more credential patterns
def sanitize(text):
    import re
    patterns = [
        # API keys and tokens
        (r'ghp_[a-zA-Z0-9]{36}', '[REDACTED-GITHUB-TOKEN]'),
        (r'github_pat_[a-zA-Z0-9_]{22,}', '[REDACTED-GITHUB-TOKEN]'),
        (r'xox[baprs]-[a-zA-Z0-9]{10,}', '[REDACTED-SLACK-TOKEN]'),
        (r'AIza[a-zA-Z0-9_-]{30,}', '[REDACTED-API-KEY]'),
        (r'ya29\.[a-zA-Z0-9_-]{100,}', '[REDACTED-API-KEY]'),
        (r'sk-[a-zA-Z0-9]{48}', '[REDACTED-OPENAI-KEY]'),
        (r'sk_proj_[a-zA-Z0-9]{48}', '[REDACTED-OPENAI-KEY]'),
        (r'[\w.-]+@[\w.-]+\.\w+', '[REDACTED-EMAIL]'),
        (r'\+?[\d\s\-\(\)]{10,}\d{4,}', '[REDACTED-PHONE]'),
        (r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}', '[REDACTED-IP]'),
        (r'Bearer [a-zA-Z0-9_-]+', '[REDACTED-TOKEN]'),
        (r'Bearer\s+([a-zA-Z0-9_-]+\.){2}[a-zA-Z0-9_-]+', '[REDACTED-JWT]'),
        (r'aws_access_key_id\s*=\s*[A-Z0-9]{16}', '[REDACTED-AWS-KEY]'),
        (r'aws_secret_access_key\s*=\s*[A-Za-z0-9/+=]{40}', '[REDACTED-AWS-SECRET]'),
    ]
    for pattern, replacement in patterns:
        text = re.sub(pattern, replacement, text, flags=re.IGNORECASE)
    return text
```

Apply to all transcript files before archiving.

**5c. Delete transcript files only after explicit user approval.**
NEVER auto-delete. Always ask: "Can I delete the archived session files? They are already backed up."

**5d. Update MEMORY.md** with a one-line project summary.

---

## Coordinator's Tool Usage

**Subagent sandboxing:** When spawning subagents, each subagent runs in an isolated sandbox with workspace-only filesystem access. Subagents cannot access credentials, environment variables, or session transcripts outside their scope. Network access is restricted per platform policy.

The Coordinator SHOULD directly call tools:
- `exec` — run commands, check files
- `write` — create output files
- `read` — examine code or documents
- `sessions_spawn` — spawn subagents for parallel work
- `subagents` — monitor subagent status

---

## Subagent Naming Convention

Format: `{project}-{role}`

Examples:
- `cureforge-gke-migration`
- `cureforge-rabbitmq-queue`
- `cureforge-review`

---

## Session Management Rules

- One Coordinator per project (isolated token accounting)
- Do NOT run multiple unrelated projects in one Coordinator
- When a project is complete, the Coordinator stops — no lingering sessions
- For long-running projects: Coordinator can spawn child Coordinators for sub-phases

---

## After Project Completion

1. Report completion to Main session (brief summary)
2. Notify the user
3. User decides: archive / continue / hold
