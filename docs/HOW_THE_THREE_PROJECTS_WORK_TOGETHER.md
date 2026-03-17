# How NemoClaw, OpenShell, Docker, and OpenClaw Work Together

This document explains the relationship between the four technologies that make up the
NemoClaw ecosystem. If you only read one document before diving into the codebase,
make it this one.

---

## Table of Contents

1. [The Four Technologies at a Glance](#1-the-four-technologies-at-a-glance)
2. [What Each Technology Does (Individually)](#2-what-each-technology-does-individually)
3. [Why They Need Each Other](#3-why-they-need-each-other)
4. [The Analogy: A Robot, a Building, Rooms, and a Contractor](#4-the-analogy-a-robot-a-building-rooms-and-a-contractor)
5. [How They Communicate](#5-how-they-communicate)
6. [Docker's Multiple Roles](#6-dockers-multiple-roles)
7. [Lifecycle: What Happens When You Run `nemoclaw onboard`](#7-lifecycle-what-happens-when-you-run-nemoclaw-onboard)
8. [Lifecycle: What Happens at Runtime](#8-lifecycle-what-happens-at-runtime)
9. [Ownership Boundaries](#9-ownership-boundaries)
10. [Dependency Diagram](#10-dependency-diagram)
11. [Where Each Technology Lives](#11-where-each-technology-lives)
12. [Common Questions](#12-common-questions)

---

## 1. The Four Technologies at a Glance

```
+======================================================================+
|                                                                       |
|   OpenClaw          Docker             OpenShell        NemoClaw      |
|   =========         ======             ==========      =========     |
|   The AI Agent      The Container      The Security     The Glue     |
|                     Engine             Runtime                       |
|                                                                       |
|   "The brain"       "The building      "The security    "The         |
|                      material"          system"          contractor"  |
|                                                                       |
+======================================================================+
```

| Technology | One-Line Description | Role in NemoClaw |
|---|---|---|
| **OpenClaw** | An autonomous AI agent framework | Does the actual AI work — conversations, code, APIs |
| **Docker** | Container engine for building and running isolated environments | Provides the containers that everything runs inside |
| **OpenShell** | A secure runtime for autonomous agents (built on Docker) | Adds security layers ON TOP of Docker containers |
| **NemoClaw** | Plugin that wires OpenClaw into OpenShell (this repo) | Orchestrates setup — builds images, creates sandboxes, configures everything |

---

## 2. What Each Technology Does (Individually)

### OpenClaw — The AI Agent

OpenClaw is an **AI agent framework**. It's an AI assistant that can:
- Have conversations with you
- Write and execute code
- Call APIs and use tools
- Manage files and projects
- Run autonomously (without you typing every command)

**Without the others**: OpenClaw runs directly on your host machine with full access
to your filesystem, network, and all installed tools. No guardrails.

**Source in this repo**: OpenClaw itself is NOT in this repo. It's an external
dependency installed via `npm install -g openclaw@2026.3.11`. NemoClaw interacts
with it through the `openclaw` CLI and plugin API.

### Docker — The Container Engine

Docker is the **foundation** that makes isolation possible. It provides:

- **Images**: Immutable templates containing an OS, libraries, and applications
  (like a snapshot of a configured computer)
- **Containers**: Running instances of images — isolated environments with their own
  filesystem, network stack, and process tree
- **Image building**: `docker build` reads a `Dockerfile` recipe and produces an image
- **Container lifecycle**: `docker run`, `docker stop`, `docker rm` manage containers
- **GPU passthrough**: `--gpus all` flag lets containers use NVIDIA GPUs

**Docker is to NemoClaw what concrete is to a building** — it's the raw material
that everything is built from. OpenShell uses Docker containers. NIM inference runs
in Docker containers. The sandbox itself is a Docker container.

**Without the others**: Docker is a generic container engine. It can run any
application in isolation, but it doesn't know about AI agents, security policies,
or inference routing. You'd have to manually configure everything.

**Key concepts for NemoClaw**:

| Docker Concept | What It Is | NemoClaw Usage |
|---|---|---|
| **Dockerfile** | Build recipe for an image | `Dockerfile` in repo root builds the sandbox image |
| **Image** | Immutable template | Contains Node.js + OpenClaw + NemoClaw plugin |
| **Container** | Running instance of image | The sandbox is a Docker container |
| **Volume** | Persistent storage | `/sandbox` directory persists data |
| **Network** | Isolated network stack | Network namespace isolates sandbox traffic |
| **`docker build`** | Build an image from Dockerfile | NemoClaw builds sandbox image during onboard |
| **`docker run`** | Start a container | OpenShell uses Docker to start sandboxes |
| **`docker pull`** | Download an image | NemoClaw pulls NIM model images |
| **`--gpus all`** | GPU access for containers | NIM containers need GPU for local inference |

**Source in this repo**: Docker itself is NOT in this repo. It's installed separately.
NemoClaw interacts with it via the `docker` CLI (for NIM) and indirectly through
OpenShell (for sandboxes).

### OpenShell — The Security Runtime

OpenShell is a **security layer built on top of Docker**. Think of it this way:
Docker provides basic container isolation; OpenShell adds enterprise-grade security
controls on top.

**What Docker gives you**: Process isolation, filesystem isolation, network namespace.

**What OpenShell adds on top of Docker**:

| Capability | Docker Alone | Docker + OpenShell |
|---|---|---|
| Filesystem | Container has its own FS | Landlock enforces per-path read/write rules |
| Network | Container has its own network | Declarative YAML policies control which hosts are allowed |
| Process | Container has its own PID space | seccomp filters block dangerous syscalls |
| Credentials | You'd pass env vars (visible to app) | Credentials injected at proxy layer (invisible to app) |
| Monitoring | `docker logs` | Real-time TUI with approval queue |
| Inference | You'd configure manually | Transparent proxy intercepts and routes model calls |

**How OpenShell uses Docker**:

```
+-------------------------------------------+
|  OpenShell Gateway                         |
|  (itself a Docker container running k3s)  |
|                                            |
|  Contains:                                 |
|  - k3s (lightweight Kubernetes)            |
|  - Network proxy                           |
|  - Policy engine                           |
|  - Credential manager                      |
|                                            |
|  Manages:                                  |
|  +-------------------------------------+   |
|  |  Sandbox Container                  |   |
|  |  (a Docker container)               |   |
|  |                                     |   |
|  |  +-------------------------------+  |   |
|  |  | OpenClaw + NemoClaw plugin    |  |   |
|  |  | (running inside)              |  |   |
|  |  +-------------------------------+  |   |
|  +-------------------------------------+   |
+-------------------------------------------+
         |
         | All running on Docker
         v
+-------------------------------------------+
|  Docker Engine                             |
|  (on the host machine)                     |
+-------------------------------------------+
```

**Source in this repo**: OpenShell itself is NOT in this repo. Installed from
https://github.com/NVIDIA/OpenShell.

### NemoClaw — The Glue

NemoClaw is the **bridge** that connects all three technologies:

- **Uses Docker directly**: Builds sandbox images (`docker build`), manages NIM
  containers (`docker run/stop/rm`), checks Docker health
- **Uses OpenShell**: Creates sandboxes, configures providers, applies policies,
  routes inference — all via the `openshell` CLI
- **Integrates with OpenClaw**: Registers as a plugin, configures models, sets up
  agent configuration

**Without the other three**: NemoClaw does nothing — it's purely an integration layer.

**Source in this repo**: This entire repository IS NemoClaw.

---

## 3. Why They Need Each Other

### The Layer Cake

Each technology builds on the one below it:

```
+---------------------------------------------------------------+
|  Layer 4:  NemoClaw (orchestration)                           |
|            "I know how to set up OpenClaw inside OpenShell"   |
+---------------------------------------------------------------+
|  Layer 3:  OpenClaw (AI agent)                                |
|            "I do the AI work — but I need a place to run"     |
+---------------------------------------------------------------+
|  Layer 2:  OpenShell (security runtime)                       |
|            "I add security on top of Docker containers"       |
+---------------------------------------------------------------+
|  Layer 1:  Docker (container engine)                          |
|            "I provide isolated containers"                    |
+---------------------------------------------------------------+
|  Layer 0:  Linux Kernel (Landlock, seccomp, namespaces)       |
|            "I provide the low-level isolation primitives"     |
+---------------------------------------------------------------+
```

### The Problem Without NemoClaw

```
Without NemoClaw, you'd manually run:

  # 1. Install Docker (if not present)
  sudo apt install docker.io

  # 2. Install OpenShell
  curl -fsSL https://github.com/NVIDIA/OpenShell/releases/... | bash

  # 3. Start the OpenShell gateway (runs as a Docker container)
  openshell gateway start --name my-gateway --gpu

  # 4. Build a sandbox Docker image with OpenClaw pre-installed
  docker build -t my-openclaw-sandbox .

  # 5. Create the sandbox (OpenShell creates a Docker container)
  openshell sandbox create \
    --from my-openclaw-sandbox \
    --name my-assistant \
    --policy openclaw-sandbox.yaml \
    --forward 18789

  # 6. Create an inference provider
  openshell provider create \
    --name nvidia-nim \
    --type openai \
    --credential "NVIDIA_API_KEY=nvapi-..." \
    --config "OPENAI_BASE_URL=https://integrate.api.nvidia.com/v1"

  # 7. Route inference
  openshell inference set \
    --provider nvidia-nim \
    --model nvidia/nemotron-3-super-120b-a12b

  # 8. (Optional) Start a local NIM Docker container for inference
  docker run -d --gpus all -p 8000:8000 --name nim-server \
    nvcr.io/nvidia/nemotron-3-super-120b:latest

  # 9. Apply policy presets
  openshell policy set --sandbox my-assistant --policy slack-preset.yaml

  # 10. Port forward
  openshell forward start --background 18789 my-assistant

  # 11. Connect
  openshell sandbox connect my-assistant
```

That's 11 commands across **two different CLIs** (`docker` and `openshell`),
with many flags, specific ordering, and error handling.

### With NemoClaw

```
  nemoclaw onboard
```

One command. NemoClaw handles all 11 steps (and more) through an interactive wizard.

### The Dependency Chain

```
NemoClaw  DEPENDS ON  Docker     (builds images, runs NIM containers)
NemoClaw  DEPENDS ON  OpenShell  (calls openshell CLI for sandbox/policy/inference)
NemoClaw  DEPENDS ON  OpenClaw   (installs in sandbox, registers as plugin)
OpenShell DEPENDS ON  Docker     (runs gateway + sandboxes as Docker containers)
OpenClaw  DEPENDS ON  Node.js    (runs on Node.js runtime)

OpenShell does NOT depend on NemoClaw or OpenClaw
OpenClaw  does NOT depend on NemoClaw, OpenShell, or Docker
Docker    does NOT depend on any of the above
```

NemoClaw is the only component that "knows about" all three others.

---

## 4. The Analogy: A Robot, a Building, Rooms, and a Contractor

```
+--------------------------------------------------------------+
|                                                               |
|  OpenClaw  =  A smart robot                                   |
|               It can clean, cook, manage tasks,               |
|               call services, order supplies.                  |
|               Very capable, but unsupervised.                 |
|                                                               |
|  Docker    =  Building materials: walls, doors, windows       |
|               You CAN build a room from these materials,      |
|               but you get plain rooms with no locks            |
|               and no alarm system.                            |
|                                                               |
|  OpenShell =  A security company that:                        |
|               - Uses the building materials (Docker) to       |
|                 build high-security rooms                     |
|               - Installs locks on doors (network policies)    |
|               - Installs cameras (monitoring TUI)             |
|               - Manages keys in a vault (credential mgr)      |
|               - Restricts which rooms you can enter (fs)      |
|               - Installs an alarm system (seccomp)            |
|                                                               |
|  NemoClaw  =  The contractor who:                             |
|               - Draws the room blueprints (blueprint.yaml)    |
|               - Tells the security company what to build      |
|               - Installs the robot in the finished room       |
|               - Configures which doors should be unlocked     |
|               - Sets up the robot's mail (inference routing)  |
|               - Can move the robot to a new room (migrate)    |
|               - Can take the robot out (eject)                |
|                                                               |
+--------------------------------------------------------------+
```

**Docker without OpenShell** = You get a room (container), but it has no locks on
the doors. The robot can go anywhere and do anything.

**OpenShell without NemoClaw** = You get a high-security room, but you have to
carry the robot in yourself and manually configure every lock.

**NemoClaw** = The contractor who handles everything: draws the blueprints, orders
the security company to build the room, carries the robot in, and hands you the key.

---

## 5. How They Communicate

### Communication Architecture

```
+------------------+                     +------------------+
|                  |  Plugin API calls   |                  |
|    OpenClaw      |<------------------->|    NemoClaw      |
|    (host proc)   |  register(),        |    (plugin)      |
|                  |  registerCli(),     |                  |
|                  |  registerProvider() |                  |
+------------------+                     +--------+---------+
                                                  |
                                       +----------+----------+
                                       |                     |
                              subprocess |                   | docker CLI
                              (python3)  |                   | (for NIM)
                                       |                     |
                              +--------+-------+    +--------+-------+
                              |                |    |                |
                              |   NemoClaw     |    |   Docker       |
                              |   (blueprint)  |    |   Engine       |
                              |                |    |   (NIM only)   |
                              +--------+-------+    +----------------+
                                       |
                                       | openshell CLI commands
                                       |
                              +--------+-------+
                              |                |
                              |   OpenShell    |
                              |   (runtime)    |
                              |                |
                              +--------+-------+
                                       |
                                       | uses Docker internally
                                       |
                              +--------+-------+
                              |                |
                              |   Docker       |
                              |   Engine       |
                              |                |
                              +--------+-------+
                                       |
                                       | creates containers
                                       |
                              +--------+-------+
                              |   SANDBOX      |
                              |   CONTAINER    |
                              |  (Docker ctnr) |
                              |  +----------+  |
                              |  | OpenClaw |  |
                              |  |(sandboxed)|  |
                              |  +----------+  |
                              +----------------+
```

### Interface Contracts

There are **five interfaces** where the technologies talk to each other:

#### Interface 1: NemoClaw Plugin ↔ OpenClaw Host

**How**: OpenClaw's Plugin API (TypeScript interfaces)

**Where in code**: `nemoclaw/src/index.ts:179` — the `register(api)` function

**What happens**:
- OpenClaw loads NemoClaw as a plugin at startup
- OpenClaw calls `register(api)` passing an API object
- NemoClaw uses the API to register:
  - `/nemoclaw` slash command (for chat interface)
  - `openclaw nemoclaw <cmd>` CLI subcommands (for terminal)
  - `nvidia-nim` inference provider (model catalog + auth)

```typescript
// NemoClaw registers itself with OpenClaw:
export default function register(api: OpenClawPluginApi): void {
    api.registerCommand({ name: "nemoclaw", handler: ... });
    api.registerCli((ctx) => { ... });
    api.registerProvider({ id: "nvidia-nim", ... });
}
```

#### Interface 2: NemoClaw TypeScript ↔ NemoClaw Python (Blueprint)

**How**: Subprocess with stdout protocol

**Where in code**: `nemoclaw/src/blueprint/exec.ts:58` spawns `python3 runner.py`

**What happens**:
- TypeScript spawns Python as a child process
- Python sends progress via stdout: `PROGRESS:50:Creating sandbox`
- Python sends run ID via stdout: `RUN_ID:nc-20260315-143022-abcd1234`
- TypeScript reads stdout and parses these lines
- Exit code 0 = success, non-zero = failure

```
TypeScript:  spawn("python3", ["runner.py", "apply", "--profile", "default"])
Python:      PROGRESS:20:Creating OpenClaw sandbox
             PROGRESS:50:Configuring inference provider
             RUN_ID:nc-20260315-143022-a1b2c3d4
             Sandbox 'openclaw' is ready.
             [exit code 0]
```

#### Interface 3: NemoClaw Blueprint ↔ OpenShell

**How**: CLI commands (`openshell ...`)

**Where in code**: `nemoclaw-blueprint/orchestrator/runner.py:169` and throughout

**What happens**:
- Python calls `openshell` CLI commands via `subprocess.run()`
- Each command is an argv list (never `shell=True`, for security)
- OpenShell executes the command and returns via exit code + stdout

```python
# NemoClaw tells OpenShell what to do:
run_cmd(["openshell", "sandbox", "create", "--from", image, "--name", "openclaw"])
run_cmd(["openshell", "provider", "create", "--name", "nvidia-nim", ...])
run_cmd(["openshell", "inference", "set", "--provider", "nvidia-nim", ...])
```

#### Interface 4: NemoClaw ↔ Docker (Direct)

**How**: CLI commands (`docker ...`)

**Where in code**: `bin/lib/nim.js` and `bin/lib/onboard.js`

NemoClaw talks to Docker directly for two purposes:

**Purpose A — NIM containers** (local inference):
```javascript
// bin/lib/nim.js:124 — Pull a model image
run(`docker pull ${image}`);

// bin/lib/nim.js:140 — Start NIM container with GPU
run(`docker run -d --gpus all -p ${port}:8000 --name ${name} --shm-size 16g ${image}`);

// bin/lib/nim.js:171 — Stop NIM container
run(`docker stop ${name} 2>/dev/null || true`);

// bin/lib/nim.js:178 — Check NIM container status
runCapture(`docker inspect --format '{{.State.Status}}' ${name}`);
```

**Purpose B — Sandbox image building** (during onboard):
```javascript
// bin/lib/onboard.js:173-176 — Stage build context and copy Dockerfile
fs.copyFileSync(path.join(ROOT, "Dockerfile"), path.join(buildCtx, "Dockerfile"));
// OpenShell's sandbox create reads the Dockerfile and builds the image internally
```

**Purpose C — Preflight health check**:
```javascript
// bin/lib/onboard.js:26 — Check Docker is running
runCapture("docker info", { ignoreError: false });
```

#### Interface 5: OpenShell ↔ Docker (Internal)

**How**: OpenShell uses Docker internally to manage its gateway and sandboxes

**What happens** (NemoClaw doesn't see this — it's inside OpenShell):
- `openshell gateway start` → Docker creates a container running k3s
- `openshell sandbox create` → Docker creates a container from the sandbox image
- `openshell sandbox connect` → Docker exec into the sandbox container
- `openshell sandbox delete` → Docker stops and removes the container
- Network policies → Docker network namespaces + OpenShell proxy

```
When NemoClaw runs: openshell sandbox create --from Dockerfile --name my-assistant

OpenShell internally does:
  1. docker build -t <temp-image> <build-context>    # Build from Dockerfile
  2. docker run -d --name <sandbox-id> <temp-image>   # Start container
  3. Apply Landlock rules to container                 # Filesystem isolation
  4. Configure seccomp profile                         # Syscall filtering
  5. Set up network proxy in k3s                       # Network isolation
  6. Register sandbox in OpenShell state               # Management metadata
```

#### Interface 6: OpenShell ↔ OpenClaw (at runtime, inside sandbox)

**How**: Network proxy (transparent to OpenClaw)

**What happens**:
- OpenClaw agent makes HTTP requests to `https://inference.local/v1`
- OpenShell's gateway proxy intercepts the request (transparent)
- Gateway injects API credentials (Bearer token)
- Gateway forwards to real endpoint (e.g., integrate.api.nvidia.com)
- Response flows back through the proxy to the agent
- OpenClaw never sees the real API key or endpoint

```
OpenClaw agent                    OpenShell proxy               NVIDIA API
     |                                 |                            |
     |  POST inference.local/v1        |                            |
     |  (no auth header)               |                            |
     |------------------------------->|                             |
     |                                 |  POST integrate.api.       |
     |                                 |  nvidia.com/v1             |
     |                                 |  Authorization: Bearer     |
     |                                 |  nvapi-*****               |
     |                                 |--------------------------->|
     |                                 |                            |
     |                                 |  200 OK + response body    |
     |                                 |<---------------------------|
     |  200 OK + response body         |                            |
     |<-------------------------------|                             |
```

---

## 6. Docker's Multiple Roles

Docker serves **four distinct purposes** in NemoClaw. This is a common source
of confusion, so here's the complete picture:

```
+----------------------------------------------------------------------+
|                  Docker's Four Roles in NemoClaw                      |
+----------------------------------------------------------------------+
|                                                                       |
|  Role 1: OPENSHELL GATEWAY HOST                                      |
|  ================================                                     |
|  OpenShell's gateway (its control plane) runs as a Docker container.  |
|  Inside this container, OpenShell runs k3s (lightweight Kubernetes)   |
|  which manages sandboxes and the network proxy.                       |
|                                                                       |
|  Who creates it: OpenShell (via `openshell gateway start`)            |
|  Who triggers it: NemoClaw (during onboard step 5)                    |
|  Container name: managed by OpenShell internally                      |
|                                                                       |
|  +------------------------------------------+                         |
|  |  Docker Container: OpenShell Gateway      |                         |
|  |  +------------------------------------+  |                         |
|  |  |  k3s (Kubernetes)                  |  |                         |
|  |  |  - Network proxy                   |  |                         |
|  |  |  - Policy engine                   |  |                         |
|  |  |  - Credential manager              |  |                         |
|  |  +------------------------------------+  |                         |
|  +------------------------------------------+                         |
|                                                                       |
+----------------------------------------------------------------------+
|                                                                       |
|  Role 2: SANDBOX CONTAINER HOST                                       |
|  ================================                                     |
|  The sandbox where OpenClaw runs IS a Docker container.               |
|  OpenShell creates it from a Docker image built by NemoClaw.          |
|                                                                       |
|  Who builds the image: NemoClaw (`Dockerfile` in repo root)           |
|  Who creates the container: OpenShell (`openshell sandbox create`)    |
|  Who triggers creation: NemoClaw (during onboard step 7)              |
|  Container contents: Node.js + OpenClaw + NemoClaw plugin + Python    |
|                                                                       |
|  +------------------------------------------+                         |
|  |  Docker Container: Sandbox                |                         |
|  |  +------------------------------------+  |                         |
|  |  |  OpenClaw agent                    |  |                         |
|  |  |  NemoClaw plugin                   |  |                         |
|  |  |  Python 3.11 + PyYAML              |  |                         |
|  |  |  Runs as 'sandbox' user (non-root) |  |                         |
|  |  +------------------------------------+  |                         |
|  |  + Landlock FS rules (from OpenShell)     |                         |
|  |  + seccomp filter (from OpenShell)        |                         |
|  |  + Network namespace (from OpenShell)     |                         |
|  +------------------------------------------+                         |
|                                                                       |
+----------------------------------------------------------------------+
|                                                                       |
|  Role 3: NIM INFERENCE CONTAINER HOST                                 |
|  =====================================                                |
|  When using local GPU inference (not cloud), NIM models run as        |
|  Docker containers with GPU access.                                   |
|                                                                       |
|  Who creates it: NemoClaw DIRECTLY (`docker run` in nim.js)           |
|  Container name: nemoclaw-nim-<sandbox-name>                          |
|  GPU access: --gpus all                                               |
|  Port: 8000 (OpenAI-compatible API)                                   |
|                                                                       |
|  +------------------------------------------+                         |
|  |  Docker Container: NIM Model Server       |                         |
|  |  +------------------------------------+  |                         |
|  |  |  NVIDIA NIM runtime                |  |                         |
|  |  |  + Loaded model weights            |  |                         |
|  |  |  + GPU access (--gpus all)         |  |                         |
|  |  |  + OpenAI-compatible REST API      |  |                         |
|  |  +------------------------------------+  |                         |
|  +------------------------------------------+                         |
|                                                                       |
+----------------------------------------------------------------------+
|                                                                       |
|  Role 4: IMAGE BUILDER                                                |
|  ======================                                               |
|  Docker builds the sandbox image from NemoClaw's Dockerfile.          |
|  This happens once (during `nemoclaw onboard`) and is cached.         |
|                                                                       |
|  Dockerfile location: repo root `Dockerfile`                          |
|  What it installs:                                                    |
|    FROM node:22-slim                   (base image)                   |
|    + python3, curl, git                (system tools)                 |
|    + openclaw@2026.3.11               (AI agent)                     |
|    + nemoclaw plugin (compiled TS)     (this project)                |
|    + blueprint files                   (orchestration)               |
|    + sandbox user (non-root)           (security)                    |
|                                                                       |
+----------------------------------------------------------------------+
```

### Docker Commands Used by Each Component

| Who Calls Docker | Command | Purpose | Where in Code |
|---|---|---|---|
| **NemoClaw** (preflight) | `docker info` | Check Docker is running | `bin/lib/onboard.js:26` |
| **NemoClaw** (NIM) | `docker pull <image>` | Download NIM model image | `bin/lib/nim.js:124` |
| **NemoClaw** (NIM) | `docker run -d --gpus all ...` | Start NIM container | `bin/lib/nim.js:140` |
| **NemoClaw** (NIM) | `docker stop/rm` | Stop NIM container | `bin/lib/nim.js:169-172` |
| **NemoClaw** (NIM) | `docker inspect` | Check NIM health | `bin/lib/nim.js:179` |
| **OpenShell** (gateway) | `docker run` (internal) | Start k3s gateway container | Inside OpenShell |
| **OpenShell** (sandbox) | `docker build` (internal) | Build sandbox image | Inside OpenShell |
| **OpenShell** (sandbox) | `docker run` (internal) | Start sandbox container | Inside OpenShell |
| **OpenShell** (sandbox) | `docker exec` (internal) | Connect to sandbox | Inside OpenShell |

### Docker Configuration Requirements

NemoClaw checks Docker configuration during preflight (`bin/lib/preflight.js`):

**cgroup v2 compatibility**: On Linux systems with cgroup v2 (modern kernels),
Docker must be configured with `"default-cgroupns-mode": "host"` in
`/etc/docker/daemon.json`. This is because OpenShell runs k3s inside a Docker
container, and k3s needs host cgroup access to manage its own cgroup hierarchies.

```
Without this fix:
  k3s inside Docker fails with:
  "openat2 /sys/fs/cgroup/kubepods/pids.max: no such file or directory"

Fix (run by `nemoclaw setup-spark`):
  Add to /etc/docker/daemon.json:
  { "default-cgroupns-mode": "host" }
  Then restart Docker.
```

**Colima (macOS)**: On macOS with Colima (Docker alternative), NemoClaw auto-detects
the Colima socket path and patches CoreDNS for DNS resolution inside the gateway.

---

## 7. Lifecycle: What Happens When You Run `nemoclaw onboard`

This traces the complete flow through all four technologies:

```
Step    Who Does It              What Happens
-----   ---------------------    -----------------------------------------------

 1      NemoClaw --> Docker       Checks Docker is running
                                  Command: docker info
                                  If not running: abort with instructions

 2      NemoClaw                  Checks OpenShell is installed
                                  Command: openshell --version
                                  If not installed: downloads and installs

 3      NemoClaw --> Docker       Checks Docker cgroup v2 configuration
                                  Reads /etc/docker/daemon.json
                                  Verifies "default-cgroupns-mode": "host"

 4      NemoClaw                  Detects GPU hardware
                                  Runs: nvidia-smi (NVIDIA) or
                                        system_profiler (Apple)

 5      NemoClaw --> OpenShell    Calls: openshell gateway start --name nemoclaw
        OpenShell --> Docker      OpenShell starts a Docker container running k3s
                                  This is the gateway/control plane

 6      NemoClaw --> Docker       Stages a Docker build context:
                                  - Copies Dockerfile from repo root
                                  - Copies nemoclaw/ (compiled plugin)
                                  - Copies nemoclaw-blueprint/
                                  - Copies scripts/

 7      NemoClaw --> OpenShell    Calls: openshell sandbox create --from Dockerfile
        OpenShell --> Docker      OpenShell calls docker build to create the image,
                                  then docker run to start the container.
                                  Applies Landlock, seccomp, and network policies.
                                  Result: sandbox container running with OpenClaw
                                  and NemoClaw plugin pre-installed.

 8      NemoClaw                  Prompts user for NVIDIA API key
                                  Saves to ~/.nemoclaw/credentials.json (mode 0600)

 9      NemoClaw --> OpenShell    Calls: openshell provider create --name nvidia-nim
                                        --credential "NVIDIA_API_KEY=nvapi-..."
                                  OpenShell stores credentials for injection

10      NemoClaw --> OpenShell    Calls: openshell inference set
                                        --provider nvidia-nim
                                        --model nvidia/nemotron-3-super-120b-a12b
                                  OpenShell configures inference routing

11      OpenClaw (in sandbox)     OpenClaw starts with NemoClaw plugin pre-installed
                                  Plugin registers nvidia-nim provider + models

12      NemoClaw --> OpenShell    Applies policy presets (pypi, npm, etc.)
                                  Calls: openshell policy set for each preset

13      NemoClaw --> OpenShell    Calls: openshell forward start --background 18789
        OpenShell --> Docker      Sets up port forwarding from host to sandbox container

 *      (optional)
        NemoClaw --> Docker       If user chose local NIM inference:
                                  docker pull <nim-image>
                                  docker run -d --gpus all -p 8000:8000 <nim-image>
                                  Starts a SEPARATE Docker container for the model

RESULT: Three Docker containers are potentially running:
        1. OpenShell gateway (k3s) - always
        2. Sandbox (OpenClaw + NemoClaw) - always
        3. NIM model server (GPU) - only if local inference chosen
```

### Visualizing the Running State

After `nemoclaw onboard` completes:

```
+======================================================================+
|                          HOST MACHINE                                 |
|                                                                       |
|  +-------------------------------+                                    |
|  |  nemoclaw CLI                 |  (Node.js process, not a container)|
|  |  Manages everything from here |                                    |
|  +-------------------------------+                                    |
|                                                                       |
|  +============================= DOCKER ENGINE =======================+|
|  |                                                                    ||
|  |  +--------------------------+    +-----------------------------+   ||
|  |  | Container 1:             |    | Container 2:                |   ||
|  |  | OpenShell Gateway        |    | Sandbox                     |   ||
|  |  |                          |    |                             |   ||
|  |  | k3s cluster              |--->| OpenClaw agent              |   ||
|  |  | Network proxy            |    | NemoClaw plugin             |   ||
|  |  | Policy engine            |    | Python 3.11                 |   ||
|  |  | Credential manager       |    | Runs as 'sandbox' user      |   ||
|  |  |                          |    |                             |   ||
|  |  | Manages Container 2      |    | Filesystem: Landlock        |   ||
|  |  |                          |    | Process: seccomp            |   ||
|  |  |                          |    | Network: namespace + proxy  |   ||
|  |  +--------------------------+    +-----------------------------+   ||
|  |                                                                    ||
|  |  +--------------------------+  (only if local inference chosen)    ||
|  |  | Container 3:             |                                      ||
|  |  | NIM Model Server         |                                      ||
|  |  |                          |                                      ||
|  |  | GPU: --gpus all          |                                      ||
|  |  | Port: 8000               |                                      ||
|  |  | OpenAI-compatible API    |                                      ||
|  |  +--------------------------+                                      ||
|  |                                                                    ||
|  +====================================================================+|
|                                                                       |
+======================================================================+
```

---

## 8. Lifecycle: What Happens at Runtime

After setup, here's what happens during normal operation:

```
+-------------------------------------------------------------------+
|  HOST MACHINE                                                      |
|                                                                    |
|  User: nemoclaw my-assistant connect                               |
|      |                                                             |
|      v                                                             |
|  NemoClaw: openshell forward start ... && openshell sandbox connect|
|      |                                                             |
|      v                                                             |
|  OpenShell: docker exec (internally) into sandbox container       |
|      |                                                             |
+------|-------------------------------------------------------------+
       |
       v
+-------------------------------------------------------------------+
|  DOCKER CONTAINER: SANDBOX                                         |
|                                                                    |
|  User: openclaw tui                                                |
|      |                                                             |
|      v                                                             |
|  OpenClaw agent receives user message                              |
|      |                                                             |
|      | Agent decides to call AI model                              |
|      v                                                             |
|  OpenClaw: POST https://inference.local/v1/chat/completions       |
|      |     (this URL is set by NemoClaw in openclaw.json)          |
|      |                                                             |
|      v                                                             |
|  OpenShell proxy (running in gateway Docker container):            |
|      |  1. Intercepts request to inference.local                   |
|      |  2. Looks up provider config (set by NemoClaw during setup) |
|      |  3. Injects: Authorization: Bearer nvapi-***                |
|      |  4. Rewrites URL to: integrate.api.nvidia.com/v1            |
|      |  5. Checks network policy: nvidia endpoint allowed? YES     |
|      v                                                             |
+------|-------------------------------------------------------------+
       |
       | HTTPS with injected credentials
       v
+-------------------------------------------------------------------+
|  NVIDIA CLOUD API (or NIM Docker container on port 8000)           |
|  Returns: AI model response                                       |
+-------------------------------------------------------------------+
```

### Who Does What at Runtime

| Event | OpenClaw | Docker | OpenShell | NemoClaw |
|---|---|---|---|---|
| User connects to sandbox | - | Hosts the container | Routes `connect` to container | Calls `openshell sandbox connect` |
| User types a message | Processes it, runs the agent | Hosts the process | - | - |
| Agent calls AI model | Sends HTTP to inference.local | Carries network traffic | Intercepts, injects creds, forwards | Configured the provider |
| Agent accesses a file | Reads/writes normally | Provides filesystem | Landlock enforces path rules | Configured the policy |
| Agent calls an allowed API | Sends HTTP request | Carries network traffic | Checks policy, allows | Applied the preset |
| Agent calls a blocked API | Sends HTTP request | Carries network traffic | Blocks, logs to TUI | Not involved |
| NIM processes a request | - | Hosts the NIM container, provides GPU | Routes to NIM container | Started the NIM container |

**Key insight**: At runtime, **Docker is the silent workhorse** — it hosts all the
containers but doesn't make decisions. **OpenShell** makes security decisions.
**OpenClaw** does AI work. **NemoClaw** configured everything during setup and is
mostly idle (just status queries via `/nemoclaw`).

---

## 9. Ownership Boundaries

```
+-----------------+------------------+------------------+------------------+
|    OpenClaw     |     Docker       |    OpenShell     |    NemoClaw      |
+-----------------+------------------+------------------+------------------+
| AI agent exec   | Container runtime| Sandbox creation | Setup wizard     |
| Task planning   | Image building   | Network proxy    | Blueprint mgmt   |
| Code generation | Process isolation| Policy enforce   | Inference config |
| Tool usage      | Network stacks   | Credential inject| Migration logic  |
| Plugin system   | GPU passthrough  | FS isolation     | Policy presets   |
| Sessions        | Storage/volumes  | Process sandbox  | Rollback/eject   |
| Chat TUI        | Port forwarding  | Operator TUI     | NIM management   |
| Model selection | cgroup/namespace | Model routing    | Provider register|
+-----------------+------------------+------------------+------------------+
| "What to do"    | "Where to run"   | "What's allowed" | "How to set up"  |
+-----------------+------------------+------------------+------------------+
```

### Configuration Ownership

| Config File | Written By | Read By | Purpose |
|---|---|---|---|
| `Dockerfile` | NemoClaw | Docker (via OpenShell) | Sandbox image recipe |
| `openclaw.json` | NemoClaw (during setup) | OpenClaw | Agent config, model selection |
| `openclaw-sandbox.yaml` | NemoClaw | OpenShell | Security policy |
| `presets/*.yaml` | NemoClaw | OpenShell | Network allowlists |
| `blueprint.yaml` | NemoClaw | NemoClaw blueprint | Sandbox recipe |
| Provider config | NemoClaw via OpenShell | OpenShell | Inference credentials + endpoint |
| `/etc/docker/daemon.json` | System admin / `nemoclaw setup-spark` | Docker | cgroup mode |

---

## 10. Dependency Diagram

```
                    +------------------+
                    |    NemoClaw      |
                    |    (this repo)   |
                    +--------+---------+
                             |
                   depends on all three
                  /          |           \
                 v           v            v
  +------------------+ +------------------+ +------------------+
  |    OpenClaw       | |    OpenShell     | |    Docker         |
  |  (AI framework)   | | (security layer) | | (container engine)|
  +------------------+ +--------+---------+ +------------------+
                                |                     ^
                                | built on top of     |
                                +---------------------+

  OpenShell depends on Docker (uses it internally for containers)
  OpenClaw does NOT depend on Docker, OpenShell, or NemoClaw
  Docker does NOT depend on any of the above

  Lowest level:
  +--------------------------------------------------+
  |  Linux Kernel                                     |
  |  (namespaces, cgroups, Landlock, seccomp)         |
  |  Docker and OpenShell both use kernel features    |
  +--------------------------------------------------+
```

### Version Requirements

| Technology | Minimum Version | Where Specified | Why |
|---|---|---|---|
| Docker | 20.10+ | Required by OpenShell | Container runtime |
| OpenShell | 0.1.0 | `blueprint.yaml:2` | Sandbox management |
| OpenClaw | 2026.3.0 | `blueprint.yaml:3` | AI agent framework |
| Node.js | 20.0.0 | `package.json:engines` | NemoClaw + OpenClaw runtime |
| Python | 3.11+ | Blueprint runner | Blueprint orchestration |

---

## 11. Where Each Technology Lives

```
NemoClaw-main/                        <-- This entire repo IS NemoClaw
    |
    +-- bin/nemoclaw.js               <-- NemoClaw's host CLI
    +-- bin/lib/nim.js                <-- NemoClaw talks to Docker (NIM containers)
    +-- bin/lib/onboard.js            <-- NemoClaw talks to Docker (preflight) +
    |                                     OpenShell (gateway, sandbox, inference)
    +-- bin/lib/preflight.js          <-- NemoClaw checks Docker config (cgroups)
    |
    +-- nemoclaw/src/                 <-- NemoClaw's OpenClaw plugin (TypeScript)
    |                                     This is what OpenClaw loads as a plugin
    |
    +-- nemoclaw-blueprint/           <-- NemoClaw's blueprint orchestrator (Python)
    |                                     Calls openshell CLI commands
    |
    +-- Dockerfile                    <-- Docker image recipe for the sandbox
    |                                     Docker builds this into a container image
    |                                     that contains OpenClaw + NemoClaw plugin
    |
    +-- package.json                  <-- Lists openclaw@2026.3.11 as dependency

OpenClaw is NOT in this repo.         <-- Installed via: npm install -g openclaw
OpenShell is NOT in this repo.        <-- Installed from: github.com/NVIDIA/OpenShell
Docker is NOT in this repo.           <-- Installed from: docker.com or apt
```

---

## 12. Common Questions

### "Can I use OpenClaw without Docker, OpenShell, or NemoClaw?"

**Yes.** OpenClaw works standalone on bare Node.js. It just won't have sandboxing,
network policies, or managed inference routing. The agent runs on your host with
full access to everything.

### "Can I use OpenShell without NemoClaw?"

**Yes.** OpenShell can sandbox any application. You'd manually run `openshell`
commands to create sandboxes and apply policies. NemoClaw automates the specific
case of sandboxing OpenClaw.

### "Can I use OpenShell without Docker?"

**No.** OpenShell requires Docker. Its gateway runs as a Docker container
(k3s inside Docker), and sandboxes are Docker containers with additional security
layers applied on top.

### "Can I use NemoClaw without Docker?"

**No.** Docker is a hard requirement. NemoClaw checks for Docker during preflight
(step 1 of onboard). Without Docker, neither OpenShell nor NIM containers can run.

### "Can I use NemoClaw without OpenClaw or OpenShell?"

**No.** NemoClaw is purely an integration layer. Without OpenClaw, there's no agent
to sandbox. Without OpenShell, there's no sandbox runtime. Without Docker, neither
OpenShell nor the sandbox containers can exist.

### "Which technology handles security?"

- **Docker** provides basic isolation (container namespace, filesystem)
- **OpenShell** adds security ON TOP of Docker (Landlock, seccomp, network policies, credential injection)
- **NemoClaw** configures the security policies at setup time
- **OpenClaw** is the thing being secured — it runs inside the sandbox

### "Which technology handles AI inference?"

- **OpenClaw** sends inference requests (it's the AI agent)
- **Docker** hosts the containers that process those requests (sandbox + optional NIM)
- **OpenShell** routes and proxies requests, injecting credentials
- **NemoClaw** configures which provider/model to use, manages NIM containers, stores credentials

### "Where does Docker end and OpenShell begin?"

Docker provides raw containers. OpenShell adds security layers on top:

```
Docker alone:                      Docker + OpenShell:
- Isolated filesystem              - Isolated filesystem
- Isolated network                 - Isolated network
- That's it                        + Landlock path-level FS rules
                                   + seccomp syscall filtering
                                   + Declarative network policies
                                   + Transparent inference proxy
                                   + Credential injection
                                   + Real-time monitoring TUI
                                   + Operator approval queue
```

### "Why does NemoClaw talk to Docker directly AND through OpenShell?"

Two different purposes:

1. **Through OpenShell**: For sandbox management (create, connect, delete sandboxes).
   OpenShell adds security on top of Docker, so NemoClaw uses OpenShell for this.

2. **Directly to Docker**: For NIM containers (local model inference). NIM is a
   standalone Docker container that doesn't need OpenShell's security features —
   it's a trusted service running on the host, not an untrusted agent.

```
NemoClaw --> OpenShell --> Docker    (for sandboxes — needs security)
NemoClaw --> Docker directly         (for NIM containers — trusted service)
```

### "How many Docker containers are running after setup?"

It depends on configuration:

| Configuration | Containers Running |
|---|---|
| Cloud inference (default) | 2: OpenShell gateway + Sandbox |
| Local NIM inference | 3: OpenShell gateway + Sandbox + NIM server |
| Multiple sandboxes | 2 + N: Gateway + N sandboxes (+ optional NIM) |

### "What if Docker isn't installed?"

NemoClaw's preflight check (`bin/lib/onboard.js:55`) detects this and aborts:

```
Docker is not running. Please start Docker and try again.
```

You must install Docker before using NemoClaw.
