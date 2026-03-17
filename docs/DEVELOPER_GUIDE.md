# NemoClaw Developer Guide

A comprehensive, step-by-step guide for developers who want to understand, modify, and extend NemoClaw. This guide assumes you have experience with C/C++/Java but are new to full-stack web development, Node.js, TypeScript, and Python ecosystem tooling.

---

## Table of Contents

1. [Prerequisite Concepts](#1-prerequisite-concepts)
2. [Setting Up Your Development Environment](#2-setting-up-your-development-environment)
3. [Understanding the Codebase](#3-understanding-the-codebase)
4. [How to Build the Project](#4-how-to-build-the-project)
5. [How to Run Tests](#5-how-to-run-tests)
6. [How to Add a New CLI Command](#6-how-to-add-a-new-cli-command)
7. [How to Add a New Policy Preset](#7-how-to-add-a-new-policy-preset)
8. [How to Add a New Inference Provider](#8-how-to-add-a-new-inference-provider)
9. [How to Modify the Blueprint](#9-how-to-modify-the-blueprint)
10. [Debugging Guide](#10-debugging-guide)
11. [Code Conventions](#11-code-conventions)
12. [Common Gotchas](#12-common-gotchas)

---

## 1. Prerequisite Concepts

### What You Need to Know (Quick Primer)

If you come from C/C++/Java, here are the key differences in this project's technology stack:

#### Node.js

**What it is**: A runtime that executes JavaScript outside the browser. Think of it like the JVM for Java, but for JavaScript.

**Key differences from C/Java**:
- Single-threaded with an event loop (no threading for I/O)
- `async`/`await` replaces threads for concurrent I/O
- No compilation step for plain JavaScript files
- `require()` loads modules (like `#include` or `import`)

**Package management**: `npm` (Node Package Manager) is like Maven/Gradle for Java.
- `package.json` = `pom.xml` / `build.gradle` — declares dependencies and build scripts
- `node_modules/` = where dependencies are installed (like `.m2` in Maven)
- `npm install` = download all dependencies
- `npm run <script>` = run a script defined in package.json

#### TypeScript

**What it is**: JavaScript with static type annotations. Think of it as "Java-like types on top of JavaScript."

**Key concepts**:
- `.ts` files must be compiled to `.js` files before running (like `.java` → `.class`)
- `tsc` (TypeScript Compiler) is like `javac`
- `tsconfig.json` controls compilation options (like compiler flags)
- `dist/` folder contains compiled output (like `target/` in Maven)
- Types are erased at runtime — they only exist during development/compilation

**Example comparison**:
```java
// Java
public interface Config {
    String getName();
    int getPort();
}
```
```typescript
// TypeScript
interface Config {
    name: string;
    port: number;
}
```

#### Module Systems: CommonJS vs ESM

NemoClaw uses **two different module systems** (this is a common source of confusion):

| Feature | CommonJS (CJS) | ECMAScript Modules (ESM) |
|---|---|---|
| Used in | `bin/` (Host CLI) | `nemoclaw/src/` (Plugin) |
| Import | `const x = require('./x')` | `import x from './x.js'` |
| Export | `module.exports = { ... }` | `export function ...` |
| Analogy | Like C's `#include` | Like Java's `import` |

**Why two systems?** The host CLI uses CommonJS because it's simpler and runs directly without compilation. The plugin uses ESM because TypeScript outputs ESM, and OpenClaw's plugin system expects ESM.

#### Python (for the Blueprint)

The blueprint orchestrator is Python 3.11+. If you know C, Python will feel familiar but without semicolons, braces, or type declarations (though type hints exist).

Key differences:
- Indentation defines blocks (no `{}`)
- `subprocess.run()` is like C's `system()` or `exec()`
- `argparse` is like C's `getopt`
- `yaml.safe_load()` reads YAML files into dictionaries

#### Docker

**What it is**: A tool that packages applications into containers — isolated environments with their own filesystem, network, and processes. Think of it as a lightweight virtual machine.

**Key concepts**:
- `Dockerfile` = Build recipe for a container image
- `docker build` = Compile the Dockerfile into an image
- `docker run` = Start a container from an image
- Each line in a Dockerfile creates a "layer" (like a git commit)

#### YAML

A human-readable data format used for configuration files (like JSON but with indentation instead of braces):

```yaml
# YAML (what NemoClaw uses for policies and blueprints)
network_policies:
  github:
    endpoints:
      - host: github.com
        port: 443
```

This is equivalent to:
```json
{
  "network_policies": {
    "github": {
      "endpoints": [
        { "host": "github.com", "port": 443 }
      ]
    }
  }
}
```

---

## 2. Setting Up Your Development Environment

### Step 1: Install Node.js

Download Node.js 22 LTS from https://nodejs.org

Verify installation:
```bash
node --version    # Should show v22.x.x or higher
npm --version     # Should show 10.x.x or higher
```

### Step 2: Install Python 3.11+

- **Linux**: `sudo apt install python3 python3-pip python3-venv`
- **macOS**: `brew install python@3.11`
- **Windows**: Download from https://python.org

Verify:
```bash
python3 --version    # Should show 3.11.x or higher
```

### Step 3: Install Docker

- **Linux**: `sudo apt install docker.io && sudo usermod -aG docker $USER`
- **macOS**: Install Docker Desktop from https://docker.com
- **Windows**: Install Docker Desktop with WSL2 backend

Verify:
```bash
docker --version    # Should show 20.10+ or higher
docker info         # Should not show errors
```

### Step 4: Clone the Repository

```bash
git clone https://github.com/NVIDIA/NemoClaw.git
cd NemoClaw
```

### Step 5: Install Dependencies

```bash
# Install root dependencies
npm install

# Install plugin dependencies
cd nemoclaw
npm install
cd ..
```

**What just happened?**
- `npm install` read `package.json`, downloaded all listed dependencies, and put them in `node_modules/`.
- The root `package.json` has one dependency: `openclaw@2026.3.11`
- The plugin `package.json` has runtime deps (commander, yaml, json5, tar) and dev deps (typescript, eslint, prettier)

### Step 6: Build the TypeScript Plugin

```bash
cd nemoclaw
npm run build
cd ..
```

**What just happened?**
- `npm run build` executed `tsc` (TypeScript Compiler)
- `tsc` read `tsconfig.json` for configuration
- It compiled every `.ts` file in `src/` into `.js` files in `dist/`
- The compiled files are what actually runs at runtime

### Step 7: Verify Everything Works

```bash
# Run all tests
npm test

# Check code quality (lint + format + type check)
cd nemoclaw
npm run check
cd ..
```

If tests pass and check succeeds, your environment is ready.

---

## 3. Understanding the Codebase

### Directory Map with Annotations

```
NemoClaw-main/
│
├── bin/                          # HOST CLI — runs on user's machine
│   ├── nemoclaw.js               #   Main entry point (Node.js script)
│   │                             #   Reads process.argv, dispatches to handlers
│   │                             #   CommonJS (require/module.exports)
│   │
│   └── lib/                      #   CLI helper modules
│       ├── runner.js             #     Shell command execution (run, runCapture)
│       ├── registry.js           #     Sandbox registry (CRUD on sandboxes.json)
│       ├── credentials.js        #     API key storage (credentials.json, mode 600)
│       ├── onboard.js            #     7-step interactive setup wizard
│       ├── nim.js                #     NIM container management (GPU detection, etc.)
│       ├── policies.js           #     Network policy preset management
│       ├── preflight.js          #     System prerequisite checks
│       ├── nim-images.json       #     Model-to-Docker-image mapping
│       └── install.sh            #     OpenShell installer script
│
├── nemoclaw/                     # OPENCLAW PLUGIN — runs inside OpenClaw process
│   ├── src/                      #   TypeScript source (ESM)
│   │   ├── index.ts              #     Plugin entry point: register(api)
│   │   │                         #     Registers: slash command, CLI, provider
│   │   │
│   │   ├── cli.ts                #     Commander.js subcommand wiring
│   │   │                         #     Maps "openclaw nemoclaw <cmd>" to handlers
│   │   │
│   │   ├── commands/             #     Command handler implementations
│   │   │   ├── launch.ts         #       Fresh bootstrap (resolve → verify → plan → apply)
│   │   │   ├── migrate.ts        #       Host → sandbox migration (snapshot → archive → sync)
│   │   │   ├── connect.ts        #       Open interactive shell in sandbox
│   │   │   ├── status.ts         #       Show sandbox + inference health
│   │   │   ├── logs.ts           #       Stream blueprint/sandbox logs
│   │   │   ├── eject.ts          #       Rollback from sandbox to host
│   │   │   ├── onboard.ts        #       Configure inference endpoint
│   │   │   ├── slash.ts          #       /nemoclaw chat command handler
│   │   │   └── migration-state.ts#       Snapshot creation/restoration logic
│   │   │
│   │   ├── blueprint/            #     Blueprint lifecycle management
│   │   │   ├── resolve.ts        #       Find blueprint (cache or registry)
│   │   │   ├── fetch.ts          #       Download from OCI registry
│   │   │   ├── verify.ts         #       SHA-256 digest + version compatibility
│   │   │   ├── exec.ts           #       Run Python runner as subprocess
│   │   │   └── state.ts          #       Read/write ~/.nemoclaw/state/nemoclaw.json
│   │   │
│   │   └── onboard/              #     Onboarding configuration
│   │       ├── config.ts         #       Load/save ~/.nemoclaw/config.json
│   │       ├── prompt.ts         #       Interactive prompts (endpoint type, model)
│   │       └── validate.ts       #       Input validation (API key format, URLs)
│   │
│   ├── dist/                     #   Compiled JavaScript (DO NOT EDIT)
│   ├── openclaw.plugin.json      #   Plugin manifest (ID, name, config schema)
│   ├── package.json              #   Plugin dependencies and scripts
│   └── tsconfig.json             #   TypeScript compiler configuration
│
├── nemoclaw-blueprint/           # PYTHON BLUEPRINT — orchestration logic
│   ├── blueprint.yaml            #   Version, profiles, compatibility constraints
│   ├── orchestrator/
│   │   └── runner.py             #   Main orchestrator: plan/apply/status/rollback
│   ├── policies/
│   │   ├── openclaw-sandbox.yaml #   Base security policy (Landlock, seccomp, network)
│   │   └── presets/              #   Service-specific network allowlists
│   │       ├── slack.yaml
│   │       ├── discord.yaml
│   │       ├── jira.yaml
│   │       ├── telegram.yaml
│   │       ├── pypi.yaml
│   │       ├── npm.yaml
│   │       ├── huggingface.yaml
│   │       ├── outlook.yaml
│   │       └── docker.yaml
│   └── migrations/
│       └── snapshot.py           #   Migration state handling
│
├── scripts/                      # SHELL SCRIPTS — setup and deployment
│   ├── nemoclaw-start.sh         #   Sandbox startup script
│   ├── setup.sh                  #   Legacy setup (deprecated)
│   ├── install.sh                #   OpenShell CLI installer
│   ├── start-services.sh         #   Telegram bridge + services
│   ├── brev-setup.sh             #   Remote VM setup
│   └── telegram-bridge.js        #   Telegram bot integration
│
├── test/                         # TESTS — Node.js native test runner
│   ├── cli.test.js               #   CLI dispatcher tests
│   ├── registry.test.js          #   Sandbox registry tests
│   ├── credentials.test.js       #   Credential storage tests
│   ├── policies.test.js          #   Policy preset tests
│   ├── preflight.test.js         #   System check tests
│   └── nim.test.js               #   NIM container tests
│
├── docs/                         # DOCUMENTATION
│
├── Dockerfile                    # Sandbox container image recipe
├── .dockerignore                 # Files excluded from Docker builds
├── package.json                  # Root project manifest
├── package-lock.json             # Locked dependency versions
├── CLAUDE.md                     # Claude Code development guidelines
├── CONTRIBUTING.md               # Contribution guidelines
└── SECURITY.md                   # Security vulnerability reporting
```

### How Data Flows Through the System

Here's the flow when a user runs `nemoclaw onboard`:

```
1. User types: nemoclaw onboard
2. bin/nemoclaw.js reads process.argv → cmd = "onboard"
3. Dispatches to onboard() function → requires lib/onboard.js
4. onboard.js runs 7 steps:
   a. preflight.js checks Docker, OpenShell, cgroups, GPU
   b. runner.js calls: openshell gateway start
   c. runner.js calls: openshell sandbox create --from Dockerfile
   d. nim.js detects GPU, presents model options
   e. runner.js calls: openshell provider create + inference set
   f. OpenClaw gateway starts inside sandbox
   g. policies.js applies network presets via openshell policy set
5. registry.js saves sandbox metadata to ~/.nemoclaw/sandboxes.json
6. credentials.js saves API key to ~/.nemoclaw/credentials.json
```

---

## 4. How to Build the Project

### Full Build

```bash
# From the project root:

# Step 1: Install all dependencies
npm install
cd nemoclaw && npm install && cd ..

# Step 2: Compile TypeScript → JavaScript
cd nemoclaw && npm run build && cd ..
```

### Watch Mode (Auto-Rebuild on Save)

During development, you can auto-rebuild when you change TypeScript files:

```bash
cd nemoclaw
npm run dev    # Runs: tsc --watch
```

This watches for file changes in `src/` and automatically recompiles to `dist/`.

### Clean Build

```bash
cd nemoclaw
npm run clean    # Removes dist/ directory
npm run build    # Fresh compile
```

### Understanding the Build Pipeline

```
nemoclaw/src/*.ts                nemoclaw/dist/*.js
    |                                 ^
    | TypeScript Compiler (tsc)       |
    | reads tsconfig.json             |
    +-------------------------------->+

tsconfig.json settings:
  - target: ES2022 (modern JavaScript)
  - module: NodeNext (ESM imports)
  - strict: true (catch more bugs)
  - outDir: dist/
  - rootDir: src/
```

**Important**: Never edit files in `dist/`. They are generated. Always edit in `src/`.

---

## 5. How to Run Tests

### Run All Tests

```bash
npm test
```

This executes: `node --test test/*.test.js`

### Run a Single Test File

```bash
node --test test/registry.test.js
```

### Understanding the Test Framework

NemoClaw uses **Node.js's built-in test runner** (not Jest or Mocha). Here's how it works:

```javascript
// test/example.test.js
const { describe, it } = require("node:test");
const assert = require("node:assert/strict");

describe("MyModule", () => {
  it("should do something", () => {
    assert.strictEqual(1 + 1, 2);
  });

  it("should handle errors", () => {
    assert.throws(() => {
      throw new Error("oops");
    }, /oops/);
  });
});
```

**Comparison to JUnit (Java)**:
- `describe()` = `@Nested class`
- `it()` = `@Test void testSomething()`
- `assert.strictEqual()` = `assertEquals()`
- `assert.throws()` = `assertThrows()`

### Writing New Tests

1. Create a new file in `test/` named `yourmodule.test.js`
2. Use `require()` to import the module you're testing
3. Use `describe()`/`it()`/`assert` to write tests
4. Run with `node --test test/yourmodule.test.js`

---

## 6. How to Add a New CLI Command

### To the Host CLI (bin/nemoclaw.js)

**Example**: Adding a `nemoclaw backup` command

**Step 1**: Add to GLOBAL_COMMANDS set in `bin/nemoclaw.js`:

```javascript
// bin/nemoclaw.js, line 23
const GLOBAL_COMMANDS = new Set([
  "onboard", "list", "deploy", "setup", "setup-spark",
  "start", "stop", "status",
  "backup",    // <-- Add here
  "help", "--help", "-h",
]);
```

**Step 2**: Create the handler function:

```javascript
// bin/nemoclaw.js, add before the Dispatch section
async function backup() {
  const sandboxName = process.argv[3] || "default";
  console.log(`  Backing up sandbox '${sandboxName}'...`);
  // Your logic here
  run(`openshell sandbox export ${sandboxName} --output /tmp/backup.tar`);
  console.log(`  ✓ Backup complete`);
}
```

**Step 3**: Add to the switch statement:

```javascript
// bin/nemoclaw.js, inside the GLOBAL_COMMANDS switch
case "backup":  await backup(); break;
```

**Step 4**: Add to the help text:

```javascript
// In the help() function
console.log(`    nemoclaw backup [name]         Backup a sandbox`);
```

### To the Plugin CLI (openclaw nemoclaw <cmd>)

**Example**: Adding `openclaw nemoclaw snapshot`

**Step 1**: Create the command handler file:

```typescript
// nemoclaw/src/commands/snapshot.ts
import type { PluginLogger, NemoClawConfig } from "../index.js";

export interface SnapshotOptions {
  name: string;
  logger: PluginLogger;
  pluginConfig: NemoClawConfig;
}

export async function cliSnapshot(opts: SnapshotOptions): Promise<void> {
  const { name, logger, pluginConfig } = opts;
  logger.info(`Creating snapshot of sandbox '${name}'...`);
  // Your implementation here
  logger.info("Snapshot complete.");
}
```

**Step 2**: Wire it up in `cli.ts`:

```typescript
// nemoclaw/src/cli.ts
import { cliSnapshot } from "./commands/snapshot.js";

// Inside registerCliCommands(), add:
nemoclaw
  .command("snapshot")
  .description("Create a snapshot of the current sandbox state")
  .option("--name <name>", "Snapshot name", "default")
  .action(async (opts: { name: string }) => {
    await cliSnapshot({ name: opts.name, logger, pluginConfig });
  });
```

**Step 3**: Rebuild:

```bash
cd nemoclaw && npm run build
```

---

## 7. How to Add a New Policy Preset

Policy presets are YAML files in `nemoclaw-blueprint/policies/presets/`.

**Example**: Adding a `gitlab` preset

**Step 1**: Create the preset file:

```yaml
# nemoclaw-blueprint/policies/presets/gitlab.yaml
name: gitlab
description: Allow GitLab API and Git operations
network_policies:
  gitlab:
    name: gitlab
    endpoints:
      - host: gitlab.com
        port: 443
        access: full
      - host: registry.gitlab.com
        port: 443
        access: full
    binaries:
      - { path: /usr/bin/git }
      - { path: /usr/bin/curl }
```

**Step 2**: The `policies.js` module auto-discovers presets from the `presets/` directory, so no code changes are needed in most cases. Verify by running:

```bash
nemoclaw my-sandbox policy-list
```

You should see `gitlab` in the list.

---

## 8. How to Add a New Inference Provider

### Step 1: Add Profile to Blueprint

Edit `nemoclaw-blueprint/blueprint.yaml`:

```yaml
# Under components.inference.profiles, add:
my-provider:
  provider_type: "openai"
  provider_name: "my-provider"
  endpoint: "https://api.my-provider.com/v1"
  model: "my-model-name"
  credential_env: "MY_PROVIDER_API_KEY"
```

### Step 2: Register in Plugin

Edit `nemoclaw/src/index.ts`, inside the `register()` function:

```typescript
api.registerProvider({
  id: "my-provider",
  label: "My Provider",
  docsPath: "https://docs.my-provider.com",
  aliases: ["myp"],
  envVars: ["MY_PROVIDER_API_KEY"],
  models: {
    chat: [
      {
        id: "my-model-name",
        label: "My Model",
        contextWindow: 32768,
        maxOutput: 4096,
      },
    ],
  },
  auth: [
    {
      type: "bearer",
      envVar: "MY_PROVIDER_API_KEY",
      headerName: "Authorization",
      label: "My Provider API Key",
    },
  ],
});
```

### Step 3: Add to Onboard Options

Edit `bin/lib/onboard.js` to add your provider to the inference options menu.

### Step 4: Rebuild

```bash
cd nemoclaw && npm run build
```

---

## 9. How to Modify the Blueprint

The blueprint is the Python layer that translates high-level intentions into `openshell` CLI commands.

### Adding a New Action

**Step 1**: Define the action function in `nemoclaw-blueprint/orchestrator/runner.py`:

```python
def action_health(blueprint: dict) -> None:
    """Check health of all components."""
    emit_run_id()
    progress(10, "Checking sandbox health")

    sandbox_name = blueprint.get("components", {}).get("sandbox", {}).get("name", "openclaw")
    result = run_cmd(
        ["openshell", "sandbox", "status", sandbox_name],
        check=False,
        capture=True,
    )

    if result.returncode == 0:
        progress(100, "All healthy")
        log(f"Sandbox '{sandbox_name}' is healthy")
    else:
        log(f"ERROR: Sandbox '{sandbox_name}' is unhealthy: {result.stderr}")
        sys.exit(1)
```

**Step 2**: Register it in the CLI parser and dispatcher:

```python
# In main():
parser.add_argument("action", choices=["plan", "apply", "status", "rollback", "health"])

# In the dispatch block:
elif args.action == "health":
    action_health(blueprint)
```

**Step 3**: Call it from TypeScript:

```typescript
const result = await execBlueprint({
  blueprintPath: blueprint.localPath,
  action: "health" as BlueprintAction,  // Add "health" to BlueprintAction type
  profile: "default",
}, logger);
```

---

## 10. Debugging Guide

### Debugging the Host CLI

```bash
# Add NODE_DEBUG for verbose output:
NODE_DEBUG=child_process nemoclaw onboard

# Or add console.log statements to bin/nemoclaw.js and bin/lib/*.js
# (no compilation needed — these are plain JavaScript)
```

### Debugging the TypeScript Plugin

```bash
# 1. Add console.log/logger.info statements to src/*.ts
# 2. Rebuild:
cd nemoclaw && npm run build

# 3. Run with OpenClaw:
openclaw nemoclaw status
```

### Debugging the Python Blueprint

```bash
# Run the blueprint runner directly:
cd nemoclaw-blueprint
NEMOCLAW_BLUEPRINT_PATH=. python3 orchestrator/runner.py plan --profile default

# Add print() statements to runner.py — they appear in stdout
```

### Debugging OpenShell Commands

```bash
# Run openshell commands directly to isolate issues:
openshell status                     # Gateway health
openshell sandbox list               # List sandboxes
openshell sandbox status openclaw    # Sandbox health
openshell sandbox logs openclaw      # Sandbox logs
openshell sandbox connect openclaw   # Interactive shell into sandbox
```

### Common Debug Patterns

**Problem**: "Blueprint runner not found"
```
Check: Is the blueprint cached?
ls ~/.nemoclaw/blueprints/0.1.0/orchestrator/runner.py
```

**Problem**: "openshell CLI not found"
```
Check: Is OpenShell installed?
which openshell
openshell --version
```

**Problem**: "Sandbox already exists"
```
Fix: Delete and recreate
openshell sandbox delete openclaw
nemoclaw onboard
```

---

## 11. Code Conventions

### License Headers

Every source file must start with:

```
// SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
// SPDX-License-Identifier: Apache-2.0
```

### TypeScript Style

- Strict mode (`"strict": true` in tsconfig.json)
- No `any` types unless absolutely necessary
- ESM imports with `.js` extensions (required for Node.js ESM):
  ```typescript
  import { foo } from "./foo.js";  // NOT ./foo or ./foo.ts
  ```
- Prefer `interface` over `type` for object shapes
- Error handling: log + return early (don't throw in command handlers)

### JavaScript Style (Host CLI)

- CommonJS (`require`/`module.exports`)
- No TypeScript — plain JavaScript
- Use `const` by default, `let` when reassignment is needed
- Never `var`

### Python Style

- Type hints for function signatures
- `argparse` for CLI arguments
- Never `shell=True` in `subprocess.run()` (security)
- Use `pathlib.Path` for filesystem operations

### Formatting

```bash
# Format all TypeScript files:
cd nemoclaw && npm run format

# Check formatting without changing files:
cd nemoclaw && npm run format:check

# Lint:
cd nemoclaw && npm run lint

# Auto-fix lint issues:
cd nemoclaw && npm run lint:fix
```

---

## 12. Common Gotchas

### Gotcha 1: Editing dist/ Instead of src/

**Problem**: You edit `nemoclaw/dist/index.js` and your changes disappear after rebuilding.

**Fix**: Always edit `nemoclaw/src/index.ts`. The `dist/` directory is generated.

### Gotcha 2: Forgetting .js Extensions in TypeScript Imports

**Problem**: TypeScript import works at compile time but fails at runtime.

```typescript
// WRONG — compiles but crashes at runtime
import { foo } from "./foo";

// CORRECT — works at runtime with Node.js ESM
import { foo } from "./foo.js";
```

### Gotcha 3: CommonJS vs ESM Confusion

**Problem**: Using `import` in `bin/` files or `require()` in `nemoclaw/src/` files.

**Rule**:
- `bin/` files use `require()` and `module.exports`
- `nemoclaw/src/` files use `import` and `export`

### Gotcha 4: Node.js Version

**Problem**: Tests fail with syntax errors or missing APIs.

**Fix**: Ensure Node.js 20+ is installed. The built-in test runner requires it.

### Gotcha 5: Credentials File Permissions

**Problem**: Credential operations fail on Windows.

**Note**: The `mode: 0o600` permission model works on Linux/macOS. On Windows, Node.js applies the closest equivalent, but NTFS permissions differ.

### Gotcha 6: Docker Socket on macOS

**Problem**: Docker commands fail with "Cannot connect to Docker daemon."

**Fix**: If using Colima (Docker alternative for macOS), set:
```bash
export DOCKER_HOST=unix://$HOME/.colima/default/docker.sock
```

### Gotcha 7: Python Not Found

**Problem**: Blueprint commands fail with "python3 not found."

**Fix**: Ensure Python 3.11+ is in your PATH. On some systems, it's `python` not `python3`:
```bash
# Check:
python3 --version
# If not found, try:
python --version
# If Python exists but as 'python', create an alias or symlink
```

### Gotcha 8: Blueprint Cache

**Problem**: Changes to `nemoclaw-blueprint/` don't take effect.

**Fix**: Clear the cached blueprint:
```bash
rm -rf ~/.nemoclaw/blueprints/0.1.0
```

The next run will re-resolve the blueprint from the local source.
