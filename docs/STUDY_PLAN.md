# NemoClaw Zero-to-Hero Study Plan

A systematic learning plan that takes you from zero knowledge of web/full-stack development to fully understanding every piece of NemoClaw. Each module pairs theory with hands-on implementation using this repository as the primary learning material.

**No time constraints.** Work through at your own pace. Each module builds on the previous one.

---

## Module 0: The Four Technologies — How They Relate

**Goal**: Before studying any code, understand what NemoClaw, OpenShell, Docker,
and OpenClaw are, why they exist as separate technologies, and how they depend
on each other.

### The Core Relationship

```
OpenClaw  = The AI Agent          (does AI work: conversations, coding, tool calling)
Docker    = The Container Engine  (provides isolated environments for everything)
OpenShell = The Security Layer    (adds security ON TOP of Docker containers)
NemoClaw  = The Glue              (connects all three: automates setup, configures everything)
```

### The Analogy

- **OpenClaw** is a very capable robot that can clean your house, cook meals, and manage
  your schedule. But it has no sense of boundaries — it will go anywhere and do anything.
- **Docker** is the building material (walls, doors, windows). You CAN build a room from
  these materials, but the rooms have no locks and no alarm system.
- **OpenShell** is a security company that uses the building materials (Docker) to build
  high-security rooms with locks on every door, cameras, and an alarm system.
- **NemoClaw** is the contractor who draws the blueprints, tells the security company
  what to build, installs the robot in the finished room, and configures which doors
  should be locked or unlocked.

### The Layer Cake

```
+---------------------------------------------------------------+
|  Layer 4:  NemoClaw       (orchestration — "how to set up")   |
+---------------------------------------------------------------+
|  Layer 3:  OpenClaw       (AI agent — "what to do")           |
+---------------------------------------------------------------+
|  Layer 2:  OpenShell      (security — "what's allowed")       |
+---------------------------------------------------------------+
|  Layer 1:  Docker         (containers — "where to run")       |
+---------------------------------------------------------------+
|  Layer 0:  Linux Kernel   (namespaces, cgroups, Landlock)     |
+---------------------------------------------------------------+
```

### Dependency Direction

```
NemoClaw  --(depends on)--> Docker     (builds images, runs NIM containers)
NemoClaw  --(depends on)--> OpenShell  (calls openshell CLI for sandbox/policy)
NemoClaw  --(depends on)--> OpenClaw   (registers as plugin, installs in sandbox)
OpenShell --(depends on)--> Docker     (runs gateway + sandboxes as containers)
OpenClaw  --(knows nothing about)--> Docker, OpenShell, or NemoClaw
Docker    --(knows nothing about)--> any of the above
```

NemoClaw is the ONLY technology that knows about all three others.

### Docker's Multiple Roles

Docker serves four distinct purposes (a common source of confusion):

| Role | What Docker Does | Who Triggers It |
|---|---|---|
| **Gateway host** | Runs OpenShell's k3s gateway as a container | OpenShell (via `openshell gateway start`) |
| **Sandbox host** | Runs the sandbox container where OpenClaw lives | OpenShell (via `openshell sandbox create`) |
| **NIM host** | Runs local NIM inference containers with GPU | NemoClaw directly (via `docker run`) |
| **Image builder** | Builds the sandbox image from NemoClaw's Dockerfile | OpenShell internally (during sandbox create) |

### What Each Technology Owns

| Responsibility | Owned By |
|---|---|
| Running AI agents, conversations, tool use | **OpenClaw** |
| Hosting all containers (gateway, sandbox, NIM) | **Docker** |
| Building container images from Dockerfiles | **Docker** |
| GPU passthrough for NIM containers | **Docker** |
| Adding security layers on top of Docker containers | **OpenShell** |
| Network policy enforcement at runtime | **OpenShell** |
| Credential injection at runtime | **OpenShell** |
| Activity monitoring (TUI) | **OpenShell** |
| Orchestrating setup of all of the above | **NemoClaw** |
| Building the sandbox Docker image | **NemoClaw** |
| Configuring which model and provider to use | **NemoClaw** |
| Managing NIM containers (pull, start, stop) | **NemoClaw** |
| Applying network policy presets | **NemoClaw** |
| Migration from host to sandbox | **NemoClaw** |

### Where They Live

```
In this repository:            NemoClaw (all of it)
Installed via npm:             OpenClaw (openclaw@2026.3.11)
Installed from GitHub:         OpenShell (github.com/NVIDIA/OpenShell)
Installed from docker.com:     Docker (or via apt/brew)
```

### Read More

For the full deep-dive with communication diagrams, runtime traces, and FAQ,
see [How NemoClaw, OpenShell, Docker, and OpenClaw Work Together](HOW_THE_THREE_PROJECTS_WORK_TOGETHER.md).

### Self-Check Questions (Module 0)

- [ ] Can OpenClaw run without Docker? (Yes — OpenClaw is just a Node.js app)
- [ ] Can OpenShell run without Docker? (No — OpenShell uses Docker for everything)
- [ ] Can NemoClaw run without Docker? (No — hard requirement, checked at preflight)
- [ ] How many Docker containers run after setup? (2 or 3: gateway + sandbox + optional NIM)
- [ ] Which technology enforces network policies at runtime? (OpenShell)
- [ ] Which technology hosts the containers where policies are enforced? (Docker)
- [ ] Which technology configures those policies at setup time? (NemoClaw)
- [ ] Which technology actually runs the AI tasks? (OpenClaw)
- [ ] Why does NemoClaw talk to Docker both directly AND through OpenShell? (Sandboxes go through OpenShell for security; NIM containers go directly because they're trusted services)

---

## Learning Path Overview

```
Module 0: The Three Projects (OpenClaw, OpenShell, NemoClaw)
    |
    v
Module 1: Foundations (JavaScript, Node.js)
    |
    v
Module 2: The Module System (CommonJS, ESM, npm)
    |
    v
Module 3: TypeScript Essentials
    |
    v
Module 4: CLI Architecture (Commander.js, Process I/O)
    |
    v
Module 5: The Host CLI Deep Dive (bin/)
    |
    v
Module 6: The Plugin System (OpenClaw Plugin API)
    |
    v
Module 7: Blueprint Orchestration (Python + Subprocess)
    |
    v
Module 8: Container Architecture (Docker, Dockerfile)
    |
    v
Module 9: Security Architecture (Landlock, seccomp, Network Policies)
    |
    v
Module 10: Inference Routing (Providers, Models, Proxy)
    |
    v
Module 11: Migration & State Management
    |
    v
Module 12: Testing Strategies
    |
    v
Module 13: Integration: Putting It All Together
    |
    v
Module 14: Contributing & Extending
```

---

## Module 1: Foundations — JavaScript and Node.js

**Goal**: Understand the language and runtime that powers NemoClaw's host CLI.

### Theory

**JavaScript** is a dynamically-typed language originally designed for browsers. **Node.js** runs JavaScript outside the browser, on servers and in CLI tools.

Key concepts for NemoClaw:

| Concept | C/C++/Java Equivalent | NemoClaw Usage |
|---|---|---|
| `const/let` | `final`/variable declaration | Everywhere |
| Arrow functions `() => {}` | Lambda / anonymous functions | Callbacks, handlers |
| `async/await` | Threads + futures | All I/O operations |
| Destructuring `{ a, b } = obj` | No equivalent | Extracting config values |
| Template literals `` `hello ${name}` `` | `String.format()` | Log messages, commands |
| `try/catch` | `try/catch` | Error handling |
| Spread `{ ...obj, key: val }` | Object.clone() + modify | Config merging |

### Hands-On: Read and Annotate

**File to study**: `bin/lib/runner.js`

This is the simplest module in the project. Read it line by line:

1. Open `bin/lib/runner.js`
2. Identify every `require()` call — these are like `#include` in C
3. Find the `run()` function — it executes shell commands
4. Find the `runCapture()` function — it captures shell output
5. Note how `execSync` is used — this is Node.js's equivalent of C's `system()`

**Exercise**: In your terminal, run:
```bash
node -e "const { execSync } = require('child_process'); console.log(execSync('echo hello', { encoding: 'utf-8' }))"
```

This demonstrates the core pattern used throughout NemoClaw.

### Self-Check Questions

- [ ] What is the difference between `const` and `let`?
- [ ] What does `async` mean before a function?
- [ ] What does `await` do?
- [ ] What is `execSync` and why is it "sync"?
- [ ] What is `require()` and what does it return?

---

## Module 2: The Module System — CommonJS, ESM, and npm

**Goal**: Understand how NemoClaw's code is organized into modules and how dependencies work.

### Theory

JavaScript has **two module systems** (this is a major source of confusion):

**CommonJS (CJS)** — Used by `bin/` files:
```javascript
// Exporting
module.exports = { myFunction };

// Importing
const { myFunction } = require('./myModule');
```

**ECMAScript Modules (ESM)** — Used by `nemoclaw/src/` files:
```typescript
// Exporting
export function myFunction() { ... }

// Importing
import { myFunction } from './myModule.js';
```

**npm** manages dependencies:
- `package.json` = declares what the project needs
- `node_modules/` = where dependencies are downloaded
- `package-lock.json` = exact versions of every dependency (like a lockfile)

### Hands-On: Trace the Dependency Graph

**File to study**: `package.json` (root) and `nemoclaw/package.json`

1. Open root `package.json`. It has one dependency: `openclaw@2026.3.11`
2. Open `nemoclaw/package.json`. Note the split:
   - `dependencies`: needed at runtime (commander, yaml, json5, tar)
   - `devDependencies`: needed only during development (typescript, eslint, prettier)
3. The `scripts` section defines commands you run with `npm run <name>`:
   - `"build": "tsc"` → `npm run build` runs the TypeScript compiler
   - `"test": "node --test test/*.test.js"` → `npm test` runs all tests

**Exercise**: Run these commands and observe what happens:
```bash
# See all available scripts:
cat package.json | node -e "process.stdin.on('data',d=>console.log(Object.keys(JSON.parse(d).scripts||{}).join('\n')))"

# See where a dependency is installed:
ls node_modules/commander 2>/dev/null || echo "Not in root"
ls nemoclaw/node_modules/commander 2>/dev/null || echo "Not in plugin"
```

### Hands-On: Trace an Import Chain

Start from `bin/nemoclaw.js:10`:
```javascript
const { ROOT, SCRIPTS, run, runCapture } = require("./lib/runner");
```

1. This loads `bin/lib/runner.js`
2. `runner.js` likely loads `child_process` (a Node.js built-in)
3. Trace all `require()` calls in `nemoclaw.js` — draw the dependency tree

```
bin/nemoclaw.js
    +-- ./lib/runner        (run, runCapture, ROOT, SCRIPTS)
    +-- ./lib/credentials   (ensureApiKey, getCredential, etc.)
    +-- ./lib/registry      (sandbox CRUD)
    +-- ./lib/nim           (NIM container management)
    +-- ./lib/policies      (policy preset management)
    +-- ./lib/onboard       (7-step wizard, loaded lazily)
```

### Self-Check Questions

- [ ] Why does `nemoclaw/src/` use `.js` extensions in imports even though the source files are `.ts`?
- [ ] What is the difference between `dependencies` and `devDependencies`?
- [ ] Why does NemoClaw have two `package.json` files?
- [ ] What does `npm run build` actually execute?

---

## Module 3: TypeScript Essentials

**Goal**: Understand the typed superset of JavaScript that powers the NemoClaw plugin.

### Theory

TypeScript = JavaScript + static types. Key additions:

```typescript
// Type annotations (like Java's type declarations)
function greet(name: string): string {
    return `Hello, ${name}`;
}

// Interfaces (like Java interfaces, but structural)
interface Config {
    name: string;
    port: number;
    debug?: boolean;  // Optional property
}

// Generics (like Java generics)
function first<T>(arr: T[]): T | undefined {
    return arr[0];
}

// Type assertions (like casts)
const value = someUnknown as string;

// Union types (no Java equivalent)
type Result = "success" | "failure";

// Null safety
const x: string | null = null;
```

### Hands-On: Study the Type Definitions

**File to study**: `nemoclaw/src/index.ts:22-133`

This file defines all the types NemoClaw uses. For each interface:

1. **OpenClawConfig** (line 24) — Generic config object. `[key: string]: unknown` means "any string key, any value" (like Java's `Map<String, Object>`)

2. **PluginLogger** (line 29) — Logger with info/warn/error/debug methods. This is *dependency injection* — the host provides the logger.

3. **PluginCommandContext** (line 37) — Data passed to slash command handlers. Note optional properties (`senderId?`).

4. **OpenClawPluginApi** (line 120) — The main API surface. Study each method:
   - `registerCommand` — Register a chat command
   - `registerCli` — Register CLI subcommands
   - `registerProvider` — Register an inference provider
   - `registerService` — Register a background service

**Exercise**: Compare these TypeScript interfaces to equivalent Java interfaces. Write the Java version of `PluginLogger`.

### Hands-On: Trace Type Flow

Follow the types from `register()` through `cliLaunch()`:

```
register(api: OpenClawPluginApi)     [index.ts:179]
    |
    | getPluginConfig(api) returns NemoClawConfig
    |
    v
cliLaunch({ force, profile, logger, pluginConfig })  [launch.ts:19]
    |
    | resolveBlueprint(pluginConfig) returns Promise<ResolvedBlueprint>
    |
    v
ResolvedBlueprint { version, localPath, manifest, cached }  [resolve.ts:17]
```

### Self-Check Questions

- [ ] What is the difference between `interface` and `type` in TypeScript?
- [ ] What does `?` mean after a property name (e.g., `senderId?: string`)?
- [ ] What does `Promise<void>` mean?
- [ ] Why are there `.js` extensions in TypeScript import statements?
- [ ] What is `tsconfig.json` and what does `strict: true` do?

---

## Module 4: CLI Architecture — Commander.js and Process I/O

**Goal**: Understand how NemoClaw builds its command-line interface.

### Theory

**Commander.js** is a library for building CLI tools. It handles argument parsing, help text generation, and subcommand routing.

```typescript
// Basic Commander pattern:
program
  .command("status")                              // Subcommand name
  .description("Show status")                     // Help text
  .option("--json", "Output as JSON", false)      // Flag with default
  .option("-n, --lines <count>", "Line count", "50")  // Named option
  .action(async (opts) => {                       // Handler function
    // opts.json = true/false
    // opts.lines = "50" or user value
  });
```

**Process I/O** — NemoClaw communicates between layers via:
- `child_process.spawn()` — Start a subprocess (used for Python runner)
- `child_process.execSync()` — Run a command and wait for result
- `stdout` / `stderr` — Standard output/error streams
- Exit codes — 0 = success, non-zero = failure

### Hands-On: Study the CLI Wiring

**File to study**: `nemoclaw/src/cli.ts`

This file wires every `openclaw nemoclaw <cmd>` subcommand:

1. Line 24: Creates the `nemoclaw` parent command
2. Lines 27-33: Wires `status` with `--json` option
3. Lines 36-50: Wires `migrate` with `--dry-run`, `--profile`, `--skip-backup`
4. Lines 53-65: Wires `launch` with `--force`, `--profile`

**Exercise**: Map each `.command()` call to its handler function and source file:

| Command | Handler | File |
|---|---|---|
| `status` | `cliStatus()` | `commands/status.ts` |
| `migrate` | `cliMigrate()` | `commands/migrate.ts` |
| `launch` | `cliLaunch()` | `commands/launch.ts` |
| `connect` | `cliConnect()` | `commands/connect.ts` |
| `logs` | `cliLogs()` | `commands/logs.ts` |
| `eject` | `cliEject()` | `commands/eject.ts` |
| `onboard` | `cliOnboard()` | `commands/onboard.ts` |

### Hands-On: Study the Dual Dispatcher

**File to study**: `bin/nemoclaw.js:306-366`

The host CLI has a *manual* dispatcher (no Commander.js):

```javascript
const [cmd, ...args] = process.argv.slice(2);
// process.argv = ["node", "nemoclaw.js", "my-assistant", "connect"]
// cmd = "my-assistant"
// args = ["connect"]
```

1. First checks if `cmd` is in `GLOBAL_COMMANDS` (onboard, list, deploy, etc.)
2. If not, checks if `cmd` is a registered sandbox name
3. If sandbox found, reads `args[0]` as the action (connect, status, logs, etc.)
4. If neither, shows error with suggestions

**Exercise**: What happens when you run `nemoclaw my-assistant logs --follow`?
Trace through the dispatcher to find exactly which function gets called and with what arguments.

### Self-Check Questions

- [ ] What is `process.argv` and what does `slice(2)` do?
- [ ] What is the difference between Commander.js and manual argv parsing?
- [ ] Why does the host CLI use manual parsing while the plugin uses Commander.js?
- [ ] What does `.option("--json", "description", false)` mean?

---

## Module 5: The Host CLI Deep Dive

**Goal**: Understand every module in `bin/lib/` and how they work together.

### Study Order

Study these files in order (simplest to most complex):

#### 5.1: `bin/lib/runner.js` — Shell Execution

The simplest module. Provides `run()` and `runCapture()`.

**Key pattern**: Wraps `execSync` with error handling and output formatting.

#### 5.2: `bin/lib/registry.js` — Sandbox Registry

CRUD operations on `~/.nemoclaw/sandboxes.json`.

**Key pattern**: Load JSON → modify → save JSON. Simple file-based "database."

**Exercise**: Read the file and answer:
- How does it handle the case when the file doesn't exist yet?
- What data is stored for each sandbox?
- How is the "default" sandbox tracked?

#### 5.3: `bin/lib/credentials.js` — Credential Management

Secure storage of API keys.

**Key pattern**: File stored with mode `0o600` (owner read/write only). Interactive prompt for missing keys.

**Exercise**: Trace what happens when you run a command that needs `NVIDIA_API_KEY`:
1. `ensureApiKey()` checks `process.env.NVIDIA_API_KEY`
2. If missing, checks `~/.nemoclaw/credentials.json`
3. If still missing, prompts the user interactively
4. Saves to credentials file for next time

#### 5.4: `bin/lib/preflight.js` — System Checks

Validates Docker, cgroups, and OpenShell before proceeding.

**Key concept**: "Fail fast" — check prerequisites before doing expensive operations.

#### 5.5: `bin/lib/nim.js` — NIM Container Management

GPU detection, Docker image management, container lifecycle.

**Exercise**: Read the `detectGpu()` function and understand how it detects:
- NVIDIA GPUs (via `nvidia-smi`)
- Apple GPUs (via `system_profiler`)
- No GPU (falls back to cloud inference)

#### 5.6: `bin/lib/policies.js` — Policy Presets

Reads YAML preset files, applies them via `openshell policy set`.

**Exercise**: Trace the flow of `applyPreset(sandboxName, "slack")`:
1. Reads `nemoclaw-blueprint/policies/presets/slack.yaml`
2. Extracts `network_policies` section
3. Calls `openshell policy set` with the merged policy

#### 5.7: `bin/lib/onboard.js` — The 7-Step Wizard

The most complex module. Orchestrates the entire setup flow.

**Study strategy**: Read each `step()` function in order:
1. `preflight()` — Uses preflight.js, nim.js
2. `startGateway()` — Creates OpenShell gateway
3. `createSandbox()` — Builds Docker image, creates sandbox
4. `setupNim()` — Configures NIM or cloud inference
5. `setupInference()` — Creates provider, sets inference route
6. `setupOpenclaw()` — Initializes OpenClaw in sandbox
7. `setupPolicies()` — Auto-detects tokens, applies presets

### Self-Check Questions

- [ ] Draw the dependency graph of all `bin/lib/` modules
- [ ] What is the complete flow when `nemoclaw onboard` runs?
- [ ] How does credential storage differ from environment variable usage?
- [ ] What happens if the user has no GPU?

---

## Module 6: The Plugin System — OpenClaw Plugin API

**Goal**: Understand how NemoClaw integrates with OpenClaw as a plugin.

### Theory

OpenClaw has a **plugin system** where extensions register themselves:

```typescript
// Plugin entry point — OpenClaw calls this function
export default function register(api: OpenClawPluginApi): void {
    api.registerCommand({ ... });    // Chat commands
    api.registerCli((ctx) => { ... }); // Terminal commands
    api.registerProvider({ ... });    // Inference providers
}
```

The plugin API (`api`) provides:
- **Logger**: `api.logger.info("message")`
- **Config**: `api.config` (OpenClaw global config), `api.pluginConfig` (plugin-specific)
- **Registration**: Register commands, CLI, providers, services
- **Events**: `api.on("hookName", handler)`

### Hands-On: Study the Registration

**File to study**: `nemoclaw/src/index.ts:179-259`

The `register()` function does three things:

1. **Lines 181-186**: Registers `/nemoclaw` slash command
   - When a user types `/nemoclaw` in OpenClaw chat, `handleSlashCommand()` is called

2. **Lines 189-194**: Registers CLI subcommands
   - When a user runs `openclaw nemoclaw <cmd>`, Commander.js routes to handlers

3. **Lines 203-245**: Registers `nvidia-nim` inference provider
   - Declares supported models with metadata (context window, max output)
   - Declares authentication method (Bearer token via NVIDIA_API_KEY)

### Hands-On: The Plugin Manifest

**File to study**: `nemoclaw/openclaw.plugin.json`

```json
{
  "id": "nemoclaw",
  "name": "NemoClaw",
  "version": "0.1.0",
  "description": "Migrate and run OpenClaw inside OpenShell",
  "configSchema": { ... }
}
```

- `id` and `name`: Identify the plugin
- `configSchema`: JSON Schema that validates plugin configuration
- OpenClaw reads this file to know about the plugin

### Self-Check Questions

- [ ] What are the three things the `register()` function does?
- [ ] How does OpenClaw know to call `register()`?
- [ ] What is the role of `openclaw.plugin.json`?
- [ ] How are plugin config values validated?

---

## Module 7: Blueprint Orchestration — Python + Subprocess

**Goal**: Understand the Python layer and how TypeScript communicates with it.

### Theory

The **blueprint** is a Python-based orchestrator that:
1. Reads a YAML configuration (`blueprint.yaml`)
2. Translates high-level actions (plan, apply, status, rollback) into `openshell` CLI commands
3. Reports progress via stdout protocol

**Communication protocol** (TypeScript ↔ Python):
```
TypeScript spawns:  python3 runner.py plan --profile default
Python sends:       PROGRESS:10:Validating blueprint
                    PROGRESS:20:Checking prerequisites
                    RUN_ID:nc-20260315-143022-a1b2c3d4
                    { ... JSON plan ... }
                    PROGRESS:100:Plan complete
Python exits:       code 0 (success) or non-zero (failure)
TypeScript parses:  stdout lines for PROGRESS: and RUN_ID:
```

### Hands-On: Study the Blueprint Runner

**File to study**: `nemoclaw-blueprint/orchestrator/runner.py`

Read in this order:

1. **`main()`** (line 312): Entry point. Uses `argparse` to parse CLI args, loads blueprint, dispatches to action functions.

2. **`load_blueprint()`** (line 45): Reads `blueprint.yaml` using PyYAML.

3. **`action_plan()`** (line 80): Validates profile, checks prerequisites, generates JSON plan.

4. **`action_apply()`** (line 138): The core function. Executes `openshell` commands:
   - Step 1: `openshell sandbox create --from <image> --name <name>`
   - Step 2: `openshell provider create --name <provider> --type <type>`
   - Step 3: `openshell inference set --provider <provider> --model <model>`
   - Step 4: Save state to `~/.nemoclaw/state/runs/<id>/plan.json`

5. **`action_rollback()`** (line 273): Stops and removes sandbox, marks run as rolled back.

### Hands-On: Study the TypeScript Side

**File to study**: `nemoclaw/src/blueprint/exec.ts`

This file spawns the Python runner and parses its output:

1. Line 58: `spawn("python3", args, { ... })` — Starts the subprocess
2. Lines 68-76: Reads stdout/stderr
3. Lines 78-87: When subprocess exits, parses `RUN_ID:` from output
4. Lines 90-96: Handles errors (python3 not found, etc.)

**Exercise**: Run the blueprint runner directly to see its output:
```bash
cd nemoclaw-blueprint
NEMOCLAW_BLUEPRINT_PATH=. python3 orchestrator/runner.py plan --profile default --dry-run
```

### Self-Check Questions

- [ ] Why is the blueprint written in Python and not TypeScript?
- [ ] What is the stdout protocol between TypeScript and Python?
- [ ] How does `exec.ts` know when the Python process has finished?
- [ ] What happens if `python3` is not installed?

---

## Module 8: Container Architecture — Docker and the Dockerfile

**Goal**: Understand how the sandbox container is built and what's inside it.

### Theory

**Docker** packages applications into containers:
- **Image**: A template (like a class in OOP)
- **Container**: A running instance of an image (like an object)
- **Dockerfile**: Build instructions for an image (like a Makefile)
- **Layers**: Each instruction creates a layer (cached for speed)

### Hands-On: Study the Dockerfile

**File to study**: `Dockerfile`

Read each instruction:

```dockerfile
FROM node:22-slim                    # Start with Node.js 22 (Debian-based)
# Why: Need Node.js for OpenClaw and NemoClaw plugin

RUN apt-get install python3 ...      # Install system packages
# Why: Python for blueprint runner, curl/git for agent operations

RUN groupadd sandbox && useradd sandbox  # Create unprivileged user
# Why: Security — agent runs as "sandbox" not root

RUN npm install -g openclaw@2026.3.11   # Install OpenClaw CLI
# Why: The agent framework

COPY nemoclaw/dist/ /opt/nemoclaw/dist/ # Copy compiled plugin
# Why: Plugin runs inside the sandbox

USER sandbox                         # Switch to unprivileged user
# Why: Everything from here runs without root privileges

ENTRYPOINT ["/bin/bash"]             # Default command
# Why: OpenShell overrides this when creating the sandbox
```

### Hands-On: Build the Image Locally

```bash
# Build the sandbox image
docker build -t nemoclaw-sandbox:local .

# Run it interactively (without OpenShell) to explore
docker run -it --rm nemoclaw-sandbox:local

# Inside the container:
whoami                    # → sandbox
ls /opt/nemoclaw/         # → dist/, openclaw.plugin.json, package.json
openclaw --version        # → 2026.3.11
python3 --version         # → 3.11.x
exit
```

### Self-Check Questions

- [ ] Why does the Dockerfile use `node:22-slim` instead of `ubuntu:22.04`?
- [ ] What is the purpose of the `sandbox` user?
- [ ] Why is `npm install --omit=dev` used instead of `npm install`?
- [ ] What would happen if `USER sandbox` was removed?

---

## Module 9: Security Architecture

**Goal**: Understand the multi-layered security model.

### Theory

NemoClaw uses **defense-in-depth** — multiple independent security layers:

```
Layer 1: Container isolation (Docker namespace)
Layer 2: Filesystem isolation (Landlock)
Layer 3: Process restrictions (seccomp, unprivileged user)
Layer 4: Network control (network namespace + policy proxy)
Layer 5: Credential isolation (OpenShell-managed injection)
```

### Hands-On: Study the Base Security Policy

**File to study**: `nemoclaw-blueprint/policies/openclaw-sandbox.yaml`

Read each section:

1. **`filesystem_policy`** (lines 18-31):
   - `read_only`: Agent can read but not write to /usr, /lib, /proc, etc.
   - `read_write`: Agent can read AND write to /sandbox and /tmp only

2. **`process`** (lines 37-39):
   - `run_as_user: sandbox` — No root access

3. **`network_policies`** (lines 40-168):
   Each policy has:
   - `name`: Identifier
   - `endpoints`: List of allowed hosts with port, method, and path rules
   - `binaries`: Which programs can use this policy

**Exercise**: Answer these questions by reading the policy file:
- Can the agent access `google.com`? (No — not in the allowlist)
- Can the agent access `api.anthropic.com`? (Yes — in `claude_code` policy)
- Can the agent call `POST /api/v1/messages` on `api.anthropic.com`? (Yes — `method: "*"` allows all)
- Can the `git` binary access `api.anthropic.com`? (No — only `/usr/local/bin/claude` is allowed)

### Hands-On: Study a Policy Preset

**File to study**: Any file in `nemoclaw-blueprint/policies/presets/`

For example, `slack.yaml` would declare:
```yaml
network_policies:
  slack:
    endpoints:
      - host: slack.com
        port: 443
      - host: api.slack.com
        port: 443
      - host: hooks.slack.com
        port: 443
```

### Self-Check Questions

- [ ] What is Landlock and how does it differ from traditional Unix permissions?
- [ ] Why are network policies per-binary? Why not just per-host?
- [ ] How does the agent access AI models if it can't see the API key?
- [ ] What happens when the agent tries to reach an unlisted host?

---

## Module 10: Inference Routing

**Goal**: Understand how AI model calls flow from agent to model and back.

### Hands-On: Trace an Inference Request

Starting from the agent's perspective:

1. Agent generates an HTTP request to `https://inference.local/v1/chat/completions`
2. This URL is configured in `/sandbox/.openclaw/openclaw.json` (see Dockerfile line 52-65)
3. The `inference.local` hostname resolves to the OpenShell gateway proxy
4. The gateway checks: is this sandbox allowed to make inference calls? (Yes, via provider config)
5. The gateway injects the API key: `Authorization: Bearer nvapi-...`
6. The gateway forwards to the real endpoint: `https://integrate.api.nvidia.com/v1/chat/completions`
7. Response flows back through the same path

### Hands-On: Study Provider Registration

**File to study**: `nemoclaw/src/index.ts:203-245`

The `registerProvider()` call declares:
- Provider ID and aliases
- Supported models with context windows and max output
- Authentication method (Bearer token via environment variable)

### Hands-On: Study Blueprint Profiles

**File to study**: `nemoclaw-blueprint/blueprint.yaml:27-55`

Compare the profiles:
- `default`: NVIDIA Cloud, model nemotron-3-super-120b, NVIDIA_API_KEY
- `ncp`: Dynamic endpoint, same model, same key
- `nim-local`: Local NIM server on port 8000, NIM_API_KEY
- `vllm`: Local vLLM on port 8000, OPENAI_API_KEY with dummy default

### Self-Check Questions

- [ ] Why does the agent connect to `inference.local` instead of the real API?
- [ ] How is the API key injected without the agent seeing it?
- [ ] What is the difference between `provider_type: "nvidia"` and `"openai"`?
- [ ] Why does the vLLM profile have `credential_default: "dummy"`?

---

## Module 11: Migration & State Management

**Goal**: Understand how existing OpenClaw installations are migrated and how state is persisted.

### Hands-On: Study the Migration State Machine

**File to study**: `nemoclaw/src/commands/migration-state.ts`

This is the most complex module. It handles:
- Detecting the host OpenClaw installation
- Creating tar archives that preserve symlinks
- Generating manifests with path mappings
- Restoring snapshots during eject

### Hands-On: Study State Persistence

**File to study**: `nemoclaw/src/blueprint/state.ts`

Simple pattern: load JSON from file → modify → save JSON to file.

State file location: `~/.nemoclaw/state/nemoclaw.json`

**Exercise**: Read `loadState()` and `saveState()`. Answer:
- What happens if the state file doesn't exist?
- How is `updatedAt` maintained?
- What does `clearState()` do?

### Hands-On: Trace the Full Migration Flow

Follow `cliMigrate()` in `nemoclaw/src/commands/migrate.ts` step by step:

1. `detectHostOpenClaw()` — What directories does it look for?
2. `resolveBlueprint()` + `verifyBlueprintDigest()` — Same as launch
3. `execBlueprint(plan)` + `execBlueprint(apply)` — Create sandbox
4. `createSnapshotBundle()` — What goes into the snapshot?
5. `buildMigrationArchives()` — How are tar archives created?
6. `syncSnapshotBundleIntoSandbox()` — How are archives copied?
7. `verifySandboxMigration()` — What does the verification script check?

### Self-Check Questions

- [ ] What is a "snapshot" and why is it needed?
- [ ] How does NemoClaw handle external roots (symlinked directories)?
- [ ] How does config path rewriting work?
- [ ] What happens if verification fails after sync?

---

## Module 12: Testing Strategies

**Goal**: Understand how NemoClaw is tested and how to write new tests.

### Hands-On: Read All Test Files

Study each test file in `test/`:

1. `test/cli.test.js` — Tests the CLI dispatcher logic
2. `test/registry.test.js` — Tests sandbox registry CRUD
3. `test/credentials.test.js` — Tests credential storage/retrieval
4. `test/policies.test.js` — Tests policy preset application
5. `test/preflight.test.js` — Tests system prerequisite checks
6. `test/nim.test.js` — Tests NIM container detection

### Hands-On: Run Tests with Verbose Output

```bash
node --test --test-reporter spec test/registry.test.js
```

### Exercise: Write a New Test

Create `test/example.test.js`:
```javascript
const { describe, it } = require("node:test");
const assert = require("node:assert/strict");

describe("Example", () => {
  it("should add numbers", () => {
    assert.strictEqual(1 + 1, 2);
  });
});
```

Run it: `node --test test/example.test.js`

### Self-Check Questions

- [ ] Why does NemoClaw use Node.js built-in test runner instead of Jest?
- [ ] How do the tests avoid calling real `openshell` commands?
- [ ] What testing patterns do you see? (Mocking, fixtures, assertions)

---

## Module 13: Integration — Putting It All Together

**Goal**: See how all the pieces fit together by tracing complete user journeys.

### Exercise 1: Trace `nemoclaw onboard` End-to-End

Draw a sequence diagram of every function call, file read, shell command, and state change from `nemoclaw onboard` to the final dashboard output. Include:
- Which module handles each step
- Which files are read/written
- Which `openshell` commands are executed
- Which state files are created

### Exercise 2: Trace `openclaw nemoclaw migrate` End-to-End

Same exercise but for migration. Include:
- Host detection logic
- Blueprint resolution + verification
- Snapshot creation + archiving
- Sandbox sync + verification
- State persistence

### Exercise 3: Trace an Inference Request

Follow a single prompt from the user typing it in `openclaw tui` through:
- OpenClaw agent processing
- HTTP request to `inference.local`
- OpenShell gateway interception
- Credential injection
- NVIDIA API call
- Response return path

### Exercise 4: Draw the Complete Architecture

Create your own version of the architecture diagram including:
- All three layers (Host CLI, Plugin, Blueprint)
- State files and their locations
- Communication protocols between layers
- External dependencies (OpenShell, Docker, NVIDIA API)

---

## Module 14: Contributing & Extending

**Goal**: Be ready to make your first contribution.

### Exercise 1: Add a New Policy Preset

Create a preset for a service you use (e.g., Google Cloud, AWS, Azure). Follow the pattern in existing presets.

### Exercise 2: Add a New CLI Command

Add a `nemoclaw <name> restart` command that stops and restarts a sandbox.

### Exercise 3: Add a New Inference Provider

Add support for a local model provider (e.g., LM Studio). Follow the pattern in `index.ts` and `blueprint.yaml`.

### Exercise 4: Write Missing Tests

Identify untested functions and write tests for them. Focus on:
- Edge cases (empty inputs, missing files)
- Error paths (invalid config, network failures)
- State transitions (before/after operations)

---

## Completion Checklist

After completing all modules, you should be able to:

- [ ] Explain NemoClaw's architecture at three different abstraction levels
- [ ] Trace any user command through the complete code path
- [ ] Build the project from source
- [ ] Run and understand all tests
- [ ] Add a new CLI command (host or plugin)
- [ ] Add a new policy preset
- [ ] Add a new inference provider
- [ ] Modify the blueprint orchestrator
- [ ] Understand the security model and its five layers
- [ ] Debug issues using logs, state files, and openshell commands
- [ ] Explain the migration flow including snapshot/restore
- [ ] Read and understand the Dockerfile
- [ ] Explain the TypeScript-to-Python communication protocol

**If you can check every box, you've gone from zero to hero.**
