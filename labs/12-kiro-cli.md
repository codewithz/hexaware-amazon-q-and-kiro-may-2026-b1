# Lab — Kiro CLI

**Day:** 2  
**Layer:** Layer 1 (Governance)  
**Duration:** 30 minutes  
**Tool:** Terminal  
**Deliverable:** Kiro CLI authenticated, verified, and core commands exercised

---

## What You Will Learn

By the end of this lab you will have:

- Understood what Kiro CLI is, how it relates to Kiro IDE, and when to use each
- Installed and authenticated Kiro CLI
- Used the core CLI commands: chat, agent, settings, translate, diagnostic

---

## Understanding Kiro CLI

### What It Is

Kiro CLI is the terminal-based interface to the same AI agent engine that powers Kiro IDE. It is not a stripped-down version or a different product — it runs the same model, reads the same `.kiro/` folder, loads the same steering files, invokes the same custom agents, and uses the same MCP configuration. The only difference is the surface: a terminal REPL instead of a graphical IDE panel.

```
┌─────────────────────────────────────────────────────┐
│                   Kiro Agent Engine                 │
│  (same model · same agents · same MCP · same rules) │
└────────────────┬────────────────┬───────────────────┘
                 │                │
     ┌───────────▼──────┐  ┌──────▼────────────┐
     │    Kiro IDE      │  │    Kiro CLI        │
     │  (GUI panel)     │  │  (terminal REPL)   │
     └──────────────────┘  └────────────────────┘
```

Both surfaces share the same `.kiro/` folder. A steering file you add in the IDE is immediately visible in the CLI, and a custom agent you create via `kiro-cli agent create` appears immediately in the IDE agent list.

### Where Kiro CLI Fits in the Development Workflow

Kiro IDE is where developers spend most of their time — writing code, reviewing spec files, running agents interactively in the side panel. Kiro CLI covers the scenarios where a GUI is not available or not practical:

| Scenario | Why CLI is the right tool |
|---|---|
| **SSH session on a remote server** | No display available — IDE cannot open |
| **CI/CD pipeline** | No human at the keyboard — agents run unattended via `--no-interactive --trust-all-tools` |
| **Pre-commit / pre-push hooks** | Shell scripts need to call the agent inline without switching context |
| **Quick one-shot questions** | Faster than opening the IDE for a single question that needs no file editing |
| **Scripted batch operations** | Run an agent across 50 files in a loop without IDE GUI overhead |
| **Pair programming in a shared terminal** | Both developers see the same session without screen-sharing an IDE window |

### The Shared .kiro/ Folder

This is the most important concept to internalise before using the CLI:

```
<project-root>/
└── .kiro/
    ├── agents/          ← Custom agent definitions — shared by IDE and CLI
    ├── hooks/           ← Event hooks — IDE only (CLI has no file-watch daemon)
    ├── settings/
    │   └── mcp.json     ← MCP server config — shared by IDE and CLI
    ├── specs/           ← Spec files — shared by IDE and CLI
    └── steering/        ← Persistent context rules — shared by IDE and CLI
```

The only folder that does not apply to CLI is `hooks/` — hooks require the IDE's file-watch daemon to detect save events. Everything else transfers directly.

### CLI vs IDE: When to Use Which

Neither tool replaces the other. Use them together:

| Task | IDE | CLI |
|---|---|---|
| Writing a new feature with agent assistance | ✓ | |
| Running the security-agent across all files in CI | | ✓ |
| Reviewing a spec file before approving | ✓ | |
| Generating a PR description from the current branch diff | ✓ | ✓ |
| Running an agent as part of a shell script | | ✓ |
| Debugging why a steering file is not being loaded | | ✓ (`/context show`) |
| Watching files and running agents on save | ✓ | |
| One-shot question while staying in the terminal | | ✓ |

### How the CLI Handles Context

The CLI manages context through sessions. Each session stores the full conversation history and the loaded context (steering files, MCP tools, active agents). Sessions are scoped to the directory you were in when you started them.

```bash
kiro-cli                       # starts or resumes a session for this directory
kiro-cli chat --resume         # explicitly resume the last session
kiro-cli chat --list-sessions  # see all saved sessions for this directory
```

If you start a long session and notice the agent's answers getting less precise, run `/usage` inside the session to check context window usage. Start a new session to reset — the agent will reload steering and MCP context fresh.

### Security Model

The CLI uses the same IAM identity as the IDE. There is no separate credential. When you authenticate with `/login`, the CLI uses either your AWS Builder ID or IAM Identity Center SSO token — the same token that governs what the IDE agent can access.

In CI/CD, the `--trust-all-tools` flag pre-approves all tool calls. This is equivalent to running the IDE agent with all permission prompts disabled. Only use it with agents that have minimal, explicitly scoped tool access — not with agents that have broad `write` permissions.

---

## Part 1 — Install and Authenticate Kiro CLI

### Step 1.1 — Install

```bash
curl -fsSL https://cli.kiro.dev/install | bash

# Reload your shell profile
source ~/.bashrc    # or: source ~/.zshrc

# Verify
kiro-cli --version
# Expected: a version number (1.x.x or higher)
```

If `kiro-cli` is not found after install, check that `~/.local/bin` or `~/.cargo/bin` is in your `PATH`:

```bash
echo $PATH
# If missing:
export PATH="$HOME/.local/bin:$PATH"
# Then retry: kiro-cli --version
```

### Step 1.2 — Sign In

```bash
kiro-cli
```

At the `>` prompt, type:

```
/login
```

A browser window will open. Choose the same credentials you use in Kiro IDE:

- **AWS Builder ID** — for individual accounts (free tier)
- **IAM Identity Center** — for enterprise SSO

Complete the browser flow and return to the terminal. The CLI should say `Logged in as [your account name]`.

### Step 1.3 — Verify Authentication

```bash
kiro-cli chat "What is my current working directory?"
```

Expected response: `Your current working directory is /home/[username]/ai-sdlc-training`

If you get an authentication error, run `/login` again inside the session.

---

## Part 2 — Core CLI Commands

### Step 2.1 — Chat Modes

```bash
# Interactive session (most common — opens a REPL)
kiro-cli

# One-shot question (exits when answered)
kiro-cli chat "Explain the InventoryService class in this project"

# Non-interactive — used in CI/CD scripts
kiro-cli chat --no-interactive --trust-all-tools "Run mvn test and report any failures"
```

> **What `--trust-all-tools` does:** In interactive mode, the CLI asks permission before reading files, running commands, etc. `--trust-all-tools` pre-approves all these actions. Only use this in CI where no human is present, and only with agents that have minimal, read-only tool access.

### Step 2.2 — Session Management

```bash
# Resume the last session (keeps all previous context)
kiro-cli chat --resume

# Pick a session from a list
kiro-cli chat --resume-picker

# List all saved sessions
kiro-cli chat --list-sessions
```

Sessions are persisted per directory. The CLI associates sessions with the directory you were in when you started them.

### Step 2.3 — In-Session Slash Commands

Start a session with `kiro-cli`, then try each of these:

```
/context show
```

Lists everything loaded: steering files, custom agents, MCP servers and their tools. This is your primary debugging command — if an agent is not seeing your steering files, run this first.

```
/usage
```

Shows how much of the context window is currently used. If you are in a long session and the agent starts giving worse answers, high context usage may be the reason. Start a new session to reset.

```
/model
```

Lists available models and lets you switch for this session. Switching model does not affect your `cli.json` settings — it only applies to the current session.

```
!git status
```

The `!` prefix runs a shell command from inside the session without leaving it:

```
!ls .kiro/agents/
!cat .kiro/steering/architecture.md
!mvn test -q
```

### Step 2.4 — Agent Management

```bash
# List agents available in the current directory
kiro-cli agent list

# Create a new agent interactively
kiro-cli agent create my-new-agent

# Edit an existing agent
kiro-cli agent edit code-review-agent

# Validate an agent file (checks YAML frontmatter syntax)
kiro-cli agent validate .kiro/agents/code-review-agent.md

# Set the default agent for this project
kiro-cli agent set-default code-review-agent

# Use a specific agent for a one-shot question
kiro-cli chat --agent security-agent "Scan InventoryService.java for issues"
```

### Step 2.5 — Settings Management

```bash
# See all currently configured settings
kiro-cli settings list

# See all available settings with descriptions
kiro-cli settings list --all

# Check a specific setting
kiro-cli settings telemetry.enabled

# Change a setting
kiro-cli settings chat.defaultModel "claude-sonnet-4-5"

# Disable the greeting (speeds up session start)
kiro-cli settings chat.greeting.enabled false

# Open the settings file in your editor
kiro-cli settings open
# Settings are stored at: ~/.kiro/settings/cli.json
```

### Step 2.6 — Translate Natural Language to Shell Commands

```bash
kiro-cli translate "find all Java files modified in the last 3 days"
# Output: find . -name "*.java" -mtime -3

kiro-cli translate "compress log files older than 7 days in the logs folder"
# Output: find logs/ -name "*.log" -mtime +7 -exec gzip {} \;

kiro-cli translate "count the number of lines in all Java files in the src directory"
# Output: find src -name "*.java" | xargs wc -l | tail -1
```

The translate command generates the shell command without running it. Copy the output to your terminal to execute.

### Step 2.7 — Diagnostics

```bash
# Full diagnostic (requires kiro-cli app running)
kiro-cli diagnostic

# Quick diagnostic without the app (works in SSH, CI)
kiro-cli diagnostic --force

# Output as JSON (for log ingestion)
kiro-cli diagnostic --format json
```

Use `--force` when troubleshooting authentication or connectivity issues in a new environment.

---

## Lab Completion Criteria

- [ ] `kiro-cli --version` returns a version number
- [ ] `kiro-cli chat "What is my current directory?"` responds with the correct path
- [ ] `/login` completed successfully — CLI shows your account name
- [ ] `kiro-cli agent list` shows agents from `.kiro/agents/`
- [ ] `/context show` inside a session lists steering files and custom agents
- [ ] `/usage` shows context window consumption for the current session
- [ ] `kiro-cli translate "find Java files modified today"` returns a valid `find` command
- [ ] `kiro-cli diagnostic --force` runs without errors
- [ ] At least one agent invoked successfully via `kiro-cli chat --agent`