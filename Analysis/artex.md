# ARTEX Architecture

This document describes the structure, runtime behavior, and interfaces among components of **ARTEX Global Edition** (`github.com/Autumn-27/artex`): an LLM multi-agent autonomous penetration-testing system with a Go backend, embedded Next.js frontend, and PostgreSQL dual-graph store.

> Scope: authorized lab / defensive research only. See [README.md](README.md) for legal notices and the [Defense & Detection Guide](docs/defense-en.md).

---

## 1. High-level overview

| Layer | Technology | Role |
|-------|------------|------|
| UI | Next.js 16 + React 19 (static export, `go:embed`) | Dashboard, tasks, assets, findings, chat, system config |
| API / orchestration | Go `net/http` (`server`) | REST `/api/*`, JWT auth, SSE, task/engine lifecycle |
| Agents | `agent` + [`norma`](https://github.com/Autumn-27/norma) SDK | Goals, planner, workers, main-agent (human-in-the-loop) |
| Persistence | PostgreSQL 16 (`db`) | Asset graph + per-task exploration graph + config |
| Side systems | `traffic`, `guard`/`intercept`, `enrich`, `llmpool`, `notify`, … | MITM recording, tool approval, DNS/HTTP enrich, LLM failover, alerts |

A single binary (`cmd/artex`) listens on `-addr` (default `127.0.0.1:8787`) for UI+API and optionally on `-proxy` (default `127.0.0.1:8788`) for the traffic-recording MITM proxy.

```mermaid
flowchart TB
  subgraph Client["Browser"]
    UI["Next.js UI<br/>Dashboard · Tasks · Assets · Findings · Chat · Settings"]
  end

  subgraph Process["artex process"]
    API["server.Server<br/>REST + JWT + SSE"]
    MGR["server.Manager<br/>tasks / stores / proxy"]
    ENG["server.Engine<br/>plannerLoop + N workers"]
    SCH["server.Scheduler<br/>agent triggers"]
    AG["agent<br/>Goals · Planner · Worker · MainAgent"]
    SUB["traffic · guard · intercept · enrich<br/>llmpool · llmrec · notify · report · sidequestion"]
  end

  subgraph Data["Persistence"]
    PG[("PostgreSQL<br/>dual graph + config")]
    SQLite[("SQLite<br/>traffic index")]
    FS["data/ + skills/"]
  end

  subgraph Ext["External"]
    LLM["LLM providers<br/>Anthropic / OpenAI-compatible"]
    Target["Authorized target"]
  end

  UI -->|HTTP / SSE same-origin| API
  API --> MGR
  API --> ENG
  API --> SCH
  MGR --> PG
  ENG --> AG
  AG --> PG
  AG --> LLM
  AG --> SUB
  SUB --> Target
  traffic --> SQLite
  MGR --> FS
```

---

## 2. Repository structure

```
artex-global/
├── cmd/artex/          # Process entry: flags, Manager, HTTP serve, self-update exit codes
├── server/             # HTTP API, Manager, Engine, Scheduler, auth, SSE, webui embed
├── agent/              # LLM roles, ToolSet (dual-graph tools), prompts, compaction
├── db/                 # PostgreSQL access + schema.sql (go:embed, idempotent migrate)
├── config/             # Paths, env overrides, i18n for server messages
├── traffic/            # go-mitmproxy recording proxy + SQLite FTS index
├── guard/              # PreToolUse/PostToolUse hooks → intercept evaluation
├── intercept/          # DB-backed allow/deny/ask rules + human approval wait
├── enrich/             # Async DNS (dnsx) + HTTP probe → asset graph
├── llmpool/            # LLM provider failover / round-robin pool
├── llmrec/             # Optional LLM call recording
├── mcphttp/            # MCP HTTP/SSE client for external tools
├── notify/             # Channels: webhook, email, Telegram, DingTalk, Feishu, WeCom
├── report/             # Finding report assembly
├── sidequestion/       # Lightweight side Q&A (no tools, bounded context)
├── evidence/           # Traffic evidence snapshots for findings
├── selfupdate/         # Staged binary swap + rollback (exit 75 → start.sh relaunch)
├── web/                # Next.js frontend (build → server/webui/dist)
├── skills/             # Agent skill packs (api-recon, playwright-cli, scopesentry, …)
├── detections/         # Defensive Sigma / Suricata / ATT&CK / triage (not runtime)
└── docs/               # Defense guides, evidence docs
```

### Package dependency graph (first-party)

Arrows mean “imports”. `server` is the composition root; `cmd/artex` wires config + Manager + Server.

```mermaid
flowchart LR
  CMD["cmd/artex"] --> SRV["server"]
  CMD --> CFG["config"]
  CMD --> SU["selfupdate"]
  CMD --> AG["agent"]

  SRV --> AG
  SRV --> DB["db"]
  SRV --> CFG
  SRV --> TR["traffic"]
  SRV --> GD["guard"]
  SRV --> IC["intercept"]
  SRV --> EN["enrich"]
  SRV --> LP["llmpool"]
  SRV --> LR["llmrec"]
  SRV --> MCP["mcphttp"]
  SRV --> NT["notify"]
  SRV --> RP["report"]
  SRV --> SQ["sidequestion"]
  SRV --> EV["evidence"]
  SRV --> SU

  AG --> DB
  AG --> GD
  AG --> IC
  AG --> LR
  AG --> EV
  AG --> SQ

  DB --> CFG
  DB --> NT
  DB --> SQ

  GD --> IC
  IC --> DB
  EN --> DB
  TR --> DB
  EV --> DB
  EV --> TR
  RP --> DB
  LR --> DB
  SQ --> LP
```

External SDK: almost all agent execution goes through **`github.com/Autumn-27/norma`** (`llm`, `tool`, `harness`, `agentcore`, `transcript`, `hook`, `permission`).

---

## 3. Component UML (logical)

```mermaid
classDiagram
  direction TB

  class Server {
    +Manager m
    +Engine engine
    +Handler() http.Handler
    +JWT auth / SSE / REST
  }

  class Manager {
    +DB pg
    +AssetStore assets
    +Traffic traffic
    +enrich.Engine enrich
    +Interceptor interceptor
    +map~string,Task~ tasks
    +NewManager(dataDir, proxyAddr)
    +Create / Pause / Resume / Delete Task
  }

  class Task {
    +ID string
    +ExpID int64
    +ExplorationStore Store
    +Guard Guard
    +notify chan
    +lifecycle / LLM state
  }

  class Engine {
    +debounce Duration
    +Broadcaster bc
    +Run(taskID)
    +plannerLoop / workerLoop
    +ControlWork / SteerWork / KillWork
  }

  class Scheduler {
    +Run(ctx)
    +fireIntervals / Findings / Goals / Timeouts / ToolCalls
  }

  class Planner {
    +Plan(ctx, taskID, assets, exploration, goal, triggers)
  }

  class Worker {
    +Execute(ctx, intent)
    +ExecuteWithMessage(...)
  }

  class MainAgent {
    +Chat / steer via hints & intents
  }

  class ToolSet {
    +ExplorationStore ts
    +AssetStore as
    +PlannerTools()
    +WorkerTools()
    +MainAgentTools()
  }

  class ExplorationStore {
    +nodes / edges / anchors / activity
    +claimNextIntent()
  }

  class AssetStore {
    +insert / list / company scope
  }

  class Guard {
    +Hooks() hook.Registry
    +PreToolUse → Interceptor
  }

  class Interceptor {
    +allow / deny / ask
    +Decide(pendingID)
  }

  class Traffic {
    +MITM proxy
    +SQLite index + FTS
  }

  class EnrichEngine {
    +TriggerDNS / TriggerHTTP
  }

  Server --> Manager
  Server --> Engine
  Server --> Scheduler
  Manager --> Task
  Manager --> AssetStore
  Manager --> Traffic
  Manager --> EnrichEngine
  Manager --> Interceptor
  Task --> ExplorationStore
  Task --> Guard
  Engine --> Manager
  Engine --> Planner
  Engine --> Worker
  Server --> MainAgent
  Planner --> ToolSet
  Worker --> ToolSet
  MainAgent --> ToolSet
  ToolSet --> ExplorationStore
  ToolSet --> AssetStore
  Guard --> Interceptor
  Worker ..> Traffic : HTTP via proxy
  ToolSet ..> EnrichEngine : async complete
```

---

## 4. Dual-graph data model

The system separates **what the target is** (global asset graph) from **how far this task has explored** (per-task exploration graph). They join via **anchors**.

### 4.1 Conceptual UML

```mermaid
classDiagram
  direction LR

  class Company {
    +id
    +name
    +nkey
  }
  class CompanyScope {
    +kind domain|ip|cidr|icp|keyword
  }
  class Asset {
    +type root_domain|subdomain|ip|service|app|endpoint
    +company_id
    +dedup keys per type
  }

  class Exploration {
    +goal
    +status open|achieved|failed
    +round_no
  }
  class Node {
    +kind begin|goal|intent|fact|finding|hint|digest
    +state
    +payload JSONB
  }
  class Edge {
    +rel spawns|derived_from|yields|proves|covers
  }
  class Anchor {
    +node_id
    +asset_id
  }
  class Activity {
    +worker
    +tool / summary / detail
    +tokens
  }
  class Finding {
    +severity / status / report
    +asset_ids
  }
  class TaskRow {
    +exploration_id UNIQUE
    +status created|running|paused|done|failed|timeout
  }

  Company "1" --> "*" CompanyScope
  Company "1" --> "*" Asset
  Exploration "1" --> "*" Node
  Exploration "1" --> "*" Edge
  Node --> "*" Edge : src/dst
  Node --> "*" Anchor
  Asset --> "*" Anchor
  Exploration "1" --> "*" Activity
  TaskRow "1" --> "1" Exploration
  TaskRow "1" --> "*" Finding
  Node ..> Finding : finding node ↔ findings row
```

### 4.2 Graph relationship (runtime view)

```mermaid
flowchart LR
  subgraph EG["Exploration graph (per task)"]
    BEGIN[begin]
    GOAL[goal]
    I1[intent]
    FACT[fact]
    FIND[finding]
    HINT[hint]
    BEGIN -->|spawns| GOAL
    GOAL -->|spawns| I1
    I1 -->|yields| FACT
    FACT -->|derived_from| I2[intent]
    I2 -->|proves| FIND
    HINT -.->|steers| I1
  end

  subgraph AG["Asset graph (global)"]
    RD[root_domain] --> SD[subdomain] --> SV[service] --> EP[endpoint]
  end

  I1 -. anchor .-> SD
  FACT -. anchor .-> SV
  FIND -. anchor .-> EP
```

**Role split**

| Role | Writes | Does not |
|------|--------|----------|
| **Planner** | intents, prove/goal_met, kill/steer work | Execute tools against the target |
| **Worker** | facts, findings, assets (for one claimed intent) | Create the next attack-chain intent (planner’s job) |
| **MainAgent** | hints, intents, goals, constraints (human steer) | Own the autonomous planning loop |

---

## 5. Runtime behavior

### 5.1 Process bootstrap

```mermaid
sequenceDiagram
  autonumber
  participant Start as start.sh
  participant Main as cmd/artex
  participant SU as selfupdate
  participant Mgr as Manager
  participant Srv as Server
  participant HTTP as net/http

  Start->>Main: exec artex
  Main->>SU: Bootstrap()
  alt Restart needed
    SU-->>Main: Restart
    Main-->>Start: exit 75 → relaunch
  end
  Main->>Mgr: NewManager(dataDir, proxyAddr)
  Mgr->>Mgr: open PG, migrate schema, start traffic/enrich
  Main->>Srv: New(ctx, mgr, skillDir, …)
  Srv->>Srv: wire agents, LLM pool, scheduler, SSE
  Main->>HTTP: ListenAndServe(addr)
  Note over HTTP: UI embed + /api/* + optional MITM :8788
```

### 5.2 Closed exploration loop (engine)

Event-driven: graph change → debounced planner wake → intents on frontier → workers claim → write graphs → wake again until `prove_goal` / terminal task state.

```mermaid
sequenceDiagram
  autonumber
  participant Ev as graph-change debounce
  participant P as Planner
  participant Fr as frontier intents
  participant W as Worker ×N
  participant G as Guard/Intercept
  participant Px as traffic MITM
  participant DB as dual graph + activity
  participant SSE as Broadcaster

  Ev-->>P: wake (+ TriggerEvents)
  P->>DB: graph_overview / coverage / scope
  P->>Fr: add_intent 0..N (ordered via shared todolist)
  W->>Fr: claimNext (one intent)
  W->>DB: load intent + anchored assets
  loop tool turns
    W->>G: PreToolUse
    alt ask
      G-->>W: wait human decide
    else allow
      W->>Px: Bash / HTTP / WebFetch
      Px-->>W: response (recorded)
    end
    W->>DB: record_fact / insert_assets / report_finding + activity
    DB-->>SSE: activity push
  end
  DB-->>Ev: graph changed
  Ev-->>P: next round
```

### 5.3 Shared planner todolist (multi-step attack chain)

Each planner wake is a **fresh LLM session**. A per-task **TodoStore** (norma) persists the ordered chain so rounds dispatch **one dependency-ready step at a time** instead of unfolding the whole chain in parallel.

```mermaid
stateDiagram-v2
  [*] --> RoundWake
  RoundWake --> ReadGraph: load overview + triggers
  ReadGraph --> UpdateTodos: mark prereq-satisfied steps done
  UpdateTodos --> DispatchNext: add_intent for next ready step
  DispatchNext --> Idle: 0 intents if no new direction
  DispatchNext --> WaitWorkers: N intents assigned
  WaitWorkers --> RoundWake: worker writes / finding / hint
  Idle --> RoundWake: heartbeat / human hint / new goal
```

### 5.4 Worker cross-process information exchange

Workers do not share memory. They search **activity traces** of other workers in the same task:

1. `search_all_worker_traces(q)` — keyword search (excludes own steps)
2. `list_worker_traces` / `get_worker_trace(intent_id, step_ids)` — pull full step bodies

### 5.5 Human-in-the-loop paths

| Path | Component | Effect |
|------|-----------|--------|
| Chat | `MainAgent` + `/api/tasks/{id}/…` chat | hints → planner; high-priority intents → frontier; goals/constraints |
| Intent control | `Engine.ControlWork` | pause / resume / cancel one intent |
| Steer | `steer_work` | inject mid-run message before next tool call (no kill) |
| Intercept ask | `intercept` + UI approvals | block tool until allow/deny/timeout policy |
| Side question | `sidequestion` | read-only Q&A over captured context (no tools) |

---

## 6. Interfaces among components

### 6.1 HTTP API surface (server ↔ web)

`Server.Handler()` registers Go 1.22+ method patterns. Major groups:

| Prefix | Purpose |
|--------|---------|
| `/api/auth/*` | setup token, login, change password (JWT) |
| `/api/health`, `/api/stats`, `/api/logs*` | health, metrics, live logs |
| `/api/tasks*`, `/api/task-*` | CRUD, control, coverage, scope, goals, constraints, archives, templates |
| `/api/exploration/*` | frontier, findings, lineage, nodes, activity |
| `/api/assets`, `/api/companies` | global asset/company graph |
| `/api/llm*`, `/api/agents*`, `/api/mcp*`, `/api/skills*`, `/api/tools*` | LLM profiles, agent prompts, MCP, skills |
| `/api/intercept*`, `/api/workspace*`, `/api/traffic*` | approvals, workspace FS, traffic browser |
| `/api/notify*`, `/api/update*`, SSE streams | notifications, self-update, activity |

Frontend client: `web/src/lib/api.ts` (typed fetch; same-origin in production, proxied in `next dev`).

### 6.2 Agent ↔ DB (ToolSet)

`agent.ToolSet` is the primary **domain interface** from LLM tools to PostgreSQL:

```mermaid
flowchart TB
  subgraph Roles["Tool subsets"]
    PT[PlannerTools]
    WT[WorkerTools]
    MT[MainAgentTools]
  end

  subgraph Domain["Dual-graph tools"]
    R1[graph_overview / node_detail / list_*]
    W1[add_intent / prove_goal / goal_met]
    W2[record_fact / report_finding / insert_assets]
    X1[search_all_worker_traces / get_worker_trace]
    S1[add_hint / set_goals / set_constraints / add_task_scope]
    C1[kill_work / steer_work]
  end

  PT --> R1
  PT --> W1
  PT --> X1
  PT --> C1
  WT --> W2
  WT --> X1
  WT --> R1
  MT --> R1
  MT --> S1
  MT --> W1
  MT --> W2
```

Callbacks on `ToolSet` close the loop back into `Engine` / UI:

- `notify` / `notifyFinding` / `notifyHint` / `notifyGoal` → wake planner with typed triggers  
- `resumeTask` → revive terminal/paused task after human adds goals  
- `killWork` / `steerWork` → engine per-intent control  
- `enrich` → `enrich.Engine` async DNS/HTTP completion  

### 6.3 Guard ↔ Intercept ↔ Human

```mermaid
sequenceDiagram
  participant W as Worker / Agent
  participant G as guard.Guard
  participant I as intercept.Interceptor
  participant DB as intercept_pending
  participant UI as Web approvals

  W->>G: PreToolUse(tool, input)
  G->>I: evaluate rules (priority order)
  alt allow
    I-->>G: proceed
    G-->>W: allow
  else deny
    I-->>W: block + message to model
  else ask
    I->>DB: pending row + activity
    I->>UI: SSE / list pending
    UI->>I: decide allow|deny
    I-->>W: resume or block
  end
```

Rules live in `intercept_rules` / `task_intercept_rules` / `asset_intercept_rules`. Built-in destructive/exfil patterns are seeded as ordinary DB rules (user-disableable).

### 6.4 Traffic recording

- Agents’ Bash / WebFetch are pointed at the MITM (`Manager` / `SetProxy`).
- `traffic.Traffic` (go-mitmproxy) writes exchanges to a file tree + SQLite (+ optional FTS5).
- Findings can bind evidence via `finding_traffic_bindings` / `traffic_evidence_snapshots` (`evidence` package).

### 6.5 LLM provider stack

```mermaid
flowchart LR
  Agent["Planner / Worker / MainAgent"] --> Prov["llm.Provider"]
  Prov --> Rec["llmrec wrapper<br/>optional"]
  Rec --> Pool["llmpool.Pool<br/>failover chain"]
  Pool --> P1["Profile A"]
  Pool --> P2["Profile B"]
  Pool --> P3["Profile C"]
```

Profiles are rows in `llm_profiles`; health in `llm_profile_health`. Task-level chains override the global active profile (`task_llm_profiles`).

### 6.6 Scheduler ↔ agents

`server.Scheduler` polls DB `agent_triggers` and fires:

- interval timers  
- new findings / goal met  
- task timeout  
- task create  
- tool-call patterns  

Used for reporter agents, Auto orchestration bindings, and custom trigger→agent wiring (`server/assembly.go`, `orchestration.go`).

---

## 7. Frontend architecture

```
web/src/
├── app/
│   ├── (auth)/login|setup
│   ├── (main)/
│   │   ├── dashboard
│   │   ├── chat
│   │   ├── function/{tasks,assets,findings,traffic,workspace,commands,llm-records,sync}
│   │   └── system/{settings,llm,agents,mcp,skills,tools,intercept,notify,logs}
│   └── (external)/…
├── components/          # domain UI + shadcn/ui
├── lib/api.ts           # backend client
├── lib/types.ts         # DTOs aligned with Go handlers
├── i18n/ + messages/    # en · ko · zh · es · ja
├── hooks/ · stores/
└── navigation/
```

**Build path:** `npm run build:static` → `web/out` copied to `server/webui/dist` → compiled with `-tags embedui`. Without the tag (dev), API-only backend; UI runs on `:5173` via `dev.sh`.

```mermaid
flowchart LR
  Pages["App Router pages"] --> API["lib/api.ts"]
  API -->|fetch /api| Go["server.Handler"]
  Pages --> SSE["EventSource activity/logs"]
  SSE --> Go
  Pages --> i18n["next-intl"]
```

---

## 8. Deployment view

```mermaid
flowchart TB
  subgraph Host["Host / Docker Compose"]
    C1["artex container<br/>:8787 UI+API<br/>:8788 MITM"]
    C2["postgres:16"]
    Vol1[("volume pgdata")]
    Vol2[("./data · ./skills")]
    C1 --> C2
    C2 --> Vol1
    C1 --> Vol2
  end
  Browser --> C1
  C1 --> LLM["LLM API"]
  C1 --> Target["Authorized targets"]
```

Supervisor scripts (`start.sh` / `start.bat`) relaunch on self-update exit code `75`. Reverse proxies must **disable buffering** for SSE.

---

## 9. Supporting & defensive packages

| Package / dir | Interface role |
|---------------|----------------|
| `notify` | Event → channel adapters (webhook/email/chat apps); used from DB/server on findings etc. |
| `report` | Assemble Markdown finding reports from DB |
| `mcphttp` | Connect external MCP tool servers into agent tool lists |
| `selfupdate` | GitHub release stage/swap/rollback |
| `skills/` | Filesystem skill packs mounted into agent work context |
| `detections/` | **Not loaded at runtime** — Sigma/Suricata/ATT&CK/indicators/triage for defenders studying ARTEX behavior |

---

## 10. Key behavioral invariants

1. **Planner is the only autonomous intent producer**; workers execute exactly one claimed intent then stop.  
2. **Asset graph is globally shared**; exploration graph is **per task**, linked by anchors.  
3. **Engine loops always run** but idle without an LLM provider; LLM can be configured at runtime from the UI.  
4. **Every tool call** can pass `guard` → `intercept` (allow / deny / ask).  
5. **Graph writes emit activity** → SSE to UI and debounced planner wakes.  
6. **Output language** for user-facing agent text is forced via `agent/prompt.go` (`langDirective`); internal reasoning prompts stay in the original language for performance.  
7. **Default bind is loopback**; first-run requires setup token (or `ARTEX_ADMIN_PASSWORD`).

---

## 11. Where to read next

| Concern | Primary code |
|---------|----------------|
| Entry / lifecycle | `cmd/artex/main.go`, `server/manager.go`, `server/engine.go` |
| HTTP routes | `server/server.go` (`Handler`) |
| Agent roles | `agent/planner.go`, `agent/worker.go`, `agent/mainagent.go`, `agent/goals.go` |
| Domain tools | `agent/tools.go`, `agent/tools_insert.go` |
| Schema | `db/schema.sql` |
| Product narrative | [README.md § System architecture](README.md#system-architecture) |
| Defender view | [docs/defense-en.md](docs/defense-en.md) |

---

*Generated from the repository source layout and package contracts. Diagrams use Mermaid (UML-style class, sequence, state, and deployment views) so they render in GitHub and most Markdown previews.*
