# MCP Servers — Where to Use Them in Your Kiro Project

> **Reference guide for:** Java / Spring Boot · Kiro IDE · Amazon Q Developer  
> **Covers:** Every location where `mcp.json` can be placed, every context where MCP tools activate, and a copy-paste template for each scenario.

---

## What is `mcp.json`?

`mcp.json` is the configuration file that tells Kiro and Amazon Q Developer which external tools (MCP servers) your agents can call. It maps a server name to a transport — either a local process (`stdio`) or a remote endpoint (`sse`).

**There is not one single `mcp.json` — there are multiple, at different scopes.** Each scope controls who sees which servers and when.

---

## The Four Places You Can Put `mcp.json`

```
~/.kiro/settings/mcp.json              # 1. User-global  (your machine, all projects)
<repo>/.kiro/settings/mcp.json         # 2. Workspace    (this project, all team members)
<repo>/.kiro/agents/<agent>.json       # 3. Per-agent    (only this agent sees these tools)
~/.aws/amazonq/mcp.json                # 4. Amazon Q Dev (Q Developer CLI / IDE, user-global)
```

Each is explained below with a full template.

---

## 1. User-Global MCP (`~/.kiro/settings/mcp.json`)

**Who it applies to:** You only, across every project on your machine.  
**When to use it:** Personal tools — your own GitHub token, your personal Jira account, a local database you always connect to.  
**Team implication:** Not committed to Git. Each developer sets this up individually.

### Template

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      },
      "autoApprove": ["get_pull_request", "list_issues", "get_issue"],
      "disabled": false
    },
    "fetch": {
      "command": "uvx",
      "args": ["mcp-server-fetch"],
      "autoApprove": ["fetch"],
      "disabled": false
    }
  }
}
```

### What each field does

| Field | Purpose |
|---|---|
| `command` | The executable to launch the MCP server process |
| `args` | Arguments passed to the command (server package name) |
| `env` | Environment variables injected into the server process — use `${VAR}` syntax, never raw secrets |
| `autoApprove` | Tool names that are approved without prompting the user — keep this list minimal |
| `disabled` | Set to `true` to turn off without deleting the config |

---

## 2. Workspace MCP (`.kiro/settings/mcp.json`)

**Who it applies to:** Every team member who opens this repository in Kiro.  
**When to use it:** Shared project tools — AWS documentation, Git operations, the project's own REST API, Jira board for this project.  
**Team implication:** Commit this file to Git. It becomes the team's shared MCP baseline.

### Template

```json
{
  "mcpServers": {
    "aws-docs": {
      "command": "uvx",
      "args": ["awslabs.aws-documentation-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      "autoApprove": ["search_documentation", "get_documentation"],
      "disabled": false
    },
    "git": {
      "command": "uvx",
      "args": ["mcp-server-git", "--repository", "."],
      "autoApprove": ["git_diff", "git_log", "git_show", "git_status"],
      "disabled": false
    },
    "jira": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-jira"],
      "env": {
        "JIRA_BASE_URL": "${JIRA_BASE_URL}",
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}",
        "JIRA_USER_EMAIL": "${JIRA_USER_EMAIL}"
      },
      "autoApprove": ["get_issue", "search_issues"],
      "disabled": false
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "${DB_CONNECTION_STRING}"
      },
      "autoApprove": [],
      "disabled": true
    }
  }
}
```

> **Security rule:** Raw credentials must never appear in this file. Use `${ENV_VAR}` references only. Create a `.env.example` at the repo root listing all required variables without values, and add `.env` to `.gitignore`.

### Companion `.env.example`

```bash
# Copy this to .env and fill in your values. Never commit .env to Git.
JIRA_BASE_URL=https://your-org.atlassian.net
JIRA_API_TOKEN=
JIRA_USER_EMAIL=
DB_CONNECTION_STRING=postgresql://localhost:5432/inventory_db
```

---

## 3. Per-Agent MCP (inside `.kiro/agents/<agent>.json`)

**Who it applies to:** Only the named agent. Other agents in the same project do not see these tools.  
**When to use it:** Least-privilege scoping — the PR agent needs Git, not the database. The schema agent needs Postgres, not Jira.  
**Team implication:** Committed to Git as part of the agent definition.

### Why this matters

If you define all MCP servers at workspace level, every agent has access to everything. That violates least-privilege. Per-agent MCP scoping means:

- The `docs-agent` can only call `aws-docs` tools
- The `pr-agent` can only call `git` tools
- The `db-schema-agent` can only call `postgres` tools

No agent can accidentally call a tool it was not designed to use.

### Template — `code-review-agent.json`

```json
{
  "name": "code-review-agent",
  "description": "Reviews Java source files for quality, security, and standards compliance",
  "model": "CLAUDE_SONNET",
  "tools": ["read_file", "list_directory"],
  "systemPrompt": "You are a senior Java engineer performing a code review. Read the file provided. Check it against the standards in .kiro/steering/testing-standards.md, architecture.md, and api-conventions.md. Report issues as a numbered list with severity (CRITICAL / HIGH / MEDIUM / LOW) and a suggested fix for each.",
  "mcp": {
    "servers": ["git"],
    "autoApprove": ["git_diff", "git_log"]
  }
}
```

### Template — `pr-description-agent.json`

```json
{
  "name": "pr-description-agent",
  "description": "Generates a structured PR description from the git diff and linked Jira ticket",
  "model": "CLAUDE_SONNET",
  "tools": ["read_file"],
  "systemPrompt": "You are a technical writer creating a GitHub pull request description. Use git_diff to get the staged changes. Use get_issue to fetch the linked Jira ticket (ticket ID will be provided). Generate a PR description with sections: Summary, Changes Made, Testing Done, Breaking Changes (if any), and Linked Ticket.",
  "mcp": {
    "servers": ["git", "jira"],
    "autoApprove": ["git_diff", "git_status", "get_issue"]
  }
}
```

### Template — `architect-agent.json`

```json
{
  "name": "architect-agent",
  "description": "Proposes AWS service configurations grounded in live AWS documentation",
  "model": "CLAUDE_SONNET",
  "tools": ["read_file", "write_file"],
  "systemPrompt": "You are a solutions architect. Before recommending any AWS service configuration, use search_documentation and get_documentation to verify the current capability, limits, and pricing. Reference .kiro/steering/architecture.md for our existing AWS topology. Never recommend a configuration based on training data alone — always verify against live documentation.",
  "mcp": {
    "servers": ["aws-docs"],
    "autoApprove": ["search_documentation", "get_documentation"]
  }
}
```

---

## 4. Amazon Q Developer MCP (`~/.aws/amazonq/mcp.json`)

**Who it applies to:** Your Amazon Q Developer CLI and IDE extension (not Kiro IDE).  
**When to use it:** Tools you want available inside Amazon Q Developer specifically — when you are working in VS Code or IntelliJ with the Q extension rather than Kiro IDE.  
**Team implication:** User-level file, not committed to Git.

### Template

```json
{
  "mcpServers": {
    "aws-docs": {
      "command": "uvx",
      "args": ["awslabs.aws-documentation-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      }
    },
    "fetch": {
      "command": "uvx",
      "args": ["mcp-server-fetch"]
    }
  }
}
```

> Amazon Q Developer reads this file automatically on launch. No additional configuration is required in the IDE.

---

## Five Real Use Cases in Your Inventory Management Service

### Use Case 1 — Agent looks up AWS DynamoDB limits before writing code

**Scenario:** Your `architect-agent` is designing the inventory reservation table. Before proposing a key structure, it checks the live AWS DynamoDB documentation for current partition key limits.

**MCP server used:** `aws-docs`  
**Tools called:** `search_documentation("DynamoDB partition key size limit")`, `get_documentation(...)`  
**Activated via:** `.kiro/agents/architect-agent.json` → `mcp.servers: ["aws-docs"]`

---

### Use Case 2 — PR description agent reads the real git diff

**Scenario:** After a developer completes the `POST /inventory/reserve` endpoint, the `@pr-description-agent` is invoked. It calls `git_diff` to read what actually changed — not what the developer says changed — and generates a PR description from the real diff.

**MCP server used:** `git`  
**Tools called:** `git_diff`, `git_log --oneline -10`  
**Activated via:** `.kiro/agents/pr-description-agent.json` → `mcp.servers: ["git"]`

---

### Use Case 3 — Security agent fetches the NVD vulnerability database

**Scenario:** The `security-review-agent` is triggered by a hook on commit. It identifies that the project uses `spring-boot 3.2.1` in `pom.xml`. It calls the `fetch` tool to query the NVD REST API for known CVEs against that version.

**MCP server used:** `fetch`  
**Tools called:** `fetch("https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=spring-boot+3.2.1")`  
**Activated via:** `.kiro/settings/mcp.json` (workspace-level, shared by all agents)

---

### Use Case 4 — Test generator agent inspects the database schema

**Scenario:** The `test-generator-agent` is writing integration tests for the inventory service. Before generating test data, it calls `postgres` MCP tools to read the actual `inventory_items` table schema — column names, types, constraints — so the generated test data matches reality.

**MCP server used:** `postgres`  
**Tools called:** `query("SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'inventory_items'")`  
**Activated via:** `.kiro/agents/test-generator-agent.json` → `mcp.servers: ["postgres"]`

---

### Use Case 5 — Docs agent fetches the Jira ticket for context

**Scenario:** A developer types `@docs-agent document JIRA-412`. The docs agent calls `get_issue("JIRA-412")` to read the acceptance criteria from the original ticket, then generates Javadoc that references the business requirement — not just the code.

**MCP server used:** `jira`  
**Tools called:** `get_issue("JIRA-412")`  
**Activated via:** `.kiro/agents/docs-agent.json` → `mcp.servers: ["jira"]`

---

## MCP Scope Decision Tree

```
Do all team members need this server?
├── YES → .kiro/settings/mcp.json  (workspace, committed to Git)
└── NO  → Is it personal/sensitive?
          ├── YES → ~/.kiro/settings/mcp.json  (user-global, not committed)
          └── NO  → Should only one agent see it?
                    ├── YES → .kiro/agents/<agent>.json  mcp.servers field
                    └── NO  → .kiro/settings/mcp.json with disabled: true until needed
```

---

## Quick Reference — Common MCP Servers

| Server | Package | What it gives your agents |
|---|---|---|
| AWS Docs | `awslabs.aws-documentation-mcp-server@latest` | Live AWS service documentation, API references, pricing |
| Git | `mcp-server-git` | `git_diff`, `git_log`, `git_show`, `git_status` on your repo |
| GitHub | `@modelcontextprotocol/server-github` | PRs, issues, repos, comments via GitHub API |
| Jira | `@modelcontextprotocol/server-jira` | Tickets, sprints, comments, transitions |
| Fetch | `mcp-server-fetch` | Any HTTP/HTTPS endpoint — NVD, internal APIs, REST services |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | Schema inspection, read queries against your DB |
| Filesystem | `@modelcontextprotocol/server-filesystem` | Scoped file read/write beyond the open workspace |
| Slack | `@modelcontextprotocol/server-slack` | Post messages, read channels — for notification agents |

---

## Governance Rules for MCP in Team Projects

**Rule 1 — No raw secrets in any `mcp.json`**  
Use `${ENV_VAR}` syntax everywhere. Secrets live in `.env` (gitignored) or your CI secret store.

**Rule 2 — `autoApprove` only for read-only tools**  
Write operations (`create_issue`, `push`, `insert`) should never be in `autoApprove`. They require explicit user confirmation every time.

**Rule 3 — Review `mcp.json` changes in pull requests**  
A new MCP server is a new permission grant. Treat it the same as an IAM policy change — it requires review.

**Rule 4 — Scope servers to the agents that need them**  
Use per-agent `mcp.servers` lists to enforce least-privilege. Do not give every agent access to every server.

**Rule 5 — Disable unused servers rather than delete them**  
Set `"disabled": true` to preserve the configuration without activating it. Makes it easy to re-enable for specific sessions.