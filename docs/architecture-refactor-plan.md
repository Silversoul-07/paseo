# Paseo Architecture Refactor: Desktop AI Tools Wrapper

## Current State Analysis

Paseo is currently a **mobile-first monitoring app** for AI coding agents. The architecture is:

```
┌─────────────────┐    WebSocket/Relay     ┌──────────────────┐
│  Mobile App     │◄─────────────────────►│  Paseo Daemon    │
│  (Expo/RN)      │                        │  (Node.js)       │
│                 │                        │                  │
│  - View output  │                        │  - Spawn agents  │
│  - Send prompts │                        │  - Stream output │
│  - Voice input  │                        │  - Manage state  │
│  - Permissions  │                        │  - MCP server    │
└─────────────────┘                        └──────┬───────────┘
                                                  │
                                           ┌──────┴───────────┐
                                           │  Agent Providers  │
                                           ├──────────────────┤
                                           │  Claude Code     │ via @anthropic-ai/claude-agent-sdk
                                           │  Codex           │ via JSON-RPC over stdio
                                           │  OpenCode        │ via HTTP server
                                           └──────────────────┘
```

### What Already Exists (Strengths)

1. **Well-designed provider abstraction** (`AgentClient` / `AgentSession` interfaces in `agent-sdk-types.ts`)
   - `AgentClient`: createSession, resumeSession, listModels, isAvailable
   - `AgentSession`: stream, run, modes, permissions, commands, model switching
   - `AgentCapabilityFlags`: streaming, persistence, modes, MCP, reasoning, tools

2. **Provider registry pattern** (`provider-registry.ts`) - factory for all providers with runtime settings

3. **Unified streaming protocol** - all providers emit `AgentStreamEvent` via async generators

4. **Desktop shell already exists** (`packages/desktop`) - Tauri wrapper pointing at the Expo app

5. **Relay infrastructure** (`packages/relay`) - E2EE WebSocket bridge for remote access

6. **Rich timeline system** - tool calls, messages, reasoning, todos, errors, compaction

7. **Slash command support** - per-provider (`listCommands()` on AgentSession)

8. **Model selection per provider** - dynamic model switching, thinking options

9. **Permission system** - unified across providers, supports tool/plan/question kinds

---

## Proposed Vision: Desktop AI Tools Wrapper

Transform Paseo from "mobile monitoring app" to **"the desktop wrapper for all AI coding CLIs"** — think what iTerm is to shells, but for AI agents.

### Core Philosophy
- **Paseo doesn't replace the CLIs** — it wraps them, providing a unified UX layer
- **Every CLI feature should be accessible** — settings, slash commands, model selection, MCP configs
- **Seamless multi-tool workflow** — switch between Claude/Codex/Gemini/Copilot in one workspace

---

## Feature 1: Gemini CLI & GitHub Copilot Support

### How to Add New Providers

The existing `AgentClient`/`AgentSession` interface is solid. Adding a new provider requires:

#### File structure per provider:
```
packages/server/src/server/agent/providers/
├── gemini/
│   ├── gemini-agent.ts           # GeminiAgentClient + GeminiAgentSession
│   ├── tool-call-mapper.ts       # Map Gemini tool calls → ToolCallTimelineItem
│   └── tool-call-detail-parser.ts
├── copilot/
│   ├── copilot-agent.ts          # CopilotAgentClient + CopilotAgentSession
│   ├── tool-call-mapper.ts
│   └── tool-call-detail-parser.ts
```

#### Key implementation points:

**Gemini CLI (`gemini`):**
- Google's Gemini CLI is open source, Node-based
- Communication: Need to investigate — likely spawned as subprocess
- No official SDK for programmatic control yet (unlike Claude's agent-sdk)
- Approach: **Stdio-based interaction** similar to how Codex app-server works, OR terminal PTY wrapping
- Models: Gemini 2.5 Pro, Flash, etc.
- Modes: Likely `auto-approve` style flags

**GitHub Copilot CLI (`gh copilot`):**
- Part of the `gh` CLI extensions system
- Communication: stdin/stdout conversation
- More limited than Claude/Codex — primarily Q&A + code suggestions
- No streaming API or session persistence (as of current release)
- May need PTY-based terminal wrapping approach

#### Registration changes needed:

1. `provider-manifest.ts` — add provider definitions with modes, voice config
2. `provider-registry.ts` — register new clients in `buildProviderRegistry()`
3. `agent-sdk-types.ts` — `AgentProvider` type is already `string` (extensible)
4. `messages.ts` — `AgentProviderSchema` derives from manifest (auto-extends)

#### Recommended approach for new CLIs without SDKs:

```typescript
// Generic PTY-based provider for CLIs without programmatic APIs
// Wraps any CLI in a pseudo-terminal, parses output heuristically

class PtyAgentSession implements AgentSession {
  private pty: IPty;  // node-pty

  async *stream(prompt) {
    this.pty.write(prompt + '\n');
    // Parse output stream, detect tool calls, messages, etc.
    // Use heuristic parsers per provider
  }
}
```

This would serve as a **fallback adapter** for CLIs that don't have structured APIs.

### Effort Estimate
- Gemini: Medium (needs research on Gemini CLI's internal API/protocol)
- Copilot: Medium-High (limited programmatic control, may need PTY approach)
- Generic PTY adapter: High (robust output parsing is hard)

---

## Feature 2: Full CLI Feature Parity — Desktop Wrapper Architecture

### Current Gap

The app shows agent output and lets you send prompts, but **doesn't expose the full depth of each CLI's features**:
- Settings management (`.claude/settings.json`, `.codex/config.yaml`)
- Full slash command discovery and execution
- Provider-specific configuration (MCP servers, permissions, hooks)
- Project-level config (`.claude/`, `.codex/`, CLAUDE.md)

### Proposed Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Desktop App (Tauri + Expo Web)                                  │
│                                                                  │
│  ┌──────────────────────────────────┐  ┌───────────────────────┐ │
│  │  Workspace View                  │  │  Settings Panel       │ │
│  │                                  │  │                       │ │
│  │  ┌─────────┐ ┌─────────┐        │  │  - Provider configs   │ │
│  │  │ Agent 1 │ │ Agent 2 │ [+New] │  │  - MCP servers        │ │
│  │  │ Claude  │ │ Codex   │        │  │  - Model defaults     │ │
│  │  └────┬────┘ └────┬────┘        │  │  - Permission modes   │ │
│  │       │            │             │  │  - Hooks & prompts    │ │
│  │  ┌────▼────────────▼─────────┐  │  │  - Project settings   │ │
│  │  │  Chat / Stream View       │  │  │                       │ │
│  │  │                           │  │  └───────────────────────┘ │
│  │  │  - Message thread         │  │                             │
│  │  │  - Tool call cards        │  │  ┌───────────────────────┐ │
│  │  │  - Inline diffs           │  │  │  Command Palette      │ │
│  │  │  - Permission prompts     │  │  │  (Cmd+K / Cmd+P)      │ │
│  │  │                           │  │  │                       │ │
│  │  └───────────────────────────┘  │  │  - /slash commands    │ │
│  │                                  │  │  - Switch model       │ │
│  │  ┌───────────────────────────┐  │  │  - Switch mode        │ │
│  │  │  Input Area               │  │  │  - Switch provider    │ │
│  │  │  [Provider] [Model ▼]    │  │  │  - Open settings      │ │
│  │  │  [Mode ▼] [MCP ▼]       │  │  │  - Run tasks          │ │
│  │  │  ┌─────────────────────┐  │  │  └───────────────────────┘ │
│  │  │  │ Type your prompt... │  │  │                             │
│  │  │  └─────────────────────┘  │  │                             │
│  │  └───────────────────────────┘  │                             │
│  └──────────────────────────────────┘                            │
└──────────────────────────────────────────────────────────────────┘
```

### Key New Components

#### A. Command Palette (Cmd+K)
- Aggregates slash commands from all providers
- Quick model/mode switching
- Provider switching mid-conversation (if supported)
- Search through agent history

#### B. Settings Panel
- **Per-provider settings** - read/write provider config files
- **MCP server management** - add/remove/configure MCP servers per provider
- **Permission presets** - save common permission configurations
- **Project settings** - edit CLAUDE.md, .codex/config.yaml from UI

#### C. Enhanced Input Area
- Inline provider/model selector (already partially exists)
- Slash command autocomplete with provider context
- File/image drag-and-drop (already exists)
- Command history (up arrow)

### Server-Side Changes Needed

Add new WS messages for settings management:

```typescript
// New message types in messages.ts
type SettingsInboundMessage =
  | { type: "read_provider_settings"; provider: AgentProvider }
  | { type: "write_provider_settings"; provider: AgentProvider; settings: Record<string, unknown> }
  | { type: "list_mcp_servers"; provider?: AgentProvider }
  | { type: "update_mcp_server"; name: string; config: McpServerConfig }
  | { type: "read_project_config"; cwd: string; provider?: AgentProvider }
  | { type: "write_project_config"; cwd: string; provider: AgentProvider; content: string }

type SettingsOutboundMessage =
  | { type: "provider_settings"; provider: AgentProvider; settings: Record<string, unknown> }
  | { type: "mcp_servers_list"; servers: Record<string, McpServerConfig & { provider: string }> }
  | { type: "project_config"; cwd: string; provider: AgentProvider; content: string }
```

---

## Feature 3: User Workflow Design

### Option A: Provider-First Flow (Recommended)

```
1. User opens Paseo → sees Workspace with project folders
2. Clicks [+ New Agent] or types in input area
3. Input area shows: [Provider ▼ Claude] [Model ▼ Sonnet 4.5] [Mode ▼ Auto]
4. User types prompt → agent launches with selected config
5. During session: can switch model/mode via Cmd+K or inline selectors
6. Can open another tab with different provider for same project
```

**Why provider-first:** Each provider has different capabilities, modes, and models. Users who use AI coding tools already think in terms of "I'll use Claude for this" or "Codex for that."

### Option B: Model-First Flow (Like Perplexity)

```
1. User opens Paseo → sees chat input with model selector
2. Picks model (Sonnet 4.5 / GPT-5.1 / Gemini 2.5 Pro)
3. Provider is auto-resolved from model
4. User types → agent launches
```

**Problem:** Model names don't uniquely identify providers. "Sonnet 4.5" → Claude is obvious, but future cross-provider models break this.

### Option C: Hybrid Flow (Recommended for V2)

```
1. Quick start: Just type in the input area (uses default provider+model)
2. Power users: Click provider/model selectors to customize
3. Command palette (Cmd+K): Switch anything mid-session
4. Keyboard shortcuts: Cmd+1/2/3 to switch providers
```

### Model Council Feature

Inspired by Perplexity's multi-model approach:

```
┌─────────────────────────────────────────────────────┐
│  Model Council Mode                                  │
│                                                      │
│  Prompt: "Refactor the auth module to use JWT"       │
│                                                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │ Claude       │ │ Codex       │ │ Gemini      │   │
│  │ Sonnet 4.5   │ │ GPT-5.1     │ │ 2.5 Pro     │   │
│  │              │ │             │ │             │   │
│  │ [Response    │ │ [Response   │ │ [Response   │   │
│  │  streaming]  │ │  streaming] │ │  streaming] │   │
│  │              │ │             │ │             │   │
│  └─────────────┘ └─────────────┘ └─────────────┘   │
│                                                      │
│  [Accept Claude's] [Accept Codex's] [Merge & Edit]  │
└─────────────────────────────────────────────────────┘
```

Implementation approach:
- Send same prompt to N providers simultaneously (already have multi-agent via `AgentManager`)
- Display responses side-by-side
- Let user pick which response to apply, or manually merge
- **Use case:** Architecture decisions, code review, getting multiple perspectives

### Workflow Summary

```
Default flow:
  Open app → Pick workspace → Type prompt → Agent runs (default provider)

Power user flow:
  Open app → Cmd+K → Select provider/model → Type prompt → Agent runs

Council flow:
  Open app → Enable council mode → Type prompt → Multiple agents respond → Pick best

Mid-session:
  Cmd+K → Switch model (same provider) or Switch provider (new agent tab)
  /command → Execute provider-specific slash commands
  Cmd+, → Open settings for current provider
```

---

## Feature 4: Remote Session Support

### Current State

- Paseo already has **relay infrastructure** (`packages/relay`) for remote access
- The relay bridges WebSocket connections with E2EE (end-to-end encryption)
- The mobile app already connects to daemons over the relay
- Claude Code natively supports `--resume` which enables remote session continuity

### What "Remote Session" Means

Two distinct concepts:

#### A. Remote Daemon Access (Already Exists)
```
Phone/Laptop ──► Relay (Cloudflare) ──► Your Desktop Daemon ──► Local Agents
```
This **already works**. The relay + E2EE system handles this.

#### B. Remote Agent Execution (New)
```
Desktop App ──► Daemon ──► SSH to Remote Machine ──► Agent runs there
```
This is about running agents on **remote machines** (cloud VMs, dev containers, SSH hosts).

### Analysis: Is Remote Agent Execution Worth It?

**Pros:**
- Run agents on powerful remote machines (more CPU/memory)
- Work with codebases that live on remote servers
- Cloud development environments (GitHub Codespaces, etc.)
- Claude Code already supports `claude --resume` for session continuity

**Cons:**
- Significant complexity: SSH tunneling, process management, state sync
- Each provider handles remote differently (or not at all)
- Latency issues for real-time streaming
- Authentication complexity (SSH keys, cloud auth)

**Recommendation: Defer to V2, leverage existing relay.**

The relay already provides remote access to your local daemon. For the "run on remote machine" case, it's better to:
1. Install Paseo daemon on the remote machine
2. Connect to it via relay (already works)
3. Agents run locally on that remote machine

This is architecturally simpler and already supported. True remote agent spawning (SSH → spawn Claude on another machine) adds complexity for marginal benefit.

### If We Do Implement Remote Execution (V2)

```typescript
// Extension to AgentSessionConfig
export type AgentSessionConfig = {
  // ... existing fields
  remote?: {
    type: "ssh";
    host: string;
    port?: number;
    user?: string;
    identityFile?: string;
  } | {
    type: "container";
    containerId: string;
    runtime?: "docker" | "podman";
  };
};

// Each provider would need to support remote spawning:
// Claude: SSH + `claude --resume <id>` on remote
// Codex: SSH + `codex app-server` on remote
// OpenCode: SSH + `opencode serve --port <port>` on remote
```

---

## Implementation Phases

### Phase 1: Foundation (2-3 weeks)
- [ ] Command palette infrastructure (Cmd+K)
- [ ] Enhanced provider/model/mode selectors in input area
- [ ] Settings panel scaffold (read-only view of provider configs)
- [ ] Slash command autocomplete from `listCommands()`

### Phase 2: New Providers (3-4 weeks per provider)
- [ ] Gemini CLI provider (research CLI API first)
- [ ] GitHub Copilot provider (likely PTY-based)
- [ ] Generic PTY adapter for unsupported CLIs

### Phase 3: Full Settings Management (2-3 weeks)
- [ ] Read/write provider settings via daemon
- [ ] MCP server management UI
- [ ] Project config editor (CLAUDE.md, etc.)
- [ ] Permission preset management

### Phase 4: Model Council (2 weeks)
- [ ] Multi-agent split view
- [ ] Simultaneous prompt dispatch
- [ ] Response comparison and selection UI

### Phase 5: Remote Execution (3-4 weeks, V2)
- [ ] SSH-based remote agent spawning
- [ ] Container-based execution
- [ ] Remote state synchronization

---

## Files to Modify (Phase 1)

### Server
- `packages/server/src/server/agent/provider-manifest.ts` — new providers
- `packages/server/src/server/agent/provider-registry.ts` — register providers
- `packages/server/src/shared/messages.ts` — settings/config messages
- `packages/server/src/server/session.ts` — handle settings messages

### App
- `packages/app/src/screens/agent/draft-agent-screen.tsx` — enhanced selectors
- `packages/app/src/components/agent-input-area.tsx` — command palette trigger
- New: `packages/app/src/components/command-palette.tsx`
- New: `packages/app/src/screens/settings/provider-settings-screen.tsx`
- New: `packages/app/src/components/model-council-view.tsx`

### Desktop
- `packages/desktop/src-tauri/src/lib.rs` — keyboard shortcut registration (Cmd+K)
