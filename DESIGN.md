# iOS Claude Agents — Design Document

> A native iOS app that runs a multi-agent AI system powered by the Claude API,
> similar in spirit to OpenHands/OpenClaude.

---

## Vision

Users can create, configure, and orchestrate specialized AI agents to complete
complex tasks collaboratively — directly from their iPhone or iPad.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   iOS App (SwiftUI)                  │
│                                                      │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────┐  │
│  │Dashboard │  │  Chat UI  │  │  Agent Config UI  │  │
│  └──────────┘  └───────────┘  └──────────────────┘  │
│                       │                              │
│         ┌─────────────▼──────────────┐               │
│         │      Agent Manager         │               │
│         │  (orchestrates all agents) │               │
│         └────────────────────────────┘               │
│              │              │                        │
│   ┌──────────▼───┐    ┌─────▼──────────┐            │
│   │ Orchestrator │    │  Specialized   │            │
│   │ Agent        │───▶│  Agents        │            │
│   └──────────────┘    │  (Coder,       │            │
│                       │   Researcher,  │            │
│                       │   Writer, etc.)│            │
│                       └───────────────┘            │
│                              │                       │
│              ┌───────────────▼────────────┐          │
│              │        Tool Registry        │          │
│              │  web_search / fetch_url /   │          │
│              │  memory / code_helper / ... │          │
│              └────────────────────────────┘          │
│                              │                       │
│              ┌───────────────▼────────────┐          │
│              │    Anthropic API Client     │          │
│              │  (Messages API + Streaming) │          │
│              └────────────────────────────┘          │
└─────────────────────────────────────────────────────┘
                              │
                    ┌─────────▼────────┐
                    │  api.anthropic.com│
                    │  /v1/messages     │
                    └──────────────────┘
```

---

## Core Components

### 1. Agents

| Agent | Role |
|---|---|
| **Orchestrator** | Decomposes complex tasks, delegates to sub-agents, synthesizes results |
| **Coder** | Writes, reviews, and explains code |
| **Researcher** | Searches web, summarizes sources |
| **Writer** | Drafts content (docs, emails, posts) |
| **Analyst** | Interprets data, finds patterns |
| **Custom** | User-defined system prompt and tool set |

Each agent has:
- A Claude model assignment (Opus / Sonnet / Haiku)
- A system prompt (preset or custom)
- An enabled tool list
- A conversation history

### 2. Agent Loop

```
User Task
    │
    ▼
[Orchestrator] → breaks into subtasks → [Sub-Agents]
                                              │
                                    Claude API call
                                              │
                                    ┌─────────▼──────────┐
                                    │  text response?    │──▶ show to user
                                    │  tool_use?         │──▶ run tool → inject result → loop
                                    └────────────────────┘
                                              │
                                    [Orchestrator] collects results
                                              │
                                    Final synthesized answer
```

### 3. Tool System

Tools are registered globally; agents opt-in per config:

| Tool | Description |
|---|---|
| `web_search` | Searches the web (Brave/Serper API) |
| `fetch_url` | Fetches and extracts text from a URL |
| `memory` | Key-value store across the conversation |
| `code_helper` | Saves/retrieves code snippets |
| *(future)* `run_code` | Executes code via a remote sandbox |
| *(future)* `image_gen` | Generates images via diffusion models |

### 4. UI Screens

| Screen | Purpose |
|---|---|
| **Dashboard** | Overview of all agents and active tasks |
| **Chat** | Single-agent conversation with streaming |
| **Pipeline** | Multi-agent task with subtask tree and progress |
| **Agent Detail** | Status, conversation log, token usage |
| **Create Agent** | Pick type, model, tools, custom prompt |
| **Settings** | API key, default model, tool API keys |

### 5. Data Persistence

| Layer | Technology | What's stored |
|---|---|---|
| Secrets | Keychain | Anthropic API key, tool API keys |
| App state | SwiftData | Agent configs, task history, message logs |
| Ephemeral | In-memory | Active tool state (memory tool, code snippets) |

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| UI | SwiftUI | Native, declarative, iOS 17+ |
| State | `@Observable` + `@MainActor` | Modern Swift concurrency |
| Networking | `URLSession` async/await | Native SSE streaming, no deps |
| Persistence | SwiftData | Modern replacement for CoreData |
| Streaming | `AsyncThrowingStream` | Native SSE parsing |
| Package mgmt | Swift Package Manager | No CocoaPods needed |
| Build config | XcodeGen (`project.yml`) | `.xcodeproj` generated, not committed |

---

## Key Design Decisions

1. **No backend required** — App calls Anthropic directly. Optional user-provided proxy URL.
2. **Tool execution on device** — All built-in tools run on-device. Code *execution* requires an external sandbox (iOS security constraint).
3. **Streaming first** — All Claude responses stream token-by-token for responsiveness.
4. **Model per agent** — Each agent can use a different model (e.g., Haiku for fast sub-tasks, Opus for orchestration).
5. **XcodeGen** — `project.yml` is committed; `.xcodeproj` is gitignored and generated locally.

---

## Phased Roadmap

| Phase | Scope |
|---|---|
| **P1 – Core** | Single agent chat, streaming, API key setup, SwiftData persistence |
| **P2 – Multi-agent** | Orchestrator + sub-agents, task pipeline view, subtask tree UI |
| **P3 – Tools** | `web_search`, `fetch_url`, `memory`, `code_helper` |
| **P4 – Polish** | Markdown rendering, syntax highlighting, export history |
| **P5 – Extensions** | Custom tool plugins, backend proxy support, image generation |

---

## Open Questions / Known Gaps

| Question | Options |
|---|---|
| **Code execution** | (a) WASM runtime on-device, (b) user-configured remote server, (c) Shortcuts integration |
| **Search provider** | Brave Search API (preferred), Serper, or Bing |
| **Cost visibility** | Show token usage + estimated cost per task/agent |
| **Multi-user** | Out of scope for now — single user, one API key |

---

## Commit History

| Commit | Description |
|---|---|
| `initial` | Add DESIGN.md — architecture, components, roadmap |
