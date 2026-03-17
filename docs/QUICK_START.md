# NemoClaw Quick Start Guide

Get from zero to a running sandboxed AI agent in 5 minutes.

---

## Prerequisites Checklist

Before you start, confirm you have:

- [ ] **Node.js 20+** installed (`node --version`)
- [ ] **Docker** installed and running (`docker info`)
- [ ] **NVIDIA API key** from https://build.nvidia.com (starts with `nvapi-`)

---

## Step 1: Install NemoClaw

```bash
npm install -g nemoclaw
```

Verify:
```bash
nemoclaw help
```

---

## Step 2: Run the Setup Wizard

```bash
nemoclaw onboard
```

The wizard guides you through 7 steps:

1. **Preflight** — Checks Docker, OpenShell, GPU
2. **Gateway** — Starts the security proxy
3. **Sandbox** — Creates an isolated container (accept default name `my-assistant`)
4. **NIM** — Choose "NVIDIA Cloud API" (option 1)
5. **Inference** — Enter your NVIDIA API key when prompted
6. **OpenClaw** — Agent auto-configures
7. **Policies** — Accept suggested presets (pypi, npm)

**Time**: ~3-5 minutes on first run (builds Docker image).

---

## Step 3: Connect to Your Sandbox

```bash
nemoclaw my-assistant connect
```

You're now inside the sandbox. Try:
```bash
whoami     # → sandbox
pwd        # → /sandbox
```

---

## Step 4: Talk to the AI Agent

```bash
openclaw tui
```

Type a message and press Enter. Try: "Hello! What can you help me with?"

Press `Ctrl+C` to exit.

---

## Step 5: Monitor Activity

In a **new terminal**:
```bash
openshell term
```

This shows real-time network requests and sandbox activity.

---

## What's Next?

- **[User Guide](USER_GUIDE.md)** — 10 detailed use cases + full feature walkthrough
- **[CLI Reference](CLI_REFERENCE.md)** — Every command and option
- **[Architecture](ARCHITECTURE.md)** — How the system works under the hood

---

## Quick Reference

| Task | Command |
|---|---|
| Connect to sandbox | `nemoclaw my-assistant connect` |
| Check status | `nemoclaw my-assistant status` |
| View logs | `nemoclaw my-assistant logs --follow` |
| List sandboxes | `nemoclaw list` |
| Add network access | `nemoclaw my-assistant policy-add` |
| Monitor activity | `openshell term` |
| Delete sandbox | `nemoclaw my-assistant destroy` |
