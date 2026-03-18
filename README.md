# pi-interactive-subagents

Interactive subagents for [pi](https://github.com/badlogic/pi-mono) — spawn, orchestrate, and manage sub-agent sessions in [cmux](https://github.com/manaflow-ai/cmux) terminals.

This package gives pi the ability to delegate work to specialized sub-agents that run in their own terminal sessions. A main orchestrator session spawns scouts, planners, workers, and reviewers — each visible in a side-by-side cmux split. The user can watch progress in real-time, interact with interactive agents, and get summaries when autonomous agents finish.

## Install

```bash
pi install git:github.com/HazAT/pi-interactive-subagents
```

> **Requires [cmux](https://github.com/manaflow-ai/cmux).** Start pi inside cmux for subagent support: `cmux pi`

## What's Included

### Extensions

**Subagents** — 4 tools + 3 commands for spawning and managing sub-agents:

| Tool | Description |
|------|-------------|
| `subagent` | Spawn a sub-agent in a dedicated cmux terminal |
| `subagents_list` | List available agent definitions |
| `set_tab_title` | Update cmux tab title to show progress |
| `subagent_resume` | Resume a previous sub-agent session |

| Command | Description |
|---------|-------------|
| `/plan` | Start a full planning workflow |
| `/iterate` | Fork into an interactive subagent for quick fixes |
| `/subagent <agent> <task>` | Spawn a named agent directly |

**Session Artifacts** — 2 tools for session-scoped file storage:

| Tool | Description |
|------|-------------|
| `write_artifact` | Write plans, context, notes to a session-scoped directory |
| `read_artifact` | Read artifacts from current or previous sessions |

### Bundled Agents

Five agent definitions ship with the package. Each has a specific role, model, and system prompt:

| Agent | Model | Role |
|-------|-------|------|
| **planner** | Opus (medium thinking) | Interactive brainstorming — clarifies requirements, explores approaches, writes plans, creates todos |
| **scout** | Haiku | Fast codebase reconnaissance — maps files, patterns, conventions |
| **worker** | Sonnet | Implements tasks from todos — writes code, runs tests, makes polished commits |
| **reviewer** | Opus (medium thinking) | Reviews code for bugs, security issues, correctness |
| **visual-tester** | Sonnet | Visual QA via Chrome CDP — screenshots, responsive testing, interaction testing |

Agent discovery follows priority: **project-local** (`.pi/agents/`) > **global** (`~/.pi/agent/agents/`) > **package-bundled**. Override any bundled agent by placing your own version in the higher-priority location.

---

## The `/plan` Workflow

The `/plan` command orchestrates a full planning-to-implementation pipeline. It's the primary way to build features with subagents.

```
/plan Add a dark mode toggle to the settings page
```

### How It Works

```
┌─────────────────────────────────────────────────────┐
│  Phase 1: Investigation (main session)              │
│  Quick codebase scan — file structure, patterns     │
├─────────────────────────────────────────────────────┤
│  Phase 2: Planning (interactive subagent)           │
│  User collaborates with the planner agent           │
│  → Clarify requirements                             │
│  → Explore approaches                               │
│  → Validate design                                  │
│  → Write plan artifact                              │
│  → Create scoped todos                              │
├─────────────────────────────────────────────────────┤
│  Phase 3: Review Plan (main session)                │
│  Confirm todos, adjust if needed                    │
├─────────────────────────────────────────────────────┤
│  Phase 4: Execute (sequential workers)              │
│  Scout gathers context → Workers implement todos    │
│  Each worker: claim → implement → test → commit     │
├─────────────────────────────────────────────────────┤
│  Phase 5: Review (reviewer subagent)                │
│  Code review with priority triage (P0–P3)           │
│  Fix critical issues, skip nits                     │
└─────────────────────────────────────────────────────┘
```

**Phase 1** — The main session does a quick investigation (30–60 seconds) to gather context. For larger codebases, it spawns a scout agent first.

**Phase 2** — An interactive planner opens in a side-by-side terminal split. The planner walks through requirements, proposes approaches, validates the design section by section, and produces a plan artifact + scoped todos. The user collaborates directly — answering questions, picking approaches, confirming design decisions. When done, Ctrl+D returns to the main session.

**Phase 3** — The main session reviews the plan and todos. The user can adjust before execution begins.

**Phase 4** — A scout gathers implementation context, then workers execute todos one at a time. Each worker claims a todo, implements it, runs tests, makes a polished commit, and closes the todo. Workers run sequentially to avoid git conflicts.

**Phase 5** — A reviewer examines all changes, producing a prioritized report. P0/P1 issues get fixed immediately by additional workers. P2/P3 items are noted or skipped.

Throughout the workflow, cmux tab titles update to show current phase:

```
🔍 Investigating: dark mode    →    💬 Planning: dark mode
→    🔨 Executing: 1/3    →    🔨 Executing: 2/3
→    🔎 Reviewing: dark mode    →    ✅ Done: dark mode
```

---

## The `/iterate` Workflow

For quick, focused work — bugfixes, small changes, ad-hoc tasks — without polluting the main session's context.

```
/iterate Fix the off-by-one error in the pagination logic
```

This forks the current session into an interactive subagent. The sub-agent has full conversation context (it knows everything discussed so far) and opens in a side-by-side terminal. Make the fix, verify it, and Ctrl+D to return. The main session gets a summary of what was done.

Use `/iterate` when:
- You spot a bug during a larger workflow
- You want to try something without cluttering the main session
- You need hands-on work with full context

---

## Direct Subagent Spawning

For fine-grained control, spawn agents directly:

```
/subagent scout Analyze the authentication module
/subagent worker Implement TODO-abc123
/subagent reviewer Review the last 3 commits
```

Or let the LLM use the `subagent` tool with full parameter control:

```typescript
subagent({
  name: "Scout",
  agent: "scout",
  interactive: false,
  task: "Map the database schema and query patterns"
})
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | string | required | Display name (shown in cmux tab) |
| `task` | string | required | Task prompt for the sub-agent |
| `agent` | string | — | Load defaults from agent definition |
| `interactive` | boolean | `true` | User collaborates vs. autonomous |
| `fork` | boolean | `false` | Copy current session for full context |
| `model` | string | — | Override agent's default model |
| `systemPrompt` | string | — | Append to system prompt |
| `skills` | string | — | Comma-separated skill names |
| `tools` | string | — | Comma-separated tool names |

---

## Parallel Subagents

The `parallel_subagents` tool runs multiple autonomous sub-agents concurrently with a single tool call. Each agent gets its own cmux terminal in a tiled layout, and progress updates stream in as each agent completes.

```typescript
parallel_subagents({
  agents: [
    { name: "Scout: Auth", agent: "scout", task: "Analyze the authentication module" },
    { name: "Scout: Database", agent: "scout", task: "Map the database schema and query patterns" },
    { name: "Scout: API", agent: "scout", task: "Document the REST API endpoints" },
  ]
})
```

### Terminal Layout

Parallel agents are arranged in a tiled layout next to the orchestrator:

```
┌──────────────────┬──────────────────┐
│                  │  Scout: Auth     │
│                  ├──────────────────┤
│   Orchestrator   │  Scout: Database │
│                  ├──────────────────┤
│                  │  Scout: API      │
└──────────────────┴──────────────────┘
```

The first agent splits right from the orchestrator (side-by-side). Each subsequent agent splits down from the previous one (stacked vertically). All terminals are visible simultaneously.

### Progress Rendering

Unlike separate `subagent` calls, `parallel_subagents` is a single tool call — so progress updates render immediately as each agent finishes:

```
1/3 done · 45s elapsed
  ✓ Scout: Auth — done
  ⟳ Scout: Database — 32s · 12 msgs (8.2KB)
  ⟳ Scout: API — 28s · 8 msgs (5.1KB)
```

### Good Candidates for Parallelism

- Multiple scouts gathering context from different parts of the codebase
- Independent workers on non-overlapping tasks (be careful with git conflicts)
- A scout + a researcher running simultaneously
- Parallel research on different topics

All agents run autonomously (`interactive: false` is enforced). Each parameter accepts the same options as `subagent` except `interactive` and `fork`.

---

## Custom Agents

Create your own agent definitions. Place a `.md` file in `.pi/agents/` (project) or `~/.pi/agent/agents/` (global):

```markdown
---
name: my-agent
description: Does something specific
model: anthropic/claude-sonnet-4-6
thinking: minimal
tools: read, bash, edit, write
---

# My Agent

You are a specialized agent that does X...
```

The frontmatter configures the agent defaults. The body becomes the system prompt.

---

## Requirements

- [pi](https://github.com/badlogic/pi-mono) — the coding agent
- [cmux](https://github.com/manaflow-ai/cmux) — terminal multiplexer for side-by-side splits

Start pi inside cmux:

```bash
cmux pi
```

## License

MIT
