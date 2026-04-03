# How OpenWork Launches and Controls OpenCode

> Detailed step-by-step architectural documentation of how OpenWork integrates with, launches, and controls OpenCode.

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Component Roles](#component-roles)
- [Launch Paths](#launch-paths)
  - [Path A: Desktop App (Tauri)](#path-a-desktop-app-tauri)
  - [Path B: Orchestrator CLI](#path-b-orchestrator-cli)
- [Step-by-Step: Desktop Launch Flow](#step-by-step-desktop-launch-flow)
- [Step-by-Step: Orchestrator Launch Flow](#step-by-step-orchestrator-launch-flow)
- [Process Spawning Details](#process-spawning-details)
  - [OpenCode Engine Spawning](#opencode-engine-spawning)
  - [OpenWork Server Spawning](#openwork-server-spawning)
  - [OpenCode Router Spawning](#opencode-router-spawning)
- [Communication Architecture](#communication-architecture)
  - [Tauri IPC (Frontend ↔ Rust Backend)](#tauri-ipc-frontend--rust-backend)
  - [HTTP REST (App ↔ OpenWork Server)](#http-rest-app--openwork-server)
  - [HTTP REST (App/Server ↔ OpenCode)](#http-rest-appserver--opencode)
  - [Child Process Events](#child-process-events)
- [Port Allocation Strategy](#port-allocation-strategy)
- [Authentication and Credentials](#authentication-and-credentials)
- [Database Access Patterns](#database-access-patterns)
- [Health Checks and Readiness](#health-checks-and-readiness)
- [Process Lifecycle Management](#process-lifecycle-management)
  - [Startup Sequencing](#startup-sequencing)
  - [Graceful Shutdown](#graceful-shutdown)
  - [Runtime Upgrades](#runtime-upgrades)
  - [Hot Reload](#hot-reload)
- [Sandbox Mode](#sandbox-mode)
- [Environment Variables Reference](#environment-variables-reference)
- [Key Source Files](#key-source-files)

---

## Overview

OpenWork is a multi-layer system where **OpenCode is the AI execution engine** and **OpenWork is the experience layer**. OpenWork does not embed OpenCode as a library — it spawns OpenCode as a separate child process and communicates with it over HTTP REST APIs.

The core integration pattern is:

```
User → OpenWork UI → Tauri/Orchestrator → spawn OpenCode process → HTTP API → AI pipeline
```

OpenWork manages three coordinated services:

| Service | Role | Binary |
|---------|------|--------|
| **OpenCode** | AI code execution engine | `opencode` |
| **OpenWork Server** | API layer (config, workspace, proxy) | `openwork-server` |
| **OpenCode Router** | Message bridge (Telegram, Slack) | `opencode-router` |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Desktop App (SolidJS)                     │
│                                                                  │
│  ┌──────────┐  ┌───────────────────┐  ┌──────────────────────┐  │
│  │ Chat UI  │  │ Workspace Manager │  │ Settings / Config    │  │
│  └────┬─────┘  └────────┬──────────┘  └──────────┬───────────┘  │
│       │                 │                         │              │
│       ▼                 ▼                         ▼              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              lib/tauri.ts  (Tauri invoke())               │   │
│  │              lib/opencode.ts  (OpenCode SDK)              │   │
│  │              lib/openwork-server.ts  (Server client)      │   │
│  └────────────────────────┬─────────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────────┘
                            │ Tauri IPC
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                    Tauri Backend (Rust)                            │
│                                                                   │
│  ┌────────────────┐ ┌──────────────────┐ ┌─────────────────────┐ │
│  │ EngineManager  │ │ OpenworkServer   │ │ OpenCodeRouter      │ │
│  │                │ │ Manager          │ │ Manager             │ │
│  │ • spawn/stop   │ │ • spawn/stop     │ │ • spawn/stop        │ │
│  │ • state track  │ │ • port resolve   │ │ • health check      │ │
│  └───────┬────────┘ └────────┬─────────┘ └──────────┬──────────┘ │
└──────────┼───────────────────┼──────────────────────┼────────────┘
           │                   │                      │
     spawn process       spawn process          spawn process
           │                   │                      │
           ▼                   ▼                      ▼
    ┌──────────────┐  ┌────────────────┐  ┌────────────────────┐
    │   OpenCode   │  │  OpenWork      │  │  OpenCode Router   │
    │   Engine     │  │  Server        │  │                    │
    │              │  │                │  │  • Telegram bridge  │
    │  HTTP :4096  │  │  HTTP :8787    │  │  • Slack bridge     │
    │  (random)    │  │  (48000-51000) │  │  • Health :3005     │
    └──────────────┘  └────────────────┘  └────────────────────┘
```

---

## Component Roles

### Frontend App (`apps/app/`)

- **Technology**: SolidJS + TailwindCSS
- **Role**: Thin UI layer — renders chat, workspaces, settings
- **Integration**: Calls Tauri backend via `invoke()` for process control; calls OpenCode and OpenWork Server directly via HTTP for data operations
- **Key files**:
  - `src/app/lib/tauri.ts` — Tauri command invocations
  - `src/app/lib/opencode.ts` — OpenCode SDK client wrapper
  - `src/app/lib/openwork-server.ts` — OpenWork Server HTTP client

### Tauri Backend (`apps/desktop/src-tauri/`)

- **Technology**: Rust + Tauri 2.x
- **Role**: Native process manager — spawns, monitors, and stops child processes
- **Responsibilities**: Binary resolution, port allocation, environment setup, sidecar management, credential generation
- **Key modules**:
  - `src/engine/` — OpenCode engine lifecycle
  - `src/openwork_server/` — OpenWork Server lifecycle
  - `src/opencode_router/` — Router lifecycle
  - `src/orchestrator/` — Orchestrator daemon management
  - `src/commands/` — Tauri command handlers (IPC endpoints)

### OpenWork Server (`apps/server/`)

- **Technology**: Bun/Node.js HTTP server
- **Role**: Core API layer between the frontend and OpenCode
- **Responsibilities**: Workspace management, config file management, OpenCode database access, proxy requests to OpenCode, authentication, audit logging, reload event tracking
- **Key files**:
  - `src/server.ts` — Main server with routing and proxy logic
  - `src/opencode-db.ts` — SQLite database path resolution and session seeding

### Orchestrator CLI (`apps/orchestrator/`)

- **Technology**: Node.js CLI
- **Role**: Standalone process manager (alternative to Tauri for headless/server deployments)
- **Responsibilities**: Same as Tauri backend but runs as a CLI with optional TUI dashboard
- **Key file**: `src/cli.ts` — All process management logic

### OpenCode Router (`apps/opencode-router/`)

- **Technology**: Bun/Node.js
- **Role**: Routes messages from Telegram/Slack to OpenCode workspaces
- **Key files**: `src/bridge.ts`, `src/opencode.ts`, `src/cli.ts`

---

## Launch Paths

OpenWork provides two runtime paths to launch and control OpenCode:

### Path A: Desktop App (Tauri)

```
User clicks "Start" in Desktop App
  → SolidJS frontend calls invoke("engine_start", {...})
  → Tauri Rust backend receives command
  → Rust code spawns OpenCode as child process via tauri_plugin_shell
  → Rust code also spawns OpenWork Server and OpenCode Router
  → Returns EngineInfo to frontend
```

### Path B: Orchestrator CLI

```
User runs: openwork start --workspace /path
  → Node.js CLI parses args
  → cli.ts spawnProcess() calls child_process.spawn()
  → Spawns OpenCode, OpenWork Server, and OpenCode Router
  → Monitors health and displays TUI dashboard
```

Both paths produce the same result: three coordinated processes communicating over HTTP.

---

## Step-by-Step: Desktop Launch Flow

This section traces the exact code path when a user starts OpenCode from the desktop app.

### Step 1: User Triggers Start

The frontend calls the Tauri backend:

```typescript
// apps/app/src/app/lib/tauri.ts
const info = await invoke<EngineInfo>("engine_start", {
  projectDir: "/path/to/workspace",
  preferSidecar: true,
  runtime: "direct",           // or "openwork-orchestrator"
  workspacePaths: ["/path1", "/path2"],
  opencodeBinPath: null,       // auto-resolve
});
```

### Step 2: Tauri Backend Receives Command

The Rust command handler validates inputs and prepares the launch:

```
File: apps/desktop/src-tauri/src/commands/engine.rs → engine_start()
```

1. **Validates** `project_dir` is non-empty
2. **Creates** `project_dir` if it doesn't exist
3. **Creates/validates** `opencode.jsonc` config file in the workspace
4. **Deduplicates** workspace paths, prepending the primary `project_dir`
5. **Allocates a free port** via `find_free_port()` (binds to `127.0.0.1:0`)
6. **Generates credentials**: 512-character random username and password using UUIDs
7. **Detects dev mode** from `OPENWORK_DEV_MODE` environment variable
8. **Stops any existing engine** by calling `EngineManager::stop_locked()`

### Step 3: Binary Resolution

The system locates the OpenCode binary using a priority chain:

```
1. Bundled sidecar: app.shell().sidecar("opencode")
2. System PATH: app.shell().command(program_path)
```

If neither is found, the user receives an error with installation instructions.

### Step 4: OpenCode Process Spawning

```
File: apps/desktop/src-tauri/src/engine/spawn.rs → spawn_engine()
```

The Tauri backend constructs and spawns the OpenCode process:

**Command line**:
```bash
opencode serve --hostname 127.0.0.1 --port <allocated> --cors "*"
```

**Environment variables set on the child process**:

| Variable | Value | Purpose |
|----------|-------|---------|
| `OPENCODE_CLIENT` | `openwork` | Identifies the caller to OpenCode |
| `OPENWORK` | `1` | Marks OpenWork integration mode |
| `OPENCODE_SERVER_USERNAME` | `<random-512-char>` | HTTP Basic auth username |
| `OPENCODE_SERVER_PASSWORD` | `<random-512-char>` | HTTP Basic auth password |
| `XDG_CONFIG_HOME` | Platform path | Config directory override (dev mode) |
| `XDG_DATA_HOME` | Platform path | Data directory override (dev mode) |
| `XDG_CACHE_HOME` | Platform path | Cache directory override (dev mode) |
| `XDG_STATE_HOME` | Platform path | State directory override (dev mode) |
| `OPENCODE_CONFIG_DIR` | Workspace path | Config directory for this workspace |

**Dev mode isolation**: When `OPENWORK_DEV_MODE=1`, creates an isolated directory structure under `{app_data_dir}/openwork-dev-data/` with separate XDG directories. This prevents development from interfering with production data.

**Spawning mechanism**: Uses `tauri_plugin_shell` which wraps platform-native process creation:
```rust
let (rx, child) = command.spawn().map_err(|e| format!("Failed to spawn engine: {e}"))?;
```

Returns a `(Receiver<CommandEvent>, CommandChild)` tuple — the receiver provides stdout/stderr events, and the child handle allows process control.

### Step 5: Warmup Wait

After spawning, the Tauri backend waits up to **2 seconds** for the process to stabilize:

```rust
let warmup_deadline = Instant::now() + Duration::from_secs(2);
loop {
    if output.exited { return Err("Engine exited during startup"); }
    if Instant::now() >= warmup_deadline { break; }
    thread::sleep(Duration::from_millis(150));
}
```

If the process exits during this window, the error includes captured stdout/stderr for diagnosis.

### Step 6: OpenWork Server Spawning

Concurrently with engine stabilization, the system spawns the OpenWork Server:

```
File: apps/desktop/src-tauri/src/openwork_server/spawn.rs → spawn_openwork_server()
```

**Command line**:
```bash
openwork-server \
  --host 127.0.0.1 \
  --port <48000-51000> \
  --workspace /path/to/workspace \
  --cors "*" \
  --approval auto \
  --opencode-base-url http://127.0.0.1:<opencode-port>
```

The server connects to OpenCode's HTTP API and acts as a proxy/middleware for the frontend.

### Step 7: OpenCode Router Spawning (Optional)

If messaging integration is configured:

```
File: apps/desktop/src-tauri/src/opencode_router/spawn.rs → spawn_opencode_router()
```

**Command line**:
```bash
opencode-router serve /path/to/workspace --opencode-url http://127.0.0.1:<opencode-port>
```

### Step 8: Return Engine Info

The Tauri backend stores the engine state and returns `EngineInfo` to the frontend:

```typescript
type EngineInfo = {
  running: boolean;
  pid: number | null;
  port: number | null;
  base_url: string | null;   // e.g., "http://127.0.0.1:4096"
  hostname: string | null;
  project_dir: string | null;
  runtime: "direct" | "openwork-orchestrator";
  last_stdout: string | null;
  last_stderr: string | null;
};
```

### Step 9: Frontend Connects

The frontend creates an OpenCode SDK client and begins communicating:

```typescript
// apps/app/src/app/lib/opencode.ts
const client = createClient(
  "http://127.0.0.1:4096",  // base_url from EngineInfo
  "/path/to/workspace",      // directory header
  { username, password, mode: "basic" }
);
```

Session creation, prompts, and commands now flow through the OpenCode HTTP API.

---

## Step-by-Step: Orchestrator Launch Flow

The orchestrator (`openwork start`) follows a similar but distinct path.

### Step 1: CLI Entry

```bash
openwork start --workspace /path/to/project --approval auto
```

```
File: apps/orchestrator/src/cli.ts → main() → runStart()
```

### Step 2: Configuration

1. **Parses CLI flags**: output mode, logging, TUI, sandbox, auth
2. **Creates logger**: silent if TUI mode, stdout otherwise
3. **Initializes TUI** (optional): via `startOrchestratorTui()`

### Step 3: Port Allocation

```typescript
const opencodePort = await resolvePort(explicit, "127.0.0.1");
const openworkPort = await resolvePort(explicit, "127.0.0.1");
const opencodeRouterHealthPort = await resolvePort(undefined, "127.0.0.1");
const controlPort = await resolvePort(undefined, "127.0.0.1");
```

Each port is verified via TCP bind test (`canBind()`).

### Step 4: Binary Resolution

The orchestrator resolves binaries with a priority chain:

1. **Explicit** — user-provided `--opencode-bin` path
2. **Bundled** — compiled into orchestrator package
3. **Downloaded** — fetched from GitHub releases via sidecar manifest
4. **External** — resolved from system PATH or `node_modules`

### Step 5: Credential Generation

```typescript
const credentials = resolveManagedOpencodeCredentials(args);
// Falls back to random credentials if not provided
```

### Step 6: Control Server

A lightweight HTTP server is started for runtime management:

```
GET  /runtime/versions → service snapshot
POST /runtime/upgrade  → trigger service restarts
```

Protected by a random Bearer token.

### Step 7: Service Startup (Ordered)

Services are started in a specific order with health checks between each:

**7a. Start OpenCode**:
```bash
opencode serve --hostname 127.0.0.1 --port <port> --cors <origins>
```

Environment: `OPENCODE_CLIENT=openwork-orchestrator`, `OPENWORK=1`, plus credentials, hot reload settings, and telemetry attributes.

Wait for health: polls `client.global.health()` until responsive.

**7b. Start OpenCode Router** (optional):
```bash
opencode-router serve /workspace --opencode-url http://127.0.0.1:<port>
```

Wait for health: polls `http://127.0.0.1:<health-port>/health`.

**7c. Start OpenWork Server**:
```bash
openwork-server --host 127.0.0.1 --port <port> --workspace /path \
  --approval auto --opencode-base-url http://127.0.0.1:<opencode-port>
```

Wait for health: polls `/health` endpoint, then verifies server with token issuance.

### Step 8: Continuous Monitoring

- **Process exit handlers**: if any service exits unexpectedly, all services are shut down
- **Router health polling**: every 15 seconds, checks router health via OpenWork Server
- **TUI updates**: service status reflected in terminal dashboard

---

## Process Spawning Details

### OpenCode Engine Spawning

| Aspect | Desktop (Tauri) | Orchestrator (CLI) |
|--------|----------------|-------------------|
| **File** | `engine/spawn.rs` | `cli.ts → startOpencode()` |
| **Mechanism** | `tauri_plugin_shell::spawn()` | `child_process.spawn()` |
| **Command** | `opencode serve --hostname H --port P --cors "*"` | Same |
| **Binary source** | Sidecar or system PATH | Bundled, downloaded, or external |
| **Env: OPENCODE_CLIENT** | `openwork` | `openwork-orchestrator` |
| **Env: OPENWORK** | `1` | `1` |
| **Credentials** | 512-char UUID-based | Configurable length random |
| **Output capture** | `CommandEvent` stream, truncated to 8000 chars | `prefixStream()` through logger |

### OpenWork Server Spawning

| Aspect | Desktop (Tauri) | Orchestrator (CLI) |
|--------|----------------|-------------------|
| **File** | `openwork_server/spawn.rs` | `cli.ts → startOpenworkServer()` |
| **Port range** | 48000–51000 (randomized start) | Auto-resolved free port |
| **Auth tokens** | `OPENWORK_TOKEN`, `OPENWORK_HOST_TOKEN` | Same |
| **Approval mode** | `auto` | Configurable (`--approval`) |
| **OpenCode connection** | `--opencode-base-url` CLI arg | Same + env vars |

### OpenCode Router Spawning

| Aspect | Desktop (Tauri) | Orchestrator (CLI) |
|--------|----------------|-------------------|
| **File** | `opencode_router/spawn.rs` | `cli.ts → startOpenCodeRouter()` |
| **Health port** | Ephemeral (`127.0.0.1:0`) | Ephemeral |
| **Feature detection** | N/A | Probes `--help` for `--opencode-url` support |
| **Failure behavior** | Logged, non-fatal | Non-fatal unless `--opencode-router-required` |

---

## Communication Architecture

### Tauri IPC (Frontend ↔ Rust Backend)

**Transport**: Tauri's built-in `invoke()` + event system  
**Protocol**: JSON serialization over IPC  

Key commands exposed by the Tauri backend:

| Command | Purpose |
|---------|---------|
| `engine_start` | Spawn OpenCode + side services |
| `engine_stop` | Kill all managed processes |
| `engine_restart` | Stop then start with same config |
| `engine_info` | Get current engine state snapshot |
| `engine_doctor` | Diagnostic check (binary, version, capabilities) |
| `engine_install` | Install OpenCode binary |
| `openwork_server_info` | Get server status |
| `openwork_server_restart` | Restart OpenWork Server |
| `orchestrator_status` | Get orchestrator daemon status |
| `orchestrator_workspace_activate` | Switch active workspace |
| `opencodeRouter_start` | Start the message router |
| `opencodeRouter_stop` | Stop the message router |
| `opencodeRouter_info` | Get router status and health |

### HTTP REST (App ↔ OpenWork Server)

**Base URL**: `http://127.0.0.1:<port>` (typically 48000–51000 range)  
**Auth**: `Authorization: Bearer <OPENWORK_TOKEN>`  

Key endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Server health check |
| `/workspaces` | GET | List workspaces |
| `/workspace/{id}/config` | GET/PUT | Read/write OpenCode config |
| `/workspace/{id}/skills` | GET/POST | Manage skills |
| `/workspace/{id}/sessions/{sid}` | DELETE | Delete session |
| `/workspace/{id}/export` | GET | Export workspace |
| `/workspace/{id}/import` | POST | Import workspace |
| `/workspace/{id}/blueprint/sessions/materialize` | POST | Seed blueprint sessions |
| `/opencode-router/health` | GET | Router health (proxied) |

### HTTP REST (App/Server ↔ OpenCode)

**Base URL**: `http://127.0.0.1:<port>` (dynamically allocated)  
**Auth**: HTTP Basic (`OPENCODE_SERVER_USERNAME:OPENCODE_SERVER_PASSWORD`)  
**SDK**: `@opencode-ai/sdk/v2/client`  

Key endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/global/health` | GET | Health check |
| `/session` | POST | Create session |
| `/session/{id}/prompt_async` | POST | Stream AI response |
| `/session/{id}/command` | POST | Execute command |

**Directory header**: `x-opencode-directory` is set on each request to scope OpenCode to the correct workspace path.

**Proxy pattern**: The OpenWork Server proxies many requests to OpenCode, injecting the correct credentials and directory header:

```
Frontend → OpenWork Server (/w/{workspaceId}/opencode/*) → OpenCode (/*) 
```

### Child Process Events

Both Tauri and the orchestrator capture process output:

**Tauri** (Rust):
```rust
// Event types from tauri_plugin_shell::process::CommandEvent
CommandEvent::Stdout(bytes)   // stdout data
CommandEvent::Stderr(bytes)   // stderr data  
CommandEvent::Terminated(payload)  // process exit
CommandEvent::Error(message)  // spawn error
```

**Orchestrator** (Node.js):
```typescript
prefixStream(child.stdout, "opencode", "stdout", logger, child.pid);
prefixStream(child.stderr, "opencode", "stderr", logger, child.pid);
child.on("exit", (code, signal) => handleExit("opencode", code, signal));
child.on("error", (error) => handleSpawnError("opencode", error));
```

---

## Port Allocation Strategy

Each service gets a dynamically allocated port to avoid conflicts:

### OpenCode Engine

```rust
// apps/desktop/src-tauri/src/engine/spawn.rs
fn find_free_port() -> Result<u16, String> {
    // Binds to 127.0.0.1:0, OS assigns an ephemeral port
    let listener = TcpListener::bind("127.0.0.1:0")?;
    Ok(listener.local_addr()?.port())
}
```

### OpenWork Server

```rust
// apps/desktop/src-tauri/src/openwork_server/spawn.rs
// Range: 48000–51000 (3001 ports)
fn resolve_openwork_port(host, preferred, reserved) -> Result<u16, String> {
    // 1. Try preferred port (if provided and free)
    // 2. Randomized scan through 48000–51000 range
    // 3. Fallback: 32 attempts at ephemeral ports
}
```

The randomized start prevents port collisions when multiple OpenWork instances run simultaneously.

### OpenCode Router Health

```rust
// apps/desktop/src-tauri/src/opencode_router/spawn.rs
fn resolve_opencode_router_health_port() -> Result<u16, String> {
    // Ephemeral port via 127.0.0.1:0
}
```

### Port Sharing Between Services

All ports are passed between services so they can discover each other:

```
engine_start()
  ├─ opencode_port ──────────► OpenWork Server (--opencode-base-url)
  ├─ opencode_port ──────────► OpenCode Router (--opencode-url)
  ├─ openwork_port ──────────► Frontend (OpenworkServerInfo.port)
  └─ router_health_port ────► OpenWork Server (OPENCODE_ROUTER_HEALTH_PORT)
                             ► OpenCode Router (OPENCODE_ROUTER_HEALTH_PORT)
```

---

## Authentication and Credentials

### OpenCode Authentication

OpenWork generates **per-session credentials** for OpenCode:

- **Desktop**: 512-character random strings from concatenated UUIDs
- **Orchestrator**: Configurable length random credentials

These are passed to OpenCode via environment variables (`OPENCODE_SERVER_USERNAME`, `OPENCODE_SERVER_PASSWORD`) and used by clients for HTTP Basic auth.

### OpenWork Server Authentication

Two-tier token system:

| Token | Header | Scope |
|-------|--------|-------|
| `OPENWORK_TOKEN` | `Authorization: Bearer <token>` | Client operations (read, write) |
| `OPENWORK_HOST_TOKEN` | `X-OpenWork-Host-Token: <token>` | Host-level operations (admin) |

Tokens are generated as random UUIDs and passed via environment variables.

### Credential Flow

```
Tauri Backend
  ├─ Generates: opencode_username, opencode_password
  ├─ Generates: openwork_token, openwork_host_token
  │
  ├─► OpenCode process (via env: OPENCODE_SERVER_USERNAME/PASSWORD)
  ├─► OpenWork Server (via env: OPENWORK_TOKEN, OPENWORK_HOST_TOKEN,
  │                          OPENWORK_OPENCODE_USERNAME/PASSWORD)
  ├─► OpenCode Router (via env: OPENCODE_SERVER_USERNAME/PASSWORD)
  └─► Frontend (via EngineInfo / OpenworkServerInfo responses)
```

### Frontend Auth Resolution

```typescript
// apps/app/src/app/lib/opencode.ts
// "openwork" mode → Bearer token; "basic" mode → Basic auth
const resolveAuthHeader = (auth?: OpencodeAuth) => {
  if (auth?.mode === "openwork" && auth.token) return `Bearer ${auth.token}`;
  const encoded = encodeBasicAuth(auth);
  return encoded ? `Basic ${encoded}` : null;
};
```

---

## Database Access Patterns

OpenWork reads OpenCode's SQLite database for session and message data.

### Database Path Resolution

```
File: apps/server/src/opencode-db.ts → resolveOpencodeDbPath()
```

The resolution follows a multi-location search:

1. Check `OPENCODE_DB` environment variable (absolute or relative path)
2. Check `OPENCODE_CHANNEL` for channel-specific naming (`opencode-{channel}.db`)
3. Search platform-specific data directories:
   - **Linux**: `~/.local/share/opencode/`
   - **macOS**: `~/Library/Application Support/opencode/`
   - **Windows**: `%APPDATA%/opencode/`
4. Fall back to `~/.local/share/opencode/opencode.db`

### Channel-Aware Naming

```typescript
function preferredDbNames(): string[] {
  const channel = process.env.OPENCODE_CHANNEL?.trim() || "local";
  // "latest" or "beta" channels use generic "opencode.db"
  // Other channels use "opencode-{channel}.db" with fallback to "opencode.db"
  return channel === "latest" || channel === "beta"
    ? ["opencode.db"]
    : [`opencode-${channel}.db`, "opencode.db"];
}
```

### Session Seeding

OpenWork can seed initial messages into OpenCode sessions for blueprint/template workflows:

```typescript
seedOpencodeSessionMessages({
  sessionId: "abc-123",
  workspaceRoot: "/path/to/workspace",
  messages: [{ role: "user", content: "..." }],
});
// Inserts session + message + part records into SQLite
// Idempotent: skips if messages already exist for the session
```

---

## Health Checks and Readiness

### OpenCode Health

**Endpoint**: `GET /global/health`  
**Client**: `@opencode-ai/sdk` → `client.global.health()`

```typescript
// apps/app/src/app/lib/opencode.ts
export async function waitForHealthy(client, options?) {
  const timeoutMs = options?.timeoutMs ?? 10_000;
  const pollMs = options?.pollMs ?? 250;
  // Polls every 250ms until health endpoint responds or timeout
}
```

### OpenWork Server Health

**Endpoint**: `GET /health`  
**Response**: `{ ok: boolean; version: string; uptimeMs: number }`

### OpenCode Router Health

**Endpoint**: `GET http://127.0.0.1:<health-port>/health`  
**Response**: JSON with OpenCode connectivity, Telegram/Slack channel status

The orchestrator polls router health every **15 seconds** and updates the TUI.

### Startup Health Sequence

```
1. Spawn OpenCode ──► Poll /global/health (timeout: 10s, poll: 250ms)
2. Spawn Router ────► Poll /health (timeout: 10s, poll: 500ms)
3. Spawn Server ────► Poll /health + verify token issuance
```

---

## Process Lifecycle Management

### Startup Sequencing

Services must start in order because of dependency relationships:

```
OpenCode (must be running first)
  └─► OpenCode Router (needs OpenCode URL)
  └─► OpenWork Server (needs OpenCode URL for proxy)
```

### Graceful Shutdown

**Desktop (Tauri)**:
```rust
fn stop_locked(state: &mut EngineState) {
    if let Some(ref mut child) = state.child {
        child.kill();  // Immediate SIGKILL
    }
    state.child_exited = true;
    // Reset all state fields
}
```

**Orchestrator (CLI)**:
```typescript
async function stopChild(child, timeoutMs = 2500) {
  child.kill("SIGTERM");           // Step 1: Graceful signal
  const exited = await race(
    once(child, "exit"),           // Wait for exit
    sleep(timeoutMs)               // Timeout: 2.5s
  );
  if (!exited) child.kill("SIGKILL");  // Step 2: Force kill
}
```

The orchestrator's `shutdown()` function:
1. Sets `shuttingDown = true`
2. Clears health check intervals
3. Closes control HTTP server
4. Stops sandbox container (if active)
5. Stops all children in parallel: `Promise.all(children.map(stopChild))`

**Signal handlers**:
```typescript
process.on("SIGINT",  () => shutdown().then(() => process.exit(0)));
process.on("SIGTERM", () => shutdown().then(() => process.exit(0)));
```

### Runtime Upgrades

The orchestrator supports upgrading individual services without full restart:

```
POST /runtime/upgrade
Authorization: Bearer <control-token>
Body: { "services": ["opencode", "openwork-server"] }
```

Upgrade sequence:
1. Mark service as "restarting" (prevents shutdown cascade)
2. If source is "external": run `npm install` for latest versions
3. Re-resolve binary paths
4. Stop old process with `stopChild()`
5. Start new process with updated binary
6. Wait for health check
7. Update runtime state

### Hot Reload

OpenCode supports hot reload for configuration changes:

```typescript
// Environment variables passed to OpenCode
OPENCODE_HOT_RELOAD = "1"
OPENCODE_HOT_RELOAD_DEBOUNCE_MS = "700"    // Wait 700ms after last change
OPENCODE_HOT_RELOAD_COOLDOWN_MS = "1500"   // Minimum 1.5s between reloads
```

When files in `.opencode/` or `opencode.json` change, OpenCode automatically reloads without requiring a full process restart.

---

## Sandbox Mode

The orchestrator can run services in isolated containers:

### Docker Sandbox

```bash
openwork start --workspace /path --sandbox docker
```

All three services run inside a single Docker container:

```
┌─────────────── Docker Container ───────────────┐
│                                                  │
│  OpenCode (:4096)                               │
│  OpenWork Server (:8787) ←──── port mapped ──►  │
│  OpenCode Router (:3005)                        │
│                                                  │
│  Mounted: workspace, persist-dir, config-dir    │
└──────────────────────────────────────────────────┘
```

Internal ports are stable (4096, 8787, 3005); only the OpenWork Server port is mapped to the host.

### Apple Container Sandbox

Same interface as Docker but uses Apple's `container` CLI on macOS ARM64.

### Sandbox Mount Allowlisting

Additional mounts require explicit allowlisting:

```
~/.config/openwork/sandbox-mount-allowlist.json
```

```json
{
  "version": 1,
  "roots": [
    { "path": "/home/user/projects", "description": "Project files" }
  ]
}
```

---

## Environment Variables Reference

### Set on OpenCode Process

| Variable | Value | Source |
|----------|-------|--------|
| `OPENCODE_CLIENT` | `openwork` or `openwork-orchestrator` | Identifies caller |
| `OPENWORK` | `1` | Marks OpenWork mode |
| `OPENCODE_SERVER_USERNAME` | Random string | HTTP Basic auth |
| `OPENCODE_SERVER_PASSWORD` | Random string | HTTP Basic auth |
| `OPENCODE_CONFIG_DIR` | Workspace path | Config directory |
| `XDG_DATA_HOME` | Platform path | Data directory (dev mode) |
| `XDG_CONFIG_HOME` | Platform path | Config directory (dev mode) |
| `XDG_CACHE_HOME` | Platform path | Cache directory (dev mode) |
| `XDG_STATE_HOME` | Platform path | State directory (dev mode) |
| `OPENCODE_HOT_RELOAD` | `1` or `0` | Enable hot reload |
| `OPENWORK_RUN_ID` | UUID | Telemetry correlation |

### Set on OpenWork Server Process

| Variable | Value | Source |
|----------|-------|--------|
| `OPENWORK_TOKEN` | UUID | Client auth token |
| `OPENWORK_HOST_TOKEN` | UUID | Host auth token |
| `OPENWORK_OPENCODE_USERNAME` | Same as OpenCode username | Proxy auth |
| `OPENWORK_OPENCODE_PASSWORD` | Same as OpenCode password | Proxy auth |
| `OPENCODE_ROUTER_HEALTH_PORT` | Port number | Router discovery |
| `OPENWORK_RUN_ID` | UUID | Telemetry correlation |

### Set on OpenCode Router Process

| Variable | Value | Source |
|----------|-------|--------|
| `OPENCODE_ROUTER_HEALTH_PORT` | Port number | Health endpoint |
| `OPENCODE_SERVER_USERNAME` | Same as OpenCode username | Auth |
| `OPENCODE_SERVER_PASSWORD` | Same as OpenCode password | Auth |
| `OPENCODE_URL` | `http://127.0.0.1:<port>` | OpenCode discovery |
| `OPENWORK_RUN_ID` | UUID | Telemetry correlation |

---

## Key Source Files

### Process Spawning (Tauri)

| File | Lines | Purpose |
|------|-------|---------|
| `apps/desktop/src-tauri/src/engine/spawn.rs` | ~167 | Spawn OpenCode process |
| `apps/desktop/src-tauri/src/engine/manager.rs` | ~69 | Engine state management |
| `apps/desktop/src-tauri/src/commands/engine.rs` | ~756 | Tauri command handlers |
| `apps/desktop/src-tauri/src/openwork_server/spawn.rs` | ~221 | Spawn OpenWork Server |
| `apps/desktop/src-tauri/src/opencode_router/spawn.rs` | ~74 | Spawn OpenCode Router |
| `apps/desktop/src-tauri/src/lib.rs` | — | Plugin setup, command registration |

### Process Spawning (Orchestrator)

| File | Lines | Purpose |
|------|-------|---------|
| `apps/orchestrator/src/cli.ts` | ~8600 | All orchestrator logic |
| `apps/orchestrator/bin/openwork` | — | CLI entry point |

### Server Integration

| File | Lines | Purpose |
|------|-------|---------|
| `apps/server/src/server.ts` | ~4800 | HTTP server, routing, OpenCode proxy |
| `apps/server/src/opencode-db.ts` | ~235 | Database path resolution, session seeding |

### Frontend Communication

| File | Lines | Purpose |
|------|-------|---------|
| `apps/app/src/app/lib/tauri.ts` | ~715 | Tauri IPC invocations |
| `apps/app/src/app/lib/opencode.ts` | ~299 | OpenCode SDK client |
| `apps/app/src/app/lib/openwork-server.ts` | ~1550 | OpenWork Server HTTP client |
