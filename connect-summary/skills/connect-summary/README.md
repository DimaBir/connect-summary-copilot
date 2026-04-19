# Connect Summary — Copilot Skill

A GitHub Copilot CLI skill that generates a structured **Connect performance review summary** from your Azure DevOps contributions.

## What It Does

Analyzes your ADO data and drafts a Connect-ready summary with three sections:
- 🏆 **Impact Delivered** — themed groups of your contributions with evidence links
- 💡 **Learnings & Growth** — challenges encountered and expertise developed
- 🤝 **Collaboration & Future Goals** — code review activity and growth areas

## Quick Start

Invoke the skill naturally, or explicitly with `/connect-summary`:

```
Use the /connect-summary skill to generate my Connect summary since 2025-01-01
```

or just:

```
Generate my Connect summary since 2025-01-01
```

The skill automatically detects the best way to access your ADO data.

## Installation

Copy the `connect-summary/` folder into a Copilot skills directory:

```powershell
# Option 1: Project-level (share with your team) — commit to your repo
Copy-Item -Recurse connect-summary <your-repo>\.github\skills\connect-summary

# Option 2: Agents-level (alternate location supported by Copilot CLI)
Copy-Item -Recurse connect-summary <your-repo>\.agents\skills\connect-summary

# Option 3: User-level (personal, all repos)
Copy-Item -Recurse connect-summary $HOME\.copilot\skills\connect-summary
```

## Parameters

| Parameter | Required | Default | Example |
|-----------|----------|---------|---------|
| `since_date` | **Yes** | — | `2025-01-01`, `January 2025`, `last 6 months` |
| `org` | **Yes** | — | `myorg` |
| `project` | **Yes** | — | `MyProject` |

## Example Prompts

```
# Basic
Generate my Connect summary since 2025-01-01 org myorg project MyProject

# Custom project
Connect summary since 2024-07-01 org contoso project SecurityPlatform

# Natural language date
Summarize my ADO contributions for the last 6 months in org myorg project MyProject

# Full form
Help me prepare for Connect — show my code contributions since March 2025 in org myorg project MyProject
```

## How Data Is Fetched

The skill uses a **tiered fallback strategy** — it auto-detects what's available and uses the best option. No single tool is required.

| Priority | Method | What You Need |
|----------|--------|---------------|
| 1️⃣ | **ADO MCP Tools** (PRs) | ADO MCP configured in your Copilot CLI ([setup guide](https://github.com/microsoft/azure-devops-mcp)) |
| 2️⃣ | **`az devops` CLI** (work items via WIQL) | Azure CLI + DevOps extension installed (`az extension add --name azure-devops`) + `az login` |
| 3️⃣ | **REST API + PAT** | A PAT with Code (Read) + Work Items (Read) scopes — set as `$env:ADO_PAT` locally (preferred over pasting) |
| 4️⃣ | **Manual Input** | Nothing — paste **URLs** of your PRs and work items |

The skill uses **different sources for different queries** (e.g., MCP for PRs but CLI for work items) when that gives more reliable, date-accurate results.

### Which method will be used?

The skill tries each method in order and uses the first one that works. It tells you which method it's using:

```
✅ Using ADO MCP tools (directly connected)
```
or
```
✅ Using Azure DevOps CLI
```
or
```
I don't have direct ADO access. Could you provide a PAT? ...
```

## Sample Output

The skill only uses facts visible in your PR/work item titles. It will **not fabricate metrics, mentoring claims, or cross-team impact** — if titles are too vague, it will ask you for 2–3 impact bullets first. A typical output looks like:

```markdown
# 🎯 Connect Performance Summary

**Period**: 2025-01-01 — Present | **Project**: myorg/MyProject
**PRs Created**: 23 | **PRs Reviewed**: 47 | **Work Items**: 15

---

## 🏆 1. Impact Delivered

**Alert Correlation Engine Refactor**
Refactored the alert correlation module across multiple PRs touching scoring and dedup logic.
> 📌 Evidence: [WI #12345](…), [PR #6789](…), [PR #6790](…)

**CI Security Scanning Integration**
Added SAST pipeline stages and pre-commit hooks in the build repo.
> 📌 Evidence: [WI #12350](…), [PR #6800](…)

## 💡 2. Learnings & Growth

- Multiple "fix race condition" PRs in the async processor surfaced concurrency gaps;
  follow-up PRs added retries and integration tests.
- Work visible in Kubernetes-related PRs suggests deepening expertise in container networking.

## 🤝 3. Collaboration & Future Goals

**Collaboration:**
- Reviewed 47 PRs across the repositories visible in the data.

**Future Goals:**
- Continue work in the areas represented above (to be refined by the engineer).
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "No PRs found" | Try a wider date range, or check that org/project are correct |
| ADO MCP not detected | Ensure MCP server is configured in your Copilot CLI settings |
| `az devops` not found | Run `az extension add --name azure-devops` and `az login` |
| PAT authentication fails | Ensure your PAT has Code (Read) and Work Items (Read) scopes. Set it as `$env:ADO_PAT` locally rather than pasting into chat. Revoke the PAT when done |
| Too many results | The skill auto-summarizes large datasets; try narrowing the date range |

## How It Works (Architecture)

```
User prompt → SKILL.md instructions → Copilot detects ADO access method
                                         ↓
                              Fetches PRs + Work Items
                                         ↓
                              Copilot model generates summary
                                         ↓
                              Markdown output in chat
```

- **No external AI** — uses the Copilot model itself (no Azure OpenAI)
- **No server** — pure skill file (SKILL.md), no MCP server or GitHub App
- **No email** — renders directly in Copilot Chat as formatted markdown

## Origin

Adapted from the "Code Contribution Summary for Connect" Power Automate flow, which used Azure OpenAI (gpt-4.1-nano) and Office 365 email. This skill replaces that entire flow with a single markdown file.
