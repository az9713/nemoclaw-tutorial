# NemoClaw User Guide

A complete, step-by-step guide to using NemoClaw. This guide assumes no prior experience with Docker, sandboxing, or AI agent frameworks. By the end, you will be able to set up, run, monitor, and manage sandboxed AI agents with full network policy control.

---

## Table of Contents

1. [What is NemoClaw?](#1-what-is-nemoclaw)
2. [How It Works (The Big Picture)](#2-how-it-works-the-big-picture)
3. [Prerequisites](#3-prerequisites)
4. [Installation](#4-installation)
5. [Your First Sandbox (Onboarding)](#5-your-first-sandbox-onboarding)
6. [10 Quick-Win Use Cases](#6-10-quick-win-use-cases)
7. [Managing Sandboxes](#7-managing-sandboxes)
8. [Network Policies](#8-network-policies)
9. [Inference Configuration](#9-inference-configuration)
10. [Migration (Moving Existing OpenClaw)](#10-migration-moving-existing-openclaw)
11. [Deployment to Remote GPUs](#11-deployment-to-remote-gpus)
12. [Monitoring and Logs](#12-monitoring-and-logs)
13. [Troubleshooting](#13-troubleshooting)
14. [Glossary](#14-glossary)

---

## 1. What is NemoClaw?

### The Problem

AI agents (like OpenClaw) are powerful — they can write code, call APIs, manage files, and automate tasks. But this power is also a risk:

- An agent could access sensitive files on your computer
- An agent could call unauthorized APIs and leak data
- An agent could install malicious packages
- An agent could make expensive API calls you didn't authorize

### The Solution

NemoClaw puts your AI agent in a **sandbox** — an isolated environment where:

- **Network access is controlled**: The agent can only reach endpoints you explicitly allow
- **File access is restricted**: The agent can only read/write in designated directories
- **Inference is routed safely**: AI model calls go through a controlled proxy with injected credentials
- **Everything is monitored**: You can see every network request the agent makes

Think of it like running your AI agent in a secure room with cameras, where you control which doors are open.

### Key Terminology

| Term | What it means |
|---|---|
| **Sandbox** | An isolated container where the AI agent runs |
| **OpenClaw** | The AI agent framework (the "brain" of the agent) |
| **OpenShell** | The secure runtime that creates and manages sandboxes |
| **Gateway** | The network proxy that controls what the sandbox can access |
| **Policy** | Rules that define what the sandbox is allowed to do |
| **Inference** | When the AI agent calls a language model to generate responses |
| **NIM** | NVIDIA Inference Microservice — a local model server |
| **Blueprint** | A versioned recipe that defines how to set up a sandbox |

---

## 2. How It Works (The Big Picture)

### The Four Technologies

NemoClaw is the bridge between **four technologies**:

```
+==================+  +==================+  +==================+  +==================+
|    OpenClaw      |  |    Docker        |  |    OpenShell     |  |    NemoClaw      |
|                  |  |                  |  |                  |  |                  |
|  The AI Agent    |  |  The Container   |  |  The Security    |  |  The Installer   |
|  "The brain"     |  |  Engine          |  |  Runtime         |  |  & Glue          |
|                  |  |  "The building   |  |  "The security   |  |  "The            |
|  Does the AI     |  |   material"      |  |   system"        |  |   contractor"    |
|  work:           |  |                  |  |                  |  |                  |
|  - Conversations |  |  Provides the    |  |  Adds security   |  |  Connects them:  |
|  - Code writing  |  |  containers      |  |  ON TOP of       |  |  - Builds images |
|  - Tool calling  |  |  everything      |  |  Docker:         |  |  - Creates       |
|  - Task automate |  |  runs inside:    |  |  - Policies      |  |    sandboxes     |
|                  |  |  - Gateway       |  |  - Credential    |  |  - Configures    |
|                  |  |  - Sandbox       |  |    injection     |  |    inference     |
|                  |  |  - NIM server    |  |  - Monitoring    |  |  - Manages NIM   |
+==================+  +==================+  +==================+  +==================+
```

**Think of it this way**:
- **OpenClaw** = A very capable robot that can do anything
- **Docker** = Building materials (walls, doors, windows) — you can build rooms from them, but they have no locks
- **OpenShell** = A security company that uses the building materials (Docker) to build high-security rooms with locks, cameras, and alarms
- **NemoClaw** = The contractor who draws blueprints, tells the security company what to build, installs the robot, and configures the locks

**Can they be used independently?**
- OpenClaw works without the others (but unsandboxed — risky)
- Docker works without the others (it's a general-purpose container engine)
- OpenShell works without NemoClaw/OpenClaw (but requires Docker; can sandbox any app)
- NemoClaw REQUIRES Docker, OpenShell, AND OpenClaw (it's the integration layer)

### The Complete Picture

```
YOU (the user)
  |
  | Type commands
  v
+-------------------+
| nemoclaw CLI      |   Your interface to the system
| (on your machine) |   (NemoClaw)
+--------+----------+
         |
         | Calls openshell CLI + docker CLI
         v
+=======================DOCKER ENGINE============================+
|                                                                 |
|  +-------------------+       +-------------------+              |
|  | Container 1:      |       | Container 2:      |              |
|  | OpenShell Gateway |------>| Sandbox           |              |
|  | (security proxy)  |       | +---------------+ |              |
|  | (k3s inside)      |       | | OpenClaw Agent| |              |
|  +-------------------+       | | (sandboxed)   | |              |
|                               | +---------------+ |              |
|  +-------------------+       +-------------------+              |
|  | Container 3:      |  (only if local inference)               |
|  | NIM Model Server  |                                          |
|  | (GPU: --gpus all) |                                          |
|  +-------------------+           NVIDIA Cloud API               |
|                                  (if cloud inference)           |
+=================================================================+
```

**The flow**:
1. You type a command (e.g., `nemoclaw my-assistant connect`)
2. **NemoClaw** tells **OpenShell** what to do (via `openshell` CLI commands)
3. **OpenShell** uses **Docker** to manage the sandbox container and security policies
4. **OpenClaw** (the AI agent) runs inside a **Docker** container (the sandbox), isolated from your host
5. When **OpenClaw** needs to call an AI model, the request goes through **OpenShell's** gateway (another **Docker** container)
6. **OpenShell** injects your API credentials (configured by **NemoClaw**, never seen by **OpenClaw**)
7. The request reaches the **NVIDIA Cloud API** or a local **NIM Docker container**
8. The response flows back through the gateway to **OpenClaw**

### Who Does What at Each Phase

| Phase | NemoClaw | Docker | OpenShell | OpenClaw |
|---|---|---|---|---|
| **Setup** | Runs wizard, builds images, configures everything | Builds images, starts containers | Creates sandboxes with security layers | Gets installed inside sandbox |
| **Runtime** | Mostly idle (status queries) | Hosts all containers silently | Enforces security, proxies network | Runs AI tasks, conversations |
| **Teardown** | Calls destroy/eject commands | Stops and removes containers | Cleans up security config | Stops running |

For the full deep-dive, see [How NemoClaw, OpenShell, Docker, and OpenClaw Work Together](HOW_THE_THREE_PROJECTS_WORK_TOGETHER.md).

---

## 3. Prerequisites

Before installing NemoClaw, you need:

### 3.1 Node.js (v20 or higher)

Node.js runs NemoClaw's CLI tool.

**Check if installed**:
```bash
node --version
```

If not installed:
- **Linux**: `curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - && sudo apt install -y nodejs`
- **macOS**: `brew install node@22` or download from https://nodejs.org
- **Windows**: Download from https://nodejs.org (choose the LTS version)

### 3.2 Docker

Docker runs the sandbox containers.

**Check if installed**:
```bash
docker --version
docker info    # Should not show errors
```

If not installed:
- **Linux**: `sudo apt install docker.io && sudo usermod -aG docker $USER` (then log out and back in)
- **macOS**: Install Docker Desktop from https://docker.com
- **Windows**: Install Docker Desktop with WSL2 backend from https://docker.com

**Important**: Docker must be **running** (not just installed). On macOS/Windows, open Docker Desktop. On Linux, run `sudo systemctl start docker`.

### 3.3 NVIDIA API Key

You need an API key to use NVIDIA's cloud models (free tier available).

1. Go to https://build.nvidia.com
2. Create an account (or sign in)
3. Navigate to API Keys
4. Create a new key — it starts with `nvapi-`
5. Copy the key and keep it safe

### 3.4 (Optional) GPU

- **Without GPU**: NemoClaw uses NVIDIA's cloud API for inference (works on any machine)
- **With NVIDIA GPU**: You can run models locally using NIM (faster, no API costs)
- **With Apple Silicon**: Cloud inference only (NIM requires NVIDIA GPUs)

---

## 4. Installation

### Option A: One-Line Installer (Recommended)

```bash
curl -fsSL https://nvidia.com/nemoclaw.sh | bash
```

This installs Node.js (if needed), NemoClaw, and runs the setup wizard.

### Option B: Manual Install

```bash
# Install NemoClaw globally
npm install -g nemoclaw

# Verify installation
nemoclaw --help
```

You should see:
```
  nemoclaw — NemoClaw CLI

  Getting Started:
    nemoclaw onboard                 Interactive setup wizard (recommended)
    ...
```

---

## 5. Your First Sandbox (Onboarding)

The onboard wizard creates everything you need in 7 steps.

### Run the Wizard

```bash
nemoclaw onboard
```

### Step-by-Step Walkthrough

**Step 1/7: Preflight checks**

The wizard checks your system:
```
  [1/7] Preflight checks
  ──────────────────────────────────────────────────
  ✓ Docker is running
  ✓ openshell CLI: 0.1.0
  ✓ cgroup configuration OK
  ⓘ No GPU detected — will use cloud inference
```

If anything fails, the wizard tells you exactly what to fix.

**Step 2/7: Starting OpenShell gateway**

```
  [2/7] Starting OpenShell gateway
  ──────────────────────────────────────────────────
  ✓ Gateway is healthy
```

This starts the security proxy that manages your sandbox.

**Step 3/7: Creating sandbox**

```
  [3/7] Creating sandbox
  ──────────────────────────────────────────────────
  Sandbox name [my-assistant]:
```

Press Enter to accept the default name, or type your own. The wizard then builds the sandbox container (this takes 2-5 minutes on first run).

**Step 4/7: Configuring inference**

```
  [4/7] Configuring inference (NIM)
  ──────────────────────────────────────────────────

  Inference options:
    1) NVIDIA Cloud API (build.nvidia.com)

  Choose [1]:
```

Press Enter for cloud inference. You'll be asked for your NVIDIA API key:
```
  NVIDIA API Key (nvapi-...):
```

Paste your key and press Enter.

**Step 5/7: Setting up inference provider**

```
  [5/7] Setting up inference provider
  ──────────────────────────────────────────────────
  ✓ Inference route set: nvidia-nim / nvidia/nemotron-3-super-120b-a12b
```

**Step 6/7: OpenClaw setup**

```
  [6/7] Setting up OpenClaw inside sandbox
  ──────────────────────────────────────────────────
  ✓ OpenClaw gateway launched inside sandbox
```

**Step 7/7: Policy presets**

```
  [7/7] Policy presets
  ──────────────────────────────────────────────────

  Available policy presets:
    ○ slack — Allow Slack API access
    ○ discord — Allow Discord API access
    ○ jira — Allow Jira API access
    ○ telegram — Allow Telegram Bot API access
    ○ pypi — Allow PyPI package downloads (suggested)
    ○ npm — Allow npm package downloads (suggested)
    ○ huggingface — Allow HuggingFace model downloads
    ○ outlook — Allow Microsoft Outlook/Graph API access
    ○ docker — Allow Docker Hub access

  Apply suggested presets (pypi, npm)? [Y/n/list]:
```

Press Enter to apply the suggested presets. Type `n` to skip, or `list` to pick specific ones.

### Dashboard

After setup completes, you see:
```
  ──────────────────────────────────────────────────
  Sandbox      my-assistant (Landlock + seccomp + netns)
  Model        nvidia/nemotron-3-super-120b-a12b (NVIDIA Cloud API)
  NIM          not running
  ──────────────────────────────────────────────────
  Run:         nemoclaw my-assistant connect
  Status:      nemoclaw my-assistant status
  Logs:        nemoclaw my-assistant logs --follow
  ──────────────────────────────────────────────────
```

---

## 6. 10 Quick-Win Use Cases

These use cases give you quick, hands-on experience with NemoClaw. Each builds on the previous one.

### Use Case 1: Connect to Your Sandbox

**Goal**: Enter your sandbox and see that it's a real, isolated environment.

```bash
nemoclaw my-assistant connect
```

You're now inside the sandbox. Try:
```bash
whoami                    # Shows "sandbox" (not your username)
pwd                       # Shows /sandbox (not your home directory)
ls /                      # Shows the sandbox filesystem
echo "Hello from sandbox" # Confirm you can run commands
exit                      # Return to your host
```

**What you learned**: The sandbox is an isolated environment with its own filesystem and user.

### Use Case 2: Chat with the AI Agent

**Goal**: Send a message to the AI agent and get a response.

```bash
nemoclaw my-assistant connect
```

Inside the sandbox:
```bash
openclaw agent --agent main --local -m "What is 2+2?" --session-id test1
```

You should see the agent respond with "4" (plus some reasoning).

**What you learned**: The AI agent runs inside the sandbox and can respond to your queries.

### Use Case 3: Use the Interactive Chat (TUI)

**Goal**: Have a back-and-forth conversation with the agent.

```bash
nemoclaw my-assistant connect
```

Inside the sandbox:
```bash
openclaw tui
```

This opens an interactive chat interface. Type messages and press Enter. Try:
- "Hello, who are you?"
- "Write a Python function that reverses a string"
- "Explain what Docker is in 3 sentences"

Press `Ctrl+C` to exit the TUI.

**What you learned**: The TUI provides a rich, interactive way to work with the agent.

### Use Case 4: Check Sandbox Status

**Goal**: See the health of your sandbox, inference, and NIM.

```bash
nemoclaw my-assistant status
```

Output:
```
  Sandbox: my-assistant
    Model:    nvidia/nemotron-3-super-120b-a12b
    Provider: nvidia-nim
    GPU:      no
    Policies: pypi, npm
    NIM:      not running
    Healthy:  yes
```

**What you learned**: You can monitor your sandbox without entering it.

### Use Case 5: View Sandbox Logs

**Goal**: See what's happening inside your sandbox in real time.

```bash
# View recent logs
nemoclaw my-assistant logs

# Stream logs in real time (like "tail -f")
nemoclaw my-assistant logs --follow
```

Press `Ctrl+C` to stop streaming.

**What you learned**: Logs help you understand what the agent is doing and diagnose issues.

### Use Case 6: List All Your Sandboxes

**Goal**: See all sandboxes you've created.

```bash
nemoclaw list
```

Output:
```
  Sandboxes:
    my-assistant *
      model: nvidia/nemotron-3-super-120b-a12b  provider: nvidia-nim  CPU  policies: pypi, npm

  * = default sandbox
```

**What you learned**: You can have multiple sandboxes for different purposes.

### Use Case 7: View Network Policies

**Goal**: See which network endpoints your sandbox is allowed to reach.

```bash
nemoclaw my-assistant policy-list
```

Output:
```
  Policy presets for sandbox 'my-assistant':
    ○ slack — Allow Slack API access
    ○ discord — Allow Discord API access
    ○ jira — Allow Jira API access
    ○ telegram — Allow Telegram Bot API access
    ● pypi — Allow PyPI package downloads
    ● npm — Allow npm package downloads
    ○ huggingface — Allow HuggingFace model downloads
    ○ outlook — Allow Microsoft Outlook/Graph API access
    ○ docker — Allow Docker Hub access
```

`●` = applied, `○` = not applied.

**What you learned**: You control exactly which external services the agent can access.

### Use Case 8: Add a Policy Preset

**Goal**: Allow your agent to access GitHub.

```bash
nemoclaw my-assistant policy-add
```

You'll see the preset list. Type `github` (if available) or one of the listed presets:
```
  Preset to apply: slack
  Apply 'slack' to sandbox 'my-assistant'? [Y/n]: Y
```

Verify:
```bash
nemoclaw my-assistant policy-list
```

The preset should now show `●`.

**What you learned**: You can dynamically grant network access to your sandbox.

### Use Case 9: Install a Python Package Inside the Sandbox

**Goal**: Verify that the PyPI policy preset works by installing a package.

```bash
nemoclaw my-assistant connect
```

Inside the sandbox:
```bash
pip3 install requests
python3 -c "import requests; print('requests installed successfully')"
exit
```

This works because the `pypi` policy preset allows access to `pypi.org`.

**What you learned**: Policy presets control real network access — pip works because you allowed PyPI.

### Use Case 10: Monitor Network Activity with OpenShell TUI

**Goal**: See every network request the sandbox makes in real time.

In a separate terminal:
```bash
openshell term
```

This opens the OpenShell Terminal UI, showing:
- Active sandboxes
- Network requests (allowed and blocked)
- Inference routing status

Now, in another terminal, connect to the sandbox and make a request:
```bash
nemoclaw my-assistant connect
# Inside sandbox:
curl https://api.github.com
```

Watch the OpenShell TUI — you'll see the request appear (and whether it was allowed or blocked).

**What you learned**: OpenShell gives you real-time visibility into everything the sandbox does.

---

## 7. Managing Sandboxes

### Creating a New Sandbox

The easiest way is the onboard wizard:
```bash
nemoclaw onboard
```

### Connecting to a Sandbox

```bash
nemoclaw my-assistant connect
```

This opens an interactive shell inside the sandbox. You're now "inside" the isolated environment.

### Checking Sandbox Status

```bash
nemoclaw my-assistant status
```

Shows: sandbox name, model, provider, GPU status, applied policies, NIM health.

### Viewing Logs

```bash
# Recent logs
nemoclaw my-assistant logs

# Stream in real time
nemoclaw my-assistant logs --follow
```

### Destroying a Sandbox

```bash
nemoclaw my-assistant destroy
```

This stops any NIM container and deletes the sandbox. **This is irreversible** — all data inside the sandbox is lost.

### Listing All Sandboxes

```bash
nemoclaw list
```

---

## 8. Network Policies

### How Policies Work

By default, the sandbox can only reach a small set of essential endpoints (NVIDIA API, GitHub, npm registry, Telegram). Everything else is blocked.

**Policy presets** add access to specific services. Each preset is a YAML file that declares which hosts, ports, and HTTP methods are allowed.

### Available Presets

| Preset | What it allows |
|---|---|
| `slack` | Slack API, hooks, webhooks |
| `discord` | Discord API, CDN |
| `jira` | Atlassian/Jira API |
| `telegram` | Telegram Bot API |
| `pypi` | Python Package Index (pip install) |
| `npm` | Node Package Registry (npm install) |
| `huggingface` | HuggingFace model downloads |
| `outlook` | Microsoft Graph API |
| `docker` | Docker Hub registry |

### Adding a Preset

```bash
nemoclaw my-assistant policy-add
```

Follow the interactive prompt.

### Listing Applied Presets

```bash
nemoclaw my-assistant policy-list
```

### What Happens When a Request is Blocked?

1. The agent tries to reach an unlisted host
2. OpenShell blocks the request
3. The blocked request appears in the OpenShell TUI (`openshell term`)
4. You (the operator) can approve it once or add it permanently

---

## 9. Inference Configuration

### What is Inference?

"Inference" is when the AI agent sends a prompt to a language model and gets a response. NemoClaw supports several inference providers:

| Provider | Where it runs | API Key Needed | GPU Needed |
|---|---|---|---|
| NVIDIA Cloud | NVIDIA's servers | Yes (NVIDIA_API_KEY) | No |
| Local NIM | Your machine (Docker) | Optional | Yes (NVIDIA GPU) |
| Local vLLM | Your machine | No | Yes |
| Ollama | Your machine | No | Varies |

### Changing Inference Provider

Run the plugin onboard command:
```bash
openclaw nemoclaw onboard --endpoint build --model nvidia/nemotron-3-super-120b-a12b
```

Or interactively:
```bash
openclaw nemoclaw onboard
```

### Supported Models

| Model | Size | Good For |
|---|---|---|
| `nvidia/nemotron-3-super-120b-a12b` | 120B params | General use (default) |
| `nvidia/llama-3.1-nemotron-ultra-253b-v1` | 253B params | Complex reasoning |
| `nvidia/llama-3.3-nemotron-super-49b-v1.5` | 49B params | Balanced performance |
| `nvidia/nemotron-3-nano-30b-a3b` | 30B params | Fast, lightweight |

---

## 10. Migration (Moving Existing OpenClaw)

If you already have OpenClaw installed on your host, NemoClaw can migrate it into a sandbox without losing your configuration, agents, workspaces, or extensions.

### Dry Run (See What Would Happen)

```bash
openclaw nemoclaw migrate --dry-run
```

This shows what would be migrated without making any changes.

### Perform Migration

```bash
openclaw nemoclaw migrate
```

This:
1. Detects your host OpenClaw installation
2. Creates a snapshot (backup) of everything
3. Creates a new sandbox
4. Copies your config, agents, and workspaces into the sandbox
5. Verifies everything works
6. Keeps your host installation untouched

### Rolling Back

If something goes wrong:
```bash
openclaw nemoclaw eject --confirm
```

This restores your host installation from the snapshot.

---

## 11. Deployment to Remote GPUs

NemoClaw can deploy to a remote GPU instance (via Brev).

### Prerequisites

- Brev CLI installed: https://brev.nvidia.com
- NVIDIA API key set

### Deploy

```bash
nemoclaw deploy my-gpu-box
```

This:
1. Creates a Brev GPU instance
2. Syncs NemoClaw code to the instance
3. Runs setup on the instance
4. Connects you to the sandbox

---

## 12. Monitoring and Logs

### View Sandbox Logs

```bash
nemoclaw my-assistant logs           # Recent logs
nemoclaw my-assistant logs --follow  # Stream in real time
```

### OpenShell TUI

The OpenShell TUI provides real-time monitoring:

```bash
openshell term
```

Shows:
- Active sandboxes and their status
- Network requests (allowed/blocked)
- Inference routing
- Operator approval queue

### Check Overall Status

```bash
nemoclaw status                  # All sandboxes + services
nemoclaw my-assistant status     # Specific sandbox
```

---

## 13. Troubleshooting

### "Docker is not running"

**Fix**: Start Docker.
- **macOS/Windows**: Open Docker Desktop
- **Linux**: `sudo systemctl start docker`

### "openshell CLI not found"

**Fix**: Install OpenShell.
```bash
# NemoClaw will try to install it automatically, or:
# Visit https://github.com/NVIDIA/OpenShell/releases
```

### "Gateway failed to start"

**Fix**: Destroy the old gateway and retry.
```bash
openshell gateway destroy -g nemoclaw
nemoclaw onboard
```

### "cgroup v2 detected but Docker is not configured"

**Fix**: Run the DGX Spark setup:
```bash
nemoclaw setup-spark
```

This configures Docker for cgroup v2 compatibility.

### "NVIDIA API key invalid"

**Fix**: Get a new key from https://build.nvidia.com. The key should start with `nvapi-`.

### "Sandbox already exists"

**Fix**: Either destroy the old one or choose a different name:
```bash
nemoclaw my-assistant destroy
nemoclaw onboard
```

### "Blueprint verification failed"

**Fix**: Clear the blueprint cache:
```bash
rm -rf ~/.nemoclaw/blueprints/
```

### "NIM failed to start"

**Fix**: Check GPU and Docker:
```bash
docker info | grep GPU
nvidia-smi         # Should show your GPU
```

NIM requires NVIDIA GPUs. Use cloud inference if you don't have one.

### "Connection refused" when connecting to sandbox

**Fix**: Port forwarding may have stopped. Restart it:
```bash
openshell forward start --background 18789 my-assistant
nemoclaw my-assistant connect
```

---

## 14. Glossary

| Term | Definition |
|---|---|
| **Agent** | An AI system that can take actions autonomously (not just chat) |
| **Blueprint** | A versioned recipe that defines how to create a sandbox |
| **Container** | A lightweight, isolated environment (like a mini virtual machine) |
| **Credential injection** | Technique where API keys are added by the proxy, not seen by the agent |
| **Docker** | Software that creates and manages containers |
| **Gateway** | The OpenShell security proxy that controls sandbox access |
| **GPU** | Graphics Processing Unit — hardware accelerator for AI model inference |
| **Inference** | The process of running input through an AI model to get output |
| **k3s** | Lightweight Kubernetes, used inside OpenShell's gateway |
| **Landlock** | Linux kernel feature for filesystem sandboxing |
| **NCP** | NVIDIA Cloud Partner — enterprise inference endpoint |
| **Nemotron** | NVIDIA's family of AI models |
| **NIM** | NVIDIA Inference Microservice — runs models locally in containers |
| **Node.js** | JavaScript runtime for server-side applications |
| **npm** | Node Package Manager — installs JavaScript libraries |
| **OCI** | Open Container Initiative — standard for container images |
| **Ollama** | Open-source tool for running AI models locally |
| **OpenClaw** | AI agent framework that NemoClaw sandboxes |
| **OpenShell** | NVIDIA's secure runtime for autonomous agents |
| **Policy** | Rules that define what a sandbox can access |
| **Preset** | A pre-built set of network policy rules for a specific service |
| **Proxy** | An intermediary that forwards and controls network requests |
| **Sandbox** | An isolated environment where the agent runs |
| **seccomp** | Linux kernel feature for syscall filtering |
| **Snapshot** | A backup of your configuration used for migration rollback |
| **TUI** | Terminal User Interface — interactive text-based UI |
| **vLLM** | Open-source library for fast AI model serving |
| **VRAM** | Video RAM — GPU memory used by AI models |
| **YAML** | A human-readable configuration file format |
