# iOS Claude Agents — Design Document

> A native iOS app that runs a multi-agent AI system powered by the Claude API.
> The agent core is a Swift port of **nanoclawd**'s architecture.
> Skills give the agent the ability to operate any iOS app.

---

## Vision

A fully on-device AI agent system that can understand your intent and take action
across your iPhone — scheduling meetings, sending messages, controlling smart home
devices, reading your screen, and automating any app — all driven by Claude.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        iOS App (SwiftUI)                          │
│                                                                   │
│  ┌───────────┐   ┌────────────┐   ┌──────────┐   ┌───────────┐  │
│  │ Dashboard │   │  Chat UI   │   │ Pipeline │   │ Settings  │  │
│  └───────────┘   └────────────┘   └──────────┘   └───────────┘  │
│                         │                                         │
│            ┌────────────▼────────────────┐                        │
│            │        AgentQueue           │  ← nanoclawd's        │
│            │  (Swift actor, serializes   │    GroupQueue          │
│            │   per-agent, parallel       │    in Swift            │
│            │   across agents)            │                        │
│            └────────────────────────────┘                        │
│                 │                  │                              │
│    ┌────────────▼──────┐  ┌────────▼──────────┐                  │
│    │  OrchestratorAgent│  │  SpecializedAgents│                  │
│    │  (decomposes,     │─▶│  Coder / Research │                  │
│    │   delegates,      │  │  Writer / Analyst │                  │
│    │   synthesizes)    │  │  Custom           │                  │
│    └───────────────────┘  └───────────────────┘                  │
│                                   │                              │
│            ┌──────────────────────▼──────────────────┐           │
│            │              ToolRegistry                │           │
│            │  (nanoclawd's MCP tools → Swift protocol)│          │
│            │                                          │           │
│            │  Core tools:                             │           │
│            │   web_search / fetch_url / memory        │           │
│            │                                          │           │
│            │  iOS Skills (see below):                 │           │
│            │   Shortcuts / Native / Accessibility /   │           │
│            │   Vision                                 │           │
│            └──────────────────────────────────────────┘          │
│                                   │                              │
│            ┌──────────────────────▼──────────────────┐           │
│            │           AnthropicClient                │           │
│            │    (Messages API, SSE streaming)         │           │
│            └──────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────────┘
                                    │
                       ┌────────────▼────────────┐
                       │     api.anthropic.com    │
                       │     /v1/messages          │
                       └──────────────────────────┘
```

---

## nanoclawd → Swift Port Map

Every core concept from nanoclawd has a direct Swift equivalent.
No backend is needed — everything runs in-process inside the iOS app.

| nanoclawd (Node.js) | iOS Swift equivalent | Notes |
|---|---|---|
| `GroupQueue` class | `AgentQueue` actor | Serialize per-agent, parallel across agents |
| `runContainerAgent()` | `runAgent() async` | Swift `Task` instead of Docker container |
| Docker container | In-process Swift `Task` | iOS sandbox replaces OS-level isolation |
| Volume mounts | Swift module isolation | Each agent owns its own memory/state struct |
| IPC files | Swift `AsyncStream` / actor | No filesystem IPC needed in-process |
| SQLite (`db.ts`) | SwiftData | Same schema, different ORM |
| MCP server (stdio) | `AgentTool` protocol | Swift protocol replaces JSON-RPC |
| WhatsApp channel | `UIChannel` (SwiftUI) | iOS UI is the message channel |
| `formatMessages()` XML | Same XML format | Reuse the exact prompt structure |
| Session ID (Claude SDK) | Stored in SwiftData | Persisted session per agent |
| CLAUDE.md per group | Per-agent system prompt + memory | SwiftData `AgentConfig.memoryMd` field |
| `send_message` MCP tool | `UIChannel.stream(text)` | In-process callback, no IPC |
| `schedule_task` MCP tool | `BGTaskScheduler` | iOS Background Tasks framework |
| Skills engine | Swift Package Manager plugins | Same git-merge philosophy |
| Container idle timeout | `Task.withTimeout` | Swift structured concurrency |
| Retry + cursor rollback | Same logic in Swift | Identical pattern |

---

## Agent Core: AgentQueue

Mirrors nanoclawd's `GroupQueue` exactly — serializes work per agent,
parallelizes across agents.

```
User message arrives
        │
        ▼
  AgentQueue.enqueue(agentId, task)
        │
        ├─ Agent idle?  ──▶  spawn Swift Task immediately
        │
        └─ Agent busy? ──▶  mark pendingMessages = true
                            (processed when current task finishes)

Per-agent Swift Task:
  1. Fetch messages since lastCursor (SwiftData)
  2. Format as XML  ←── same as nanoclawd formatMessages()
  3. Call Claude API (streaming)
  4. On text delta  → stream to UI via AsyncStream
  5. On tool_use   → ToolRegistry.execute() → inject result → loop
  6. On stop       → persist new cursor, update session ID
  7. On error      → rollback cursor, retry with backoff (max 5)
```

---

## Agent Loop Detail

```
AgentQueue.runAgent(agentId)
        │
        ▼
AnthropicClient.streamMessage(model, messages, system, tools)
        │
    ┌───▼────────────────────────────────┐
    │  SSE stream (AsyncThrowingStream)  │
    └───┬───────────────────────┬────────┘
        │ text delta            │ tool_use block
        ▼                       ▼
  stream to UI           ToolRegistry.execute(name, input)
  (UIChannel)                   │
                         ┌──────▼──────┐
                         │ tool result │
                         └──────┬──────┘
                                │
                   inject as tool_result message
                                │
                   loop back to AnthropicClient
        │
    stop_reason == "end_turn"
        │
        ▼
   persist cursor + session
   notify AgentQueue (drain pending)
```

---

## iOS App Skills

Skills give the agent the ability to control the entire iPhone.
All four skill layers are included — agents opt in via their `enabledTools` list.

### Skill 1 — Apple Shortcuts

Run any Shortcut by name, including user-created automations.
Works with hundreds of apps via Shortcuts actions out of the box.

**Tools:**

| Tool | Description |
|---|---|
| `run_shortcut(name, input?)` | Opens `shortcuts://run-shortcut?name=X` and streams result |
| `list_shortcuts()` | Lists all installed shortcuts via Shortcuts framework |
| `create_shortcut_url(name, steps)` | Generates an importable shortcut URL for agent-designed automations |

**Works with:** Reminders, Calendar, Messages, Mail, Maps, Music, Home, Files, Notes, Photos, third-party apps exposing Shortcuts actions.

---

### Skill 2 — Native iOS Frameworks

Direct Swift access to Apple's first-party data APIs. No UI automation needed.

**Tools:**

| Tool | Framework | Capability |
|---|---|---|
| `calendar(action, ...)` | EventKit | Create/read/update/delete events |
| `reminder(action, ...)` | EventKit | Create/read/update reminders and lists |
| `contact(action, ...)` | Contacts | Create/search contacts |
| `message(to, body)` | MessageUI | Compose iMessage/SMS (presents sheet) |
| `photo_search(query)` | Photos | Search photos by date, album, metadata |
| `health_query(type, range)` | HealthKit | Read steps, sleep, heart rate, etc. |
| `home_control(device, action)` | HomeKit | Control lights, locks, thermostats |
| `music(action, query)` | MusicKit | Play/queue/search Apple Music |
| `location(query)` | MapKit | Geocode, reverse geocode, search places |

---

### Skill 3 — Accessibility Snapshot + Tap

Takes a structural snapshot of the current screen's accessibility tree and injects
gestures. Works on any app. Similar to nanoclawd's `agent-browser` skill.

**Tools:**

| Tool | Description |
|---|---|
| `screen_snapshot(interactive_only?)` | Returns accessibility tree as structured text (labels, roles, positions) |
| `tap_element(label)` | Finds element by accessibility label and taps it |
| `tap_xy(x, y)` | Taps at normalized coordinates (0.0–1.0) |
| `type_text(text)` | Types into the focused element |
| `swipe(direction, element?)` | Swipes on screen or specific element |
| `scroll(direction, amount)` | Scrolls current scroll view |
| `press_button(name)` | Presses system button (Home, Back, Share, etc.) |

**Implementation:** `UIAccessibility` API for snapshot; touch injection via
`XCTest` private SPI (dev builds) or `AXUIElement` with Accessibility entitlement.

**Entitlement required:** `com.apple.developer.accessibility.enhance-speech`
or run in development/TestFlight mode.

---

### Skill 4 — Screen Capture + Claude Vision

Captures the screen via ReplayKit, sends frames to Claude's vision API,
and acts on what it sees. The most powerful skill — works on any app with no
entitlement beyond screen recording permission.

**Tools:**

| Tool | Description |
|---|---|
| `capture_screen()` | Grabs current screen via ReplayKit → returns base64 image |
| `analyze_screen(question)` | Capture + ask Claude vision a question about it |
| `find_and_tap(description)` | Analyze screen, locate described element, compute coordinates, tap |
| `read_screen_text()` | Capture → OCR via Vision framework → return all visible text |
| `watch_screen(condition, timeout)` | Poll screen until condition is met (e.g., "loading is done") |

**Pipeline:**
```
ReplayKit.captureFrame()
    │
    ▼
Claude Vision API (claude-sonnet-4-6, image input)
    │
    ▼
Coordinates / text / decision
    │
    ▼
Skill 3 tap_xy() or Skill 1 run_shortcut()
```

---

## Skill Architecture (nanoclawd philosophy)

Skills are additive and composable. Each skill is a Swift Package that conforms
to a `Skill` protocol and registers its tools into the `ToolRegistry`.

```swift
protocol Skill {
    var id: String { get }
    var displayName: String { get }
    var tools: [any AgentTool] { get }
    func setup() async throws   // request permissions, etc.
}
```

Built-in skills ship with the app. Custom skills can be added as local
Swift Packages — same philosophy as nanoclawd's `git merge-file` skill engine.

---

## Data Persistence (mirroring nanoclawd's db.ts)

| Table | SwiftData Model | nanoclawd equivalent |
|---|---|---|
| `Agent` | `AgentConfig` | `registered_groups` |
| `Message` | `Message` | `messages` |
| `AgentTask` | `AgentTask` | `scheduled_tasks` |
| `TaskRunLog` | `TaskRunLog` | `task_run_logs` |
| `AgentState` | `AgentState` | `router_state` |
| `Session` | `AgentSession` | `sessions` |
| Secrets | iOS Keychain | env vars / secrets via stdin |

---

## UI Screens

| Screen | Purpose |
|---|---|
| **Dashboard** | All agents, their status, active tasks |
| **Chat** | Single-agent streaming conversation |
| **Pipeline** | Multi-agent task with subtask tree |
| **Screen Control** | Live screen snapshot, agent action log (Skill 3/4) |
| **Skills** | Enable/disable skills per agent, permission status |
| **Agent Detail** | Config, memory (CLAUDE.md), token usage, session |
| **Create Agent** | Type, model, tools, custom system prompt |
| **Settings** | API key (Keychain), default model, proxy URL |

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| UI | SwiftUI | Native, declarative, iOS 17+ |
| State | `@Observable` + `@MainActor` | Modern Swift concurrency |
| Agent concurrency | Swift `actor` + `Task` | Mirrors nanoclawd's GroupQueue |
| Streaming | `AsyncThrowingStream` | Native SSE, same pattern as nanoclawd |
| Networking | `URLSession` async/await | No extra deps |
| Persistence | SwiftData | Maps 1:1 to nanoclawd's SQLite schema |
| Secrets | iOS Keychain | Replaces nanoclawd's stdin-secrets |
| Screen capture | ReplayKit | Skill 4 |
| Accessibility | UIAccessibility + Vision | Skill 3 + OCR |
| Scheduling | BGTaskScheduler | Replaces nanoclawd's task-scheduler.ts |
| Build config | XcodeGen (`project.yml`) | `.xcodeproj` gitignored |

---

## Key Design Decisions

1. **nanoclawd patterns, Swift runtime** — Same agent loop, queue, cursor/rollback, retry logic. No JS, no backend.
2. **In-process isolation** — iOS sandbox replaces Docker. Each agent has its own `Task` and isolated SwiftData context.
3. **Four-layer skill stack** — Shortcuts (broad app support) + Native APIs (deep data access) + Accessibility (any UI) + Vision (fallback for anything else).
4. **Streaming first** — SSE token-by-token, identical to nanoclawd's output markers.
5. **Model per agent** — Haiku for fast sub-tasks, Opus for orchestration, same as nanoclawd's per-group config.
6. **No config files** — Settings via in-app UI + Keychain, mirroring nanoclawd's env-only config.

---

## Phased Roadmap

| Phase | Scope |
|---|---|
| **P1 – Core agent loop** | Swift port of AgentQueue + agent loop, streaming, SwiftData, API key |
| **P2 – Multi-agent** | Orchestrator + sub-agents, pipeline view, subtask tree |
| **P3 – Core tools** | `web_search`, `fetch_url`, `memory` |
| **P4 – Skill 1** | Shortcuts integration (`run_shortcut`, `list_shortcuts`) |
| **P5 – Skill 2** | Native frameworks (EventKit, Contacts, HomeKit, HealthKit, MusicKit) |
| **P6 – Skill 3** | Accessibility snapshot + tap |
| **P7 – Skill 4** | Screen capture + Claude vision + `find_and_tap` |
| **P8 – Polish** | Markdown rendering, syntax highlight, export, token cost display |

---

## Open Questions / Known Gaps

| Question | Status |
|---|---|
| **Touch injection entitlement** | Skill 3 needs `accessibility` entitlement — may limit App Store distribution; TestFlight/dev fine |
| **Screen recording permission** | Skill 4 requires user consent per session via ReplayKit UI |
| **Background agent runs** | `BGTaskScheduler` has strict time limits (~30s); long tasks need foreground or push-triggered wake |
| **Search API** | Brave Search API preferred; key stored in Keychain |
| **Cost visibility** | Token usage + estimated cost per task shown in Agent Detail |
| **Shortcuts result streaming** | `shortcuts://` URL scheme is fire-and-forget; result requires clipboard or shared file polling |

---

## Commit History

| Commit | Description |
|---|---|
| `initial` | Add DESIGN.md — architecture, components, roadmap |
| `redesign` | Rebase on nanoclawd Swift port; add four-layer iOS skills system |
