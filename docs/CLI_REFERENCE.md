# NemoClaw CLI Reference

Complete reference for all NemoClaw commands, options, and their behavior.

---

## Two CLIs

NemoClaw has two command-line interfaces:

1. **Host CLI** (`nemoclaw`) — Runs on your machine, manages sandboxes from the outside
2. **Plugin CLI** (`openclaw nemoclaw`) — Runs inside the OpenClaw process, manages blueprint lifecycle

---

## Host CLI: `nemoclaw`

### `nemoclaw onboard`

Interactive 7-step setup wizard. Creates a sandbox from scratch.

```
nemoclaw onboard
```

**Steps**: Preflight → Gateway → Sandbox → NIM → Inference → OpenClaw → Policies

**No options** — everything is prompted interactively.

---

### `nemoclaw list`

List all registered sandboxes with their configuration.

```
nemoclaw list
```

**Output**: Sandbox name, model, provider, GPU status, applied policies. `*` marks the default sandbox.

---

### `nemoclaw deploy <instance-name>`

Deploy NemoClaw to a remote GPU instance via Brev.

```
nemoclaw deploy my-gpu-box
```

**Arguments**:

| Argument | Required | Description |
|---|---|---|
| `instance-name` | Yes | Name for the Brev VM instance |

**Environment Variables**:

| Variable | Purpose |
|---|---|
| `NVIDIA_API_KEY` | Required. NVIDIA cloud inference key |
| `GITHUB_TOKEN` | Optional. For private repo access |
| `TELEGRAM_BOT_TOKEN` | Optional. Enables Telegram bridge service |
| `NEMOCLAW_GPU` | Optional. GPU type (default: `a2-highgpu-1g:nvidia-tesla-a100:1`) |

**What it does**:
1. Creates or reuses Brev VM instance
2. Waits for SSH connectivity
3. Syncs NemoClaw code via rsync
4. Uploads credentials
5. Runs setup on VM
6. Starts services (if Telegram token present)
7. Connects to sandbox via SSH

---

### `nemoclaw start`

Start background services (Telegram bridge, tunnels).

```
nemoclaw start
```

Requires `NVIDIA_API_KEY` in environment.

---

### `nemoclaw stop`

Stop all background services.

```
nemoclaw stop
```

---

### `nemoclaw status`

Show all registered sandboxes and service status.

```
nemoclaw status
```

---

### `nemoclaw setup`

**(Deprecated)** Legacy setup script. Use `nemoclaw onboard` instead.

```
nemoclaw setup
```

---

### `nemoclaw setup-spark`

Fix Docker cgroup v2 configuration for DGX Spark / GB10 systems.

```
nemoclaw setup-spark
```

Adds `"default-cgroupns-mode": "host"` to `/etc/docker/daemon.json` and restarts Docker.

---

## Sandbox-Scoped Commands

These commands operate on a specific sandbox by name.

### `nemoclaw <name> connect`

Open an interactive shell inside the sandbox.

```
nemoclaw my-assistant connect
```

Starts port forwarding on port 18789 before connecting.

---

### `nemoclaw <name> status`

Show detailed status of a specific sandbox.

```
nemoclaw my-assistant status
```

**Shows**: Model, provider, GPU, policies, NIM container status, NIM health.

---

### `nemoclaw <name> logs [--follow]`

View sandbox logs.

```
nemoclaw my-assistant logs
nemoclaw my-assistant logs --follow
```

| Option | Description |
|---|---|
| `--follow` | Stream logs in real time (like `tail -f`) |

---

### `nemoclaw <name> policy-add`

Interactively add a network policy preset to the sandbox.

```
nemoclaw my-assistant policy-add
```

Shows available presets, prompts for selection, confirms before applying.

---

### `nemoclaw <name> policy-list`

List all policy presets with applied/available status.

```
nemoclaw my-assistant policy-list
```

`●` = applied, `○` = not applied.

---

### `nemoclaw <name> destroy`

Stop NIM container and delete the sandbox.

```
nemoclaw my-assistant destroy
```

**Warning**: This is irreversible. All data inside the sandbox is lost.

---

## Plugin CLI: `openclaw nemoclaw`

These commands run inside the OpenClaw process and manage the blueprint lifecycle.

### `openclaw nemoclaw launch`

Bootstrap OpenClaw inside an OpenShell sandbox from scratch.

```
openclaw nemoclaw launch [options]
```

| Option | Default | Description |
|---|---|---|
| `--force` | `false` | Skip warnings and force plugin-driven bootstrap |
| `--profile <profile>` | `default` | Blueprint profile to use |

**Behavior**:
- Without `--force`: warns if host OpenClaw exists (suggests migrate) or doesn't exist (suggests native OpenShell)
- With `--force`: proceeds regardless

---

### `openclaw nemoclaw migrate`

Migrate host OpenClaw installation into an OpenShell sandbox.

```
openclaw nemoclaw migrate [options]
```

| Option | Default | Description |
|---|---|---|
| `--dry-run` | `false` | Show what would be migrated without making changes |
| `--profile <profile>` | `default` | Blueprint profile to use |
| `--skip-backup` | `false` | Skip creating a host backup snapshot |

**Dry run output**: Lists what would be snapshotted, archived, and copied.

**Full run**: Creates snapshot → builds sandbox → copies archives → verifies → saves state.

---

### `openclaw nemoclaw connect`

Open an interactive shell inside the OpenClaw sandbox.

```
openclaw nemoclaw connect [options]
```

| Option | Default | Description |
|---|---|---|
| `--sandbox <name>` | `openclaw` | Sandbox name to connect to |

---

### `openclaw nemoclaw status`

Show sandbox, blueprint, and inference state.

```
openclaw nemoclaw status [options]
```

| Option | Default | Description |
|---|---|---|
| `--json` | `false` | Output as JSON |

**Shows**: Last action, blueprint version, run ID, sandbox status (running/uptime), inference config (provider/model/endpoint), rollback snapshot path.

---

### `openclaw nemoclaw logs`

Stream blueprint execution and sandbox logs.

```
openclaw nemoclaw logs [options]
```

| Option | Default | Description |
|---|---|---|
| `-f, --follow` | `false` | Follow log output |
| `-n, --lines <count>` | `50` | Number of lines to show |
| `--run-id <id>` | | Show logs for a specific blueprint run |

---

### `openclaw nemoclaw eject`

Rollback from OpenShell and restore host installation.

```
openclaw nemoclaw eject [options]
```

| Option | Default | Description |
|---|---|---|
| `--run-id <id>` | | Specific blueprint run ID to rollback from |
| `--confirm` | `false` | Skip confirmation prompt |

**Without `--confirm`**: Shows what will happen and asks you to re-run with `--confirm`.

**With `--confirm`**: Stops sandbox, rollbacks blueprint, restores host state, clears NemoClaw state.

---

### `openclaw nemoclaw onboard`

Interactive or scripted inference configuration.

```
openclaw nemoclaw onboard [options]
```

| Option | Default | Description |
|---|---|---|
| `--api-key <key>` | | API key (skips prompt) |
| `--endpoint <type>` | | Endpoint type: `build`, `ncp`, `nim-local`, `vllm`, `ollama`, `custom` |
| `--ncp-partner <name>` | | NCP partner name (when endpoint is `ncp`) |
| `--endpoint-url <url>` | | Endpoint URL (for `ncp`, `nim-local`, `ollama`, or `custom`) |
| `--model <model>` | | Model ID to use |

---

## Slash Command: `/nemoclaw`

Available inside OpenClaw chat:

```
/nemoclaw status    # Show current state
/nemoclaw help      # Show available subcommands
```

---

## Environment Variables

| Variable | Purpose | Used By |
|---|---|---|
| `NVIDIA_API_KEY` | NVIDIA cloud inference API key | onboard, deploy, inference |
| `NIM_API_KEY` | Local NIM authentication | nim-local profile |
| `OPENAI_API_KEY` | vLLM/Ollama authentication | vllm profile |
| `GITHUB_TOKEN` | GitHub API access (private repos) | deploy |
| `TELEGRAM_BOT_TOKEN` | Telegram bridge | deploy, services |
| `NEMOCLAW_EXPERIMENTAL` | Enable beta features (`"1"` to enable) | onboard (local inference) |
| `NEMOCLAW_GPU` | GPU type for Brev deploy | deploy |
| `DOCKER_HOST` | Docker socket path (auto-detected for Colima) | onboard |
| `CHAT_UI_URL` | Dashboard URL | sandbox creation |

---

## Exit Codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Error (check stderr for details) |

---

## Configuration Files

| File | Purpose | Created By |
|---|---|---|
| `~/.nemoclaw/credentials.json` | Stored API keys (mode 600) | onboard, deploy |
| `~/.nemoclaw/config.json` | Inference configuration | plugin onboard |
| `~/.nemoclaw/sandboxes.json` | Sandbox registry | onboard, create |
| `~/.nemoclaw/state/nemoclaw.json` | Plugin run state | launch, migrate |
| `~/.nemoclaw/blueprints/<ver>/` | Cached blueprints | resolve |
| `~/.nemoclaw/snapshots/<ts>/` | Migration snapshots | migrate |
