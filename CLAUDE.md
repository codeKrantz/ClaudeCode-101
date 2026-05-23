# Claude Code 101 — Reference Notes

This repo captures key concepts from the Claude Code 101 certification.

---

## How Claude Code Works

Claude Code is an agentic AI coding assistant that runs as a CLI, desktop app, or IDE extension. It operates in a read-eval-print loop: it reads your request, reasons about what to do, calls tools (read files, run commands, search the web, etc.), observes the results, and iterates until the task is complete.

Key properties:
- **Agentic**: it can take multi-step actions autonomously, not just answer questions
- **Tool-driven**: all file I/O, shell commands, and external lookups happen through discrete tools with explicit permissions
- **Permission model**: tools run in an allowlist-based sandbox; the user approves anything outside the allowed set

---

## Context

Claude Code has a finite context window shared between the conversation history, file contents it reads, tool outputs, and system instructions. Managing context well is critical to getting good results.

- **Be specific**: vague prompts force Claude to explore broadly and burn context
- **CLAUDE.md files** are loaded at startup and provide persistent, zero-cost context (see below)
- **Compaction**: when the window fills, the harness auto-summarizes older turns so the session continues; important facts should be in CLAUDE.md rather than trusted to remain verbatim in history
- **Subagents** can be used to isolate expensive searches or parallel tasks from the main context window

---

## The Importance of Planning

For non-trivial tasks, planning before coding avoids wasted work and misaligned implementations.

- Use `/plan` or the Plan agent to produce a step-by-step approach before touching code
- The plan surfaces trade-offs and critical files early, when they are cheapest to discuss
- Confirm the plan with the user before executing — a quick alignment check beats an hour of rework
- Plans live in the conversation; once approved, use Tasks to track per-step progress

---

## CLAUDE.md Files

`CLAUDE.md` is a Markdown file read automatically by Claude Code at session start. It acts as persistent, always-available context for the project.

**What to put in CLAUDE.md:**
- Project overview and architecture
- Tech stack, key dependencies, and version constraints
- Coding conventions and style rules for the repo
- Commands for build, test, lint, and deploy
- Anything Claude would otherwise have to rediscover every session

**Scope and inheritance:**
- `CLAUDE.md` at the repo root applies to the whole project
- Subdirectory `CLAUDE.md` files add or override context for that subtree
- `~/.claude/CLAUDE.md` is a global file that applies to every session on your machine

**Rule of thumb**: if you find yourself telling Claude the same thing twice, it belongs in CLAUDE.md.

---

## Subagents

Subagents are child Claude processes spawned from the main session to handle a discrete, isolated task.

**When to use them:**
- Parallel independent work (e.g., running two research queries at once)
- Protecting the main context window from large, noisy outputs (e.g., a broad codebase search)
- Tasks that match a specialized agent type (Explore, Plan, code-review, etc.)

**Available agent types include:**
| Type | Purpose |
|---|---|
| `Explore` | Fast read-only codebase search |
| `Plan` | Architecture and implementation planning |
| `claude-code-guide` | Questions about Claude Code itself or the Anthropic API |
| `general-purpose` | Multi-step research and code tasks |

**Key behavior**: subagents start cold — they have no memory of the parent conversation. The prompt must be fully self-contained.

---

## Skills

Skills are reusable, pre-built capabilities that extend what Claude Code can do within a session. Users invoke them with `/skill-name` in the prompt.

**Examples of built-in skills:**
- `/plan` — structured planning before implementation
- `/review` — code review of the current diff or a PR
- `/security-review` — security-focused review of pending changes
- `/run` — launch and drive the project's app to verify a change works
- `/verify` — confirm a fix actually behaves correctly in the running app
- `update-config` — modify Claude Code settings (hooks, permissions, env vars)

Skills are triggered via the Skill tool; Claude should invoke the relevant skill before generating any other response when the user's request matches.

---

## MCPs (Model Context Protocol)

MCP is an open standard that lets Claude Code connect to external tools and data sources through a structured server interface.

- **What they are**: MCP servers expose tools (callable actions) and resources (readable context) to Claude over a standardized protocol
- **Examples**: a database MCP lets Claude query your DB; a browser MCP lets Claude control a browser; an IDE MCP provides diagnostics and code execution
- **Configuration**: MCP servers are registered in Claude Code settings (`~/.claude/settings.json` or `.claude/settings.json`)
- **Security**: MCP tools follow the same permission model as built-in tools — the user can approve or deny each call

MCPs make Claude extensible: any capability that can be wrapped in an MCP server can be added to a Claude Code session.

---

## Hooks

Hooks are shell commands configured in `settings.json` that the Claude Code harness executes automatically in response to specific events.

**Common hook events:**
| Event | When it fires |
|---|---|
| `PreToolUse` | Before any tool call (can block or modify it) |
| `PostToolUse` | After a tool call completes |
| `Stop` | When Claude finishes a response turn |
| `UserPromptSubmit` | When the user submits a prompt |

**Use cases:**
- Auto-format or lint files after every edit
- Post a Slack notification when a long task finishes
- Log all tool calls for an audit trail
- Block certain tool patterns to enforce guardrails

**Key point**: hooks are executed by the harness, not by Claude. Memory and preferences cannot replicate hook behavior — anything that must happen automatically on every event needs a hook.

Configure hooks via the `update-config` skill or by editing `.claude/settings.json` directly.
