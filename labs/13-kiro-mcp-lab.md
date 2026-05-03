# Lab — MCP Essentials: GitHub & Filesystem Servers

**Day:** 2  
**Layer:** Layer 3 (Custom Agents) + Layer 1 (Governance)  
**Duration:** 60 minutes  
**Tools:** Kiro IDE, Node.js (npx — no global install required)  
**Deliverable:** Two MCP servers running, one custom agent wired to GitHub MCP, one practical exercise completed end-to-end

---

## What You Will Learn

By the end of this lab you will:

1. Understand what MCP is and why it matters for AI-assisted development
2. Configure two zero-install MCP servers using `npx` (GitHub + Filesystem)
3. Wire a custom Kiro agent to live GitHub data
4. Use the agent to perform a real code review task against your own repository

---

## What You Need

| Requirement | Check |
|---|---|
| Node.js 18+ installed | `node --version` |
| npx available | `npx --version` |
| GitHub account | You have one |
| GitHub Personal Access Token (PAT) | Created in Step 0 below |
| Kiro IDE open with your training repo | `.kiro/` folder exists |

> **No additional software to install.** Both MCP servers in this lab use `npx` — packages are downloaded on first use and cached. No `npm install`, no global installs, no Docker.

---

## What is MCP — In Two Minutes

Your AI agent in Kiro knows what is in your workspace and what it was trained on. It does not know what is in your GitHub repository right now — open pull requests, recent commits, issue history, file contents on a branch.

**MCP (Model Context Protocol)** is the mechanism that bridges this gap. It runs a small local server process that the agent can query for live data. The agent decides when to call it, what to ask, and how to use the result — all within your conversation.

```
Your prompt ──► Agent reasons about the task
                    │
                    ▼
          Agent calls MCP server tool
          (e.g., "get open pull requests")
                    │
                    ▼
          MCP server queries GitHub API
                    │
                    ▼
          Live data returned to agent
                    │
                    ▼
          Agent uses real data in its response
```

Without MCP, the agent guesses or asks you to paste content manually.  
With MCP, the agent retrieves it autonomously and works with accurate, current data.

---

## Step 0 — Create Your GitHub Personal Access Token

You need a GitHub PAT so the GitHub MCP server can read your repositories.

### 0.1 — Open GitHub Token Settings

Go to: **GitHub → Settings → Developer Settings → Personal access tokens → Fine-grained tokens**

Direct URL: `https://github.com/settings/tokens?type=beta`

### 0.2 — Generate a New Token

Click **Generate new token** and fill in:

| Field | Value |
|---|---|
| **Token name** | `kiro-mcp-lab` |
| **Expiration** | 7 days (sufficient for the workshop) |
| **Repository access** | Only select repositories → choose your training repo |

### 0.3 — Set Permissions

Under **Repository permissions**, set these and leave everything else as `No access`:

| Permission | Level |
|---|---|
| Contents | Read-only |
| Metadata | Read-only (auto-selected) |
| Pull requests | Read-only |
| Issues | Read-only |

### 0.4 — Generate and Copy the Token

Click **Generate token**. Copy the token value — you will only see it once.

Store it in your shell environment immediately:

```bash
export GITHUB_PERSONAL_ACCESS_TOKEN="github_pat_xxxxxxxxxxxxxxxx"
```

To make it persist across terminal sessions, add that line to your `~/.bashrc` or `~/.zshrc`:

```bash
echo 'export GITHUB_PERSONAL_ACCESS_TOKEN="github_pat_xxxxxxxxxxxxxxxx"' >> ~/.zshrc
source ~/.zshrc
```

Verify it is set:
```bash
echo $GITHUB_PERSONAL_ACCESS_TOKEN
# Should print your token — not empty
```

> **Security note:** This token is scoped to read-only on one repository with a 7-day expiry. It cannot push, delete, or access other repos. Revoke it at `https://github.com/settings/tokens` after the workshop.

---

## Part 1 — Create the MCP Configuration File

### 1.1 — Create the Settings Directory

Inside your training repository root:

```bash
mkdir -p .kiro/settings
```

### 1.2 — Create `mcp.json`

Create the file `.kiro/settings/mcp.json` with this content:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
      },
      "disabled": false,
      "autoApprove": [
        "get_file_contents",
        "list_commits",
        "search_repositories",
        "get_pull_request",
        "list_pull_requests"
      ]
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/tmp"
      ],
      "disabled": false,
      "autoApprove": [
        "read_file",
        "list_directory",
        "search_files"
      ]
    }
  }
}
```

### 1.3 — Understand What You Just Configured

**GitHub MCP server** (`@modelcontextprotocol/server-github`):
- Launched via `npx -y` — downloaded automatically on first use
- Reads your PAT from the environment (never hardcoded)
- `autoApprove` covers read-only operations — the agent can call these without pausing to ask you
- Write operations like `create_pull_request` or `create_issue` are NOT in `autoApprove` — the agent will ask you to confirm before taking those actions

**Filesystem MCP server** (`@modelcontextprotocol/server-filesystem`):
- Gives the agent access to read and write files in `/tmp`
- Scoped to `/tmp` only — the agent cannot access arbitrary paths on your machine
- `autoApprove` covers reads; write operations will prompt for confirmation

### 1.4 — Reload MCP in Kiro IDE

Open the Command Palette:
- macOS: `Cmd + Shift + P`
- Windows/Linux: `Ctrl + Shift + P`

Type: `Kiro: Reload MCP Servers` → press Enter

Wait 5–10 seconds. The first time you load the GitHub server, `npx` will download the package — this takes 15–30 seconds depending on your connection.

### 1.5 — Verify Both Servers Are Active

In the Kiro chat panel, type:

```
What MCP tools do I have available right now?
```

You should see a list that includes tools from both servers. GitHub tools will include names like `get_file_contents`, `list_commits`, `get_pull_request`. Filesystem tools will include `read_file`, `list_directory`, `write_file`.

If the list is empty or shows only one server, open the Kiro output panel (View → Output → select "Kiro MCP") to see error logs.

**Common issues at this step:**

| Symptom | Fix |
|---|---|
| `GITHUB_PERSONAL_ACCESS_TOKEN is not set` | Run `export GITHUB_PERSONAL_ACCESS_TOKEN="..."` in the terminal where Kiro was launched, then reload |
| Server listed but no tools visible | Wait 30 seconds and retry — npx may still be downloading |
| `command not found: npx` | Install Node.js 18+ first |
| GitHub tools visible but all calls fail with 401 | Token expired or wrong permissions — regenerate in Step 0 |

---

## Part 2 — First GitHub MCP Queries

Before using MCP inside an agent, test it directly from the chat panel.

### 2.1 — Repository Information

In the Kiro chat panel:

```
Using the GitHub MCP server, find my repository named [YOUR-REPO-NAME] 
owned by [YOUR-GITHUB-USERNAME] and tell me:
- The default branch
- The number of open issues
- The last 3 commit messages on the default branch
```

Replace `[YOUR-REPO-NAME]` and `[YOUR-GITHUB-USERNAME]` with your actual values.

The agent calls `search_repositories` and `list_commits` via MCP. You receive live data from GitHub — not from the agent's training data.

### 2.2 — File Contents from GitHub

```
Using the GitHub MCP server, fetch the contents of the README.md file 
from the main branch of my [YOUR-REPO-NAME] repository and summarize 
what the project does in 2 sentences.
```

The agent calls `get_file_contents` → receives the raw file content → summarizes it.

### 2.3 — Understand the Difference

Ask the same question without MCP:

```
Without using any MCP tools, what is in the README.md of my training repository?
```

The agent will either say it doesn't know or hallucinate an answer. This contrast is the core demonstration — MCP gives the agent access to real, current data that it otherwise cannot see.

---

## Part 3 — Create a GitHub-Aware Code Review Agent

### 3.1 — Create the Agents Directory

```bash
mkdir -p .kiro/agents
```

### 3.2 — Create the Agent Definition

Create `.kiro/agents/pr-reviewer.json`:

```json
{
  "name": "pr-reviewer",
  "description": "Reviews GitHub pull requests using live PR data from the GitHub MCP server. Reads the actual diff and comments with specific, line-aware feedback.",
  "tools": [
    "read_file",
    "list_directory"
  ],
  "mcpServers": [
    "github"
  ],
  "systemPrompt": "You are a senior Java developer performing code reviews on pull requests. When invoked, use the GitHub MCP server to fetch the PR details and diff. Structure every review as follows:\n\n## PR Summary\nWhat does this PR do? (1 paragraph based on the diff, not the description)\n\n## Issues Found\nFor each issue:\n- **File:** filename\n- **Severity:** CRITICAL | MAJOR | MINOR\n- **Issue:** description of the problem\n- **Fix:** concrete suggestion\n\n## What Was Done Well\nPositive observations about the code quality, testing, or design.\n\n## Verdict\nOne of: APPROVED | APPROVED WITH COMMENTS | CHANGES REQUIRED\n\nAlways fetch live PR data from GitHub MCP — never guess or fabricate PR content."
}
```

### 3.3 — Reload and Confirm Registration

In the Kiro chat panel:

```
List my available custom agents.
```

You should see `pr-reviewer` in the list.

### 3.4 — Test the Agent

If your training repository has an open pull request, use it. If not, create one:

```bash
# Create a test branch with a small change
git checkout -b feature/test-pr-review
echo "// Test change for MCP lab" >> src/main/resources/application.yml
git add .
git commit -m "test: add comment for MCP PR review exercise"
git push origin feature/test-pr-review
```

Then open a pull request on GitHub through the web UI (base: `main`, compare: `feature/test-pr-review`). Copy the PR number from the URL.

Now invoke the agent:

```
@pr-reviewer Please review PR #[PR-NUMBER] in the repository 
[YOUR-GITHUB-USERNAME]/[YOUR-REPO-NAME]
```

The agent:
1. Calls `get_pull_request` → fetches PR title, description, changed files
2. Calls `get_file_contents` → reads the diff
3. Produces a structured review in the format defined in its system prompt

---

## Part 4 — Filesystem MCP: Save Agent Output

The filesystem MCP server lets the agent write its output directly to a file — useful for persisting review reports, generating documentation, and creating artifacts without copy-pasting.

### 4.1 — Ask the Agent to Save a Report

In the Kiro chat panel (not using the custom agent this time — using the general chat):

```
Using the GitHub MCP server, fetch the last 5 commits from the main branch 
of [YOUR-GITHUB-USERNAME]/[YOUR-REPO-NAME].

Then using the filesystem MCP server, write a commit summary report to 
/tmp/commit-report.md in this format:

# Commit Report — [today's date]

## Recent Commits

| Hash | Message | Author |
|------|---------|--------|
| ... | ... | ... |

## Observations
Any patterns you notice in the commit messages (e.g., missing prefixes, 
inconsistent style, large gaps in time).
```

The agent chains two MCP servers: GitHub to fetch commits, then filesystem to write the file.

### 4.2 — Verify the Output

```bash
cat /tmp/commit-report.md
```

The file should contain real commit data from your repository formatted as a Markdown table.

### 4.3 — Copy to Your Repo

```bash
cp /tmp/commit-report.md docs/commit-report.md
git add docs/commit-report.md
git commit -m "docs: add MCP-generated commit report"
```

---

## Exercise — Full MCP-Powered Review Workflow (30 minutes)

This exercise ties everything together. You will use MCP to perform a real, end-to-end review workflow without manually copying any data.

### Scenario

Your team has a convention that all Java service classes must:
1. Have a class-level Javadoc comment
2. Use constructor injection (not field injection with `@Autowired`)
3. Not have hardcoded string literals (URLs, paths, config values)
4. Log at the start of every public method

Your task is to use MCP to verify that the most recently changed Java file in your training repository follows these conventions.

### Step-by-step

**Step E.1** — Ask the agent to identify the most recently changed file:

```
Using the GitHub MCP server, look at the last commit on the main branch 
of [YOUR-GITHUB-USERNAME]/[YOUR-REPO-NAME].

Which Java files were changed in that commit? 
List each file name and the type of change (added, modified, deleted).
```

**Step E.2** — Fetch and review the file:

```
Using the GitHub MCP server, fetch the contents of [FILENAME-FROM-ABOVE] 
from the main branch.

Review it against these four conventions:
1. Class-level Javadoc comment present
2. Constructor injection used (not @Autowired on fields)
3. No hardcoded string literals
4. Logging at the start of every public method

For each convention, state PASS or FAIL and explain why.
```

**Step E.3** — Save the review to a file:

```
Using the filesystem MCP server, write the review you just produced to 
/tmp/convention-review.md

Include the filename reviewed, the date, and all four convention checks.
```

**Step E.4** — Check the file:

```bash
cat /tmp/convention-review.md
```

**Step E.5** — If any conventions failed, ask for a fix:

```
For each FAIL item in the convention review, show me the corrected 
version of that specific code section. Do not rewrite the whole file — 
only show the corrected snippets.
```

**Step E.6** — Commit the review document:

```bash
cp /tmp/convention-review.md docs/convention-review.md
git add docs/convention-review.md
git commit -m "docs: add MCP-generated convention review for [FILENAME]"
```

### Completion Checklist

- [ ] GitHub PAT created with read-only permissions and set in environment
- [ ] `.kiro/settings/mcp.json` created with both servers
- [ ] Both servers visible when asking "what MCP tools do I have"
- [ ] GitHub MCP returned live commit data in Part 2
- [ ] `pr-reviewer` agent created and registered
- [ ] Agent reviewed a real PR (or test PR) with live GitHub data
- [ ] Filesystem MCP wrote at least one file to `/tmp`
- [ ] Exercise completed: convention review in `/tmp/convention-review.md`
- [ ] Review document committed to repo

---

## Security Cleanup After the Lab

When the workshop ends, revoke your PAT immediately:

1. Go to `https://github.com/settings/tokens`
2. Find `kiro-mcp-lab`
3. Click **Delete**

Remove the environment variable from your shell config:

```bash
# Remove the export line you added to ~/.zshrc or ~/.bashrc
# Then reload:
source ~/.zshrc
```

Remove the PAT from your current session:

```bash
unset GITHUB_PERSONAL_ACCESS_TOKEN
```

The `.kiro/settings/mcp.json` file is safe to commit — it uses `${GITHUB_PERSONAL_ACCESS_TOKEN}` (a reference, not the value). Verify before committing:

```bash
grep "GITHUB_PERSONAL_ACCESS_TOKEN" .kiro/settings/mcp.json
# Must show: "${GITHUB_PERSONAL_ACCESS_TOKEN}"
# Must NOT show: "github_pat_..."
```

---

## Reference — Full `mcp.json` for This Lab

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
      },
      "disabled": false,
      "autoApprove": [
        "get_file_contents",
        "list_commits",
        "search_repositories",
        "get_pull_request",
        "list_pull_requests"
      ]
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/tmp"
      ],
      "disabled": false,
      "autoApprove": [
        "read_file",
        "list_directory",
        "search_files"
      ]
    }
  }
}
```

## Reference — Tools Exposed by Each Server

### GitHub MCP Server

| Tool | What it does | Auto-approved |
|---|---|---|
| `search_repositories` | Find repos by name/owner | Yes |
| `get_file_contents` | Read a file from any branch | Yes |
| `list_commits` | Get commit history | Yes |
| `get_pull_request` | Fetch PR details and diff | Yes |
| `list_pull_requests` | List open/closed PRs | Yes |
| `create_pull_request` | Open a new PR | **No — requires confirmation** |
| `create_issue` | Open a new issue | **No — requires confirmation** |
| `create_or_update_file` | Write a file to GitHub | **No — requires confirmation** |

### Filesystem MCP Server

| Tool | What it does | Auto-approved |
|---|---|---|
| `read_file` | Read a file in the allowed path | Yes |
| `list_directory` | List directory contents | Yes |
| `search_files` | Find files by name pattern | Yes |
| `write_file` | Write content to a file | **No — requires confirmation** |
| `create_directory` | Create a directory | **No — requires confirmation** |
| `delete_file` | Delete a file | **No — requires confirmation** |