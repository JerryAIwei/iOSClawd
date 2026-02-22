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

## Step-by-Step Goals

Each goal maps to one roadmap phase. Every goal has a clear definition of done,
concrete test cases, and the test method used to verify it.

---

### Goal 1 — Core Agent Loop

**Objective:** A single agent can receive a text message, call the Claude API with
streaming, display tokens as they arrive, and persist the conversation across
app restarts.

**Definition of done:**
- `AgentQueue` actor serializes concurrent requests to the same agent
- SSE stream delivers tokens to UI in real time
- Conversation history survives app kill and relaunch
- API key stored in Keychain, never in code or UserDefaults

| # | Test Case | Expected Result |
|---|---|---|
| 1.1 | Send "Hello" to a fresh agent | Streaming response appears within 2s, tokens animate in |
| 1.2 | Send two messages simultaneously to same agent | Second message queued; responses arrive in order, no interleaving |
| 1.3 | Send messages to two different agents simultaneously | Both respond in parallel; neither blocks the other |
| 1.4 | Kill app mid-response, relaunch | Conversation history restored; partial response shown as-is |
| 1.5 | Enter wrong API key | Clear error message shown; no crash |
| 1.6 | Send message with no network | Graceful error with retry option |
| 1.7 | Simulate API 529 (overloaded) | Exponential backoff retried up to 5×; user sees progress |

**Test method:**
- 1.1–1.3, 1.5–1.6: **XCTest unit tests** on `AgentQueue` and `AnthropicClient`
  using `URLProtocol` stub to mock SSE responses
- 1.4: **XCUITest** — launch, send message, `XCUIApplication().terminate()`,
  relaunch, assert message row exists
- 1.7: **XCTest** with mock returning HTTP 529 three times then 200

---

### Goal 2 — Multi-Agent Orchestration

**Objective:** An Orchestrator agent can decompose a user task into subtasks,
delegate each to a specialized agent, and synthesize a final answer.

**Definition of done:**
- Orchestrator creates child `AgentTask` records in SwiftData
- Each subtask runs in its own `AgentQueue` slot concurrently
- Pipeline view shows subtask tree with live status per node
- Final synthesized answer delivered to the user when all subtasks complete

| # | Test Case | Expected Result |
|---|---|---|
| 2.1 | Ask orchestrator "Research Swift actors and write a code example" | Two subtasks created: Researcher + Coder; both run in parallel |
| 2.2 | One subtask fails mid-run | Orchestrator receives error result; synthesizes partial answer with caveat |
| 2.3 | Cancel pipeline mid-run | All in-flight `Task` handles cancelled within 1s; SwiftData status set to `.cancelled` |
| 2.4 | Pipeline with 5 subtasks | All 5 run concurrently (subject to `MAX_CONCURRENT = 5` cap) |
| 2.5 | Subtask produces a `tool_use` block | Tool executes, result injected, subtask continues correctly |
| 2.6 | App backgrounds during pipeline | Pipeline continues via `BGProcessingTask`; result available on return |

**Test method:**
- 2.1–2.5: **XCTest** on `OrchestratorAgent` with mocked `AnthropicClient` that
  returns preset subtask decomposition JSON
- 2.3: Assert `Task.isCancelled` propagates within timeout using
  `XCTestExpectation`
- 2.6: **Manual test** on device — background app, wait 10s, return; verify result

---

### Goal 3 — Core Tools

**Objective:** Agents can search the web, fetch URL content, and store/recall
key-value memory within and across conversations.

**Definition of done:**
- `ToolRegistry` resolves tool name to `AgentTool` implementation
- Tool results injected as `tool_result` blocks and loop continues
- `memory` tool values survive app restart (stored in SwiftData)

| # | Test Case | Expected Result |
|---|---|---|
| 3.1 | Agent calls `web_search("Swift concurrency")` | Returns ≥3 result snippets with titles and URLs |
| 3.2 | Agent calls `fetch_url("https://swift.org")` | Returns extracted text, ≤4000 chars, no raw HTML tags |
| 3.3 | Agent calls `memory set key="name" value="Jerry"` then later `memory get key="name"` | Returns "Jerry" in second call |
| 3.4 | Kill and relaunch app; agent calls `memory get key="name"` | Still returns "Jerry" (persisted in SwiftData) |
| 3.5 | Unknown tool name returned by Claude | `ToolRegistry` returns `.notFound` error; injected as `is_error: true` result |
| 3.6 | `fetch_url` with non-200 response | Returns descriptive error string; agent handles gracefully |

**Test method:**
- 3.1–3.2: **XCTest** with live network (integration test, gated behind
  `XCTSkipUnless(ProcessInfo.processInfo.environment["RUN_NETWORK_TESTS"] != nil)`)
- 3.3–3.4: **XCTest** with in-memory SwiftData container for 3.3;
  real on-disk container for 3.4
- 3.5–3.6: **XCTest** unit tests with mock HTTP responses

---

### Goal 4 — Skill 1: Apple Shortcuts

**Objective:** The agent can list, run, and design Apple Shortcuts to automate
any app that exposes Shortcuts actions.

**Definition of done:**
- `list_shortcuts` returns all installed shortcuts by name
- `run_shortcut(name)` opens the Shortcuts URL scheme and polls for result
- Agent can complete an end-to-end task ("add a reminder") via Shortcuts

| # | Test Case | Expected Result |
|---|---|---|
| 4.1 | Call `list_shortcuts()` | Returns list including at least one system shortcut |
| 4.2 | Call `run_shortcut("Get Battery Level")` | Returns battery % as string |
| 4.3 | Ask agent "Add a reminder: buy milk at 8pm" | Agent calls `run_shortcut` with correct Reminders shortcut; reminder appears in Reminders app |
| 4.4 | Call `run_shortcut("NonExistentShortcut")` | Returns error "Shortcut not found"; agent tries alternative |
| 4.5 | Shortcut takes >10s to run | Tool waits up to 30s; returns timeout error with partial result if available |

**Test method:**
- 4.1: **XCUITest** — launch app, trigger `list_shortcuts`, assert non-empty array
- 4.2, 4.5: **Manual test** on physical device (Simulator lacks Shortcuts app)
- 4.3: **Manual end-to-end test** — verify reminder in Reminders.app after agent run
- 4.4: **XCTest** unit test — mock URL scheme handler returns not-found

---

### Goal 5 — Skill 2: Native iOS Frameworks

**Objective:** The agent can read and write data in Calendar, Reminders,
Contacts, Photos, Health, Home, and Music without using the UI.

**Definition of done:**
- Each framework tool requests the correct permission on first use
- CRUD operations reflect immediately in the corresponding Apple app
- Errors (permission denied, not found) returned as structured strings

| # | Test Case | Expected Result |
|---|---|---|
| 5.1 | Ask agent "Schedule a meeting tomorrow at 3pm called Team Sync" | Event appears in Calendar.app with correct time |
| 5.2 | Ask agent "What's my step count today?" | Returns today's step count from HealthKit |
| 5.3 | Ask agent "Find contact named Tim Cook" | Returns name, email, phone from Contacts |
| 5.4 | Ask agent "Turn off the living room lights" | HomeKit `home_control` toggles the accessory off |
| 5.5 | Call any tool with permission denied | Returns "Permission denied for [framework]" and prompts user to grant in Settings |
| 5.6 | `calendar` tool creates duplicate event on retry | SwiftData idempotency key prevents duplicate EKEvent insertion |

**Test method:**
- 5.1, 5.3: **XCTest** with `EventKit`/`Contacts` in-memory store mock
- 5.2: **XCTest** with `HKHealthStore` mock (inject mock query results)
- 5.4: **Manual test** on device with real HomeKit accessories (or HomeKit Accessory Simulator)
- 5.5: **XCTest** — set authorization status to `.denied` via mock store, assert error string
- 5.6: **XCTest** unit test — call `calendar create` twice with same idempotency key, assert one event

---

### Goal 6 — Skill 3: Accessibility Snapshot + Tap

**Objective:** The agent can read the current screen's element tree and inject
taps and text, enabling it to operate any app that has accessibility support.

**Definition of done:**
- `screen_snapshot()` returns a labeled, hierarchical text description of all
  visible interactive elements
- `tap_element(label)` correctly activates the matching element
- The agent can navigate a real app (e.g., open Settings > WiFi) using only
  snapshot + tap calls

| # | Test Case | Expected Result |
|---|---|---|
| 6.1 | Call `screen_snapshot()` on the app's own Chat screen | Returns structured list with element labels, roles, and frame hints |
| 6.2 | Call `screen_snapshot(interactive_only: true)` | Returns only tappable/editable elements |
| 6.3 | Call `tap_element("Send")` when Send button visible | Button activates; message sends |
| 6.4 | Call `tap_element("ElementThatDoesNotExist")` | Returns "Element not found"; no crash |
| 6.5 | Ask agent "Open Settings and turn on Airplane Mode" | Agent calls snapshot repeatedly, taps correct elements in sequence |
| 6.6 | Call `type_text("Hello")` with text field focused | "Hello" appears in field |

**Test method:**
- 6.1–6.2: **XCUITest** — launch test host app, call `screen_snapshot()`,
  assert known elements appear in output string
- 6.3, 6.6: **XCUITest** — inject tap/type via `XCUIElement` proxy, assert UI state changes
- 6.4: **XCTest** unit test — mock accessibility tree with no matching element
- 6.5: **Manual end-to-end test** on device; record screen for review

---

### Goal 7 — Skill 4: Screen Capture + Claude Vision

**Objective:** The agent can capture a screenshot, understand it via Claude's
vision API, and act on what it sees — enabling automation of any app regardless
of accessibility support.

**Definition of done:**
- `capture_screen()` returns a valid base64 PNG via ReplayKit
- `analyze_screen(question)` returns a Claude vision response describing or
  answering questions about the current screen
- `find_and_tap(description)` correctly locates and taps a described element
  in an app with no accessibility labels

| # | Test Case | Expected Result |
|---|---|---|
| 7.1 | Call `capture_screen()` | Returns non-empty base64 string; decoded PNG matches current screen |
| 7.2 | Call `analyze_screen("What app is open?")` | Claude returns correct app name |
| 7.3 | Call `read_screen_text()` on a screen with visible text | Returns OCR'd text matching what's visible (≥80% accuracy) |
| 7.4 | Call `find_and_tap("the blue Submit button")` | Claude identifies coordinates; tap fires at correct location |
| 7.5 | Call `watch_screen("loading spinner is gone", timeout: 10)` | Polls every 500ms; resolves when spinner disappears |
| 7.6 | Screen capture permission denied by user | Returns "Screen recording permission required" error; shows permission prompt |
| 7.7 | App with no accessibility labels (e.g., a game) | `find_and_tap` still works via vision coordinates |

**Test method:**
- 7.1: **XCTest** — call `capture_screen()`, base64-decode, assert `UIImage(data:)` non-nil
- 7.2–7.4: **Integration test** with live Claude API (gated behind env flag);
  manual verification of visual accuracy
- 7.3: **XCTest** — render a `UILabel` with known text offscreen, run Vision OCR, assert ≥80% match
- 7.5: **XCTest** with mock screen states — assert polling stops at correct state
- 7.6: **XCTest** — mock `RPScreenRecorder.isAvailable = false`, assert error returned
- 7.7: **Manual end-to-end test** on device with a target app

---

### Goal 8 — Polish

**Objective:** The app is production-ready: responses render beautifully,
token costs are visible, and conversations can be exported.

**Definition of done:**
- Markdown in agent responses renders with correct formatting
- Code blocks have syntax highlighting per language
- Each task shows input/output token count and estimated USD cost
- Conversations exportable as Markdown or JSON

| # | Test Case | Expected Result |
|---|---|---|
| 8.1 | Agent responds with `**bold**`, `# heading`, ` ```swift ``` ` | Renders bold text, large heading, highlighted Swift code block |
| 8.2 | Agent responds with a table in Markdown | Table renders with borders and alternating row colors |
| 8.3 | Complete a task; view Agent Detail | Token usage (input + output) and estimated cost (USD) displayed |
| 8.4 | Export conversation as Markdown | File saved to Files.app; content matches messages verbatim |
| 8.5 | Export conversation as JSON | Valid JSON with `role`, `content`, `timestamp` fields per message |
| 8.6 | Render a 500-line streaming response | No frame drops below 60fps during streaming (measured by Instruments) |

**Test method:**
- 8.1–8.2: **XCUITest** — assert rendered `Text` or `WebView` contains expected
  attributed string elements (bold, heading font size, monospace font)
- 8.3: **XCTest** — mock API response with known `usage` block; assert cost label text
- 8.4–8.5: **XCTest** — export to temp directory, read file, assert content/schema
- 8.6: **Instruments Time Profiler** on physical device; assert Core Animation
  commit time ≤16ms during streaming

---

## Commit History

| Commit | Description |
|---|---|
| `initial` | Add DESIGN.md — architecture, components, roadmap |
| `redesign` | Rebase on nanoclawd Swift port; add four-layer iOS skills system |
| `goals` | Add step-by-step goals with test cases and test methods per phase |
