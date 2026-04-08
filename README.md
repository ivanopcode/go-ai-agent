# Go Minimal Agent Runtime — Spec

**Status:** Draft

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
   - [Full Context Control](#full-context-control)
   - [Skills over Protocols](#skills-over-protocols)
   - [Transparent Orchestration](#transparent-orchestration)
2. [Scope](#2-scope)
3. [Architecture](#3-architecture)
4. [Implementation Order](#4-implementation-order)
5. [Future (Post-MVP)](#5-future-post-mvp)
6. [Reference Projects](#6-reference-projects)

---

## 1. Design Philosophy

### Full Context Control

Every piece of context that enters the agent's prompt must be explicit and traceable. No hidden globals, no implicit merging of instruction files from parent directories.

- **Local-only contexts.** No global `agents.md`, `CLAUDE.md`, or similar files injected behind the scenes. The agent sees exactly what's in its config and project directory. Nothing more.
- **Context merging is explicit.** When multiple context sources exist (system prompt, project instructions, skills), the merge order and result must be inspectable. No magic layering.

### Skills over Protocols

Extensibility through **skills** — structured instructions that expand the agent's knowledge, not its runtime.

A skill is a `SKILL.md` file with YAML frontmatter (name, description, triggers) and a markdown body (workflows, rules, references). The agent loads a skill into its context when a trigger matches, then executes the task using its existing tools (bash, read, write). No new runtime capabilities needed — skills teach the agent **how** to use what it already has.

This replaces MCP (Model Context Protocol), which solves the same problem — extensibility — through a runtime protocol layer with dynamic discovery, handshake, and invocation. MCP adds indirection and opacity. Skills are just documents:

- **No protocol layer.** A skill is a file, not a service. No discovery, no handshake, no transport.
- **Progressive disclosure.** Frontmatter (~100 words) is always visible for triggering. Body loads only when triggered. Bundled references/scripts load on demand. Context window is not wasted.
- **Opt-in via project config.** Global skills (`~/.config/agent/skills/`) exist but are never loaded unless explicitly listed in the project config. If it's not in the config, it doesn't exist.
- **Inspectable.** You can read every skill the agent has access to. The merge order is explicit. No magic.
- **Project-level context assembly.** The project spec defines not just *which* skills are loaded, but *how* their contexts compose — order, priority, which parts of a skill's body are included or excluded. The agent runtime reads this spec and assembles the final context accordingly. In Codex CLI / Claude Code, this assembly happens inside the runtime opaquely. Here, the project owns the recipe.

Skills can bundle scripts and reference docs, but these are executed by existing tools (bash runs a script, read loads a reference), not by a plugin runtime.

### Transparent Orchestration

Current sub-agent implementations (Claude Code, Cursor, etc.) create opaque systems — you can't see what a sub-agent is doing, what context it has, or why it made a decision. The root TUI has no visibility into child agents.

We don't have a solution yet, but the constraint is clear:
- Sub-agent context, state, and progress must be visible from the root TUI
- No fire-and-forget agents running in invisible contexts
- The user must be able to inspect any agent's full context at any time

This needs rethinking before implementation. Not just "not in MVP" — architecturally unsolved.

---

## 2. Scope

A compiled Go binary that implements a ReAct agent loop with pluggable tools and configurable LLM endpoint.

### In Scope (MVP)

- **ReAct loop:** send -> stream response -> tool calls -> execute -> send results -> repeat
- **OpenAI Responses API** (single transport, replaceable base URL)
- **Streaming:** SSE-based, parsed manually (~30 LOC)
- **Roles:** system, developer, user
- **System prompt:** configurable via file, flag, or env (not hardcoded)
- **Pluggable tools:** Tool interface + registry, register at startup
- **Initial tools:** bash, read, write (edit later)
- **TUI mode:** Bubbletea (input bottom, chat top, streaming)
- **Headless mode:** plain stdout (print mode)
- **Config:** single JSON file + CLI flags + env vars, priority: flag > env > file > defaults
- **Sessions:** JSONL append-only persistence, resume support
- **Compaction:** not implemented in MVP but architecture supports it (conversation state designed for it)

### Out of Scope (deferred)

- Multiple LLM providers (only OpenAI Responses API shape for now)
- Image input
- Permission system (yolo mode for now)
- OAuth
- Web UI
- Skills (format is defined, runtime loading is deferred — see Philosophy)

### Rejected

- **MCP** — skills achieve the same goal with less indirection (see Philosophy)
- **Global context injection** — violates Full Context Control (see Philosophy)
- **Opaque sub-agents** — violates Transparent Orchestration (see Philosophy)

---

## 3. Architecture

### Core Principle: Agent Loop is UI-Agnostic

The loop emits events into a channel. TUI and print mode are different consumers.

```
cmd/agent/main.go              # entry, flags, mode

internal/
  config/
    config.go                  # Config struct, load from file/env/flags

  llm/
    client.go                  # OpenAI Responses API client (raw HTTP + SSE)
    types.go                   # Message, ToolCall, ToolResult, Response

  agent/
    loop.go                    # ReAct cycle, emits events to channel
    conversation.go            # Message history, system prompt management
    event.go                   # Event types (EventText, EventToolCall, etc.)

  tool/
    registry.go                # Tool interface + registry
    bash.go
    read.go
    write.go

  session/
    writer.go                  # JSONL append-only writer
    reader.go                  # JSONL reader for session resume

  ui/
    tui.go                     # bubbletea Model
    print.go                   # headless stdout mode
```

### Event System

```go
type Event interface{ eventMarker() }

type EventText       struct { Delta string }
type EventToolCall   struct { ID, Name string; Args json.RawMessage }
type EventToolResult struct { ID string; Output string; IsError bool }
type EventDone       struct { StopReason string }
type EventError      struct { Err error }
```

### Tool Interface

```go
type Tool struct {
    Name        string
    Description string
    Schema      json.RawMessage
    Execute     func(ctx context.Context, args json.RawMessage) (string, error)
}

type Registry struct {
    tools map[string]Tool
}
```

No magic, no plugins, no dynamic loading. Slice of structs, registered at startup.

### Config

```go
type Config struct {
    BaseURL          string `json:"base_url"`
    APIKey           string `json:"api_key"`
    Model            string `json:"model"`
    SystemPrompt     string `json:"system_prompt"`
    SystemPromptFile string `json:"system_prompt_file"`
    MaxTokens        int    `json:"max_tokens"`
}
```

Priority: CLI flag > env > config file (`~/.config/agent/config.json`) > defaults.

### Sessions (JSONL)

```go
type Entry struct {
    Type      string          `json:"type"`
    Timestamp time.Time       `json:"timestamp"`
    Role      string          `json:"role,omitempty"`
    Content   string          `json:"content,omitempty"`
    Raw       json.RawMessage `json:"raw,omitempty"`
}
```

Append-only. One file = one session. Crash-safe (lose at most last line).
Storage: `~/.config/agent/sessions/<timestamp>-<short-id>.jsonl`

### Conversation State (Compaction-Ready)

```go
type Conversation struct {
    SystemPrompt string
    Messages     []Message
}
```

System prompt always preserved separately. When compaction comes:
- Count tokens (heuristic: `len/4`, later tiktoken-go)
- Summarize old messages via LLM call
- Replace N messages with one summary message
- Append compaction entry to JSONL

### Dependencies (Minimum)

| Package | Purpose | When |
|---------|---------|------|
| stdlib `net/http` | HTTP client | Step 1 |
| stdlib `encoding/json` | JSON | Step 1 |
| stdlib `bufio` | SSE parsing | Step 1 |
| stdlib `os` | File I/O, config | Step 1 |
| `charmbracelet/bubbletea` | TUI | Step 6 |
| `charmbracelet/lipgloss` | TUI styles | Step 6 |
| `charmbracelet/glamour` | Markdown rendering | Step 6 |

No LLM SDKs. Raw HTTP. SSE parsed manually.

---

## 4. Implementation Order

```
Step 1: config + llm/types + llm/client
        Config loading, types, streaming HTTP client
        Test: send message, receive streamed text

Step 2: tool/registry + tool/bash
        One tool, registration, JSON Schema

Step 3: agent/loop + agent/conversation + session/writer
        ReAct loop + conversation state + JSONL persistence
        Test: "list files" -> bash ls -> response

Step 4: ui/print + cmd/agent/main
        Headless mode, working CLI with persistence
        ** MILESTONE: working agent in one binary **

Step 5: tool/read + tool/write + tool/edit
        File operations

Step 6: ui/tui
        Bubbletea: input bottom, chat top, streaming
        ** MILESTONE: full TUI agent **
```

After step 4: ~800-1000 LOC, working agent.
After step 6: ~1500-2000 LOC, full TUI agent.
Binary size: ~8-12MB.

---

## 5. Future (Post-MVP)

In rough priority order:

1. **Compaction** — token counting + LLM summarization
2. **Skills** — SKILL.md loading by triggers, opt-in via project config, progressive disclosure
3. **Edit tool** — diff-based file editing
4. **Multiple providers** — Anthropic Messages API, etc.
5. **Transparent sub-agents** — requires design work (see Philosophy)

---

## 6. Reference Projects

| Project | What to learn |
|---------|--------------|
| CCX-Go (10K LOC) | Cleanest minimal agent loop, raw HTTP to Claude, Bubbletea TUI |
| Workshop (1.9K LOC) | Progressive build, 6 files, educational |
| OpenCode/Crush (42K LOC) | Production-grade architecture, LSP, sessions |
| Goat (48K LOC) | Embeddable library design, 22 tools, permissions |
| Pi (102K LOC TS) | Original inspiration: extensions, compaction, session format |
