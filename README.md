# connect-summary-copilot

A distributable GitHub Copilot CLI plugin that generates a structured **performance-review summary** from your Azure DevOps contributions — PRs created, PRs reviewed, and completed work items — with zero dependency on Azure OpenAI. The Copilot model itself writes the summary.

## What it does

When you invoke `/connect-summary since YYYY-MM-DD`, the skill:

1. Auto-detects which Azure DevOps access method is available in your environment (ADO MCP tools → `az` CLI → bearer-token REST → PAT REST → manual paste).
2. Fetches completed PRs you authored, PRs you reviewed, and closed work items assigned to you since the given date.
3. Generates a structured markdown summary with three sections: **Impact Delivered**, **Learnings & Growth**, **Collaboration & Future Goals** — with clickable links for every PR and work item ID.
4. Has strict anti-fabrication rules: never invents metrics, stops to ask if titles are too vague to infer impact.

## Included skills

| Skill | Description |
|-------|-------------|
| `connect-summary` | Generate a Connect performance review summary from ADO contributions |

## Requirements

- **Azure DevOps access** via any one of:
  - ADO MCP server configured in Copilot CLI (best), or
  - `az` CLI with `az login` (bearer-token REST fallback works even when `az boards` keyring is broken), or
  - A Personal Access Token exported as `$env:ADO_PAT` (scopes: Code Read + Work Items Read), or
  - Manual paste of PR / work-item URLs.
- **Org and project** are **required** — pass them with every invocation (e.g. `org myorg project MyProject`). If your project uses the **Basic** process template, see the note in `SKILL.md` about substituting `System.ChangedDate` for `Microsoft.VSTS.Common.ClosedDate` in the WIQL.

## Install

Install directly from GitHub (public repo — no auth or collaborator access required):

```bash
copilot plugin install DimaBir/connect-summary-copilot
```

> **Hit `EBUSY: resource busy or locked` on install/update?** Another Copilot CLI session or file-indexer is holding the marketplace cache dir open. Fix:
> ```powershell
> # Close all other `copilot` sessions first, then:
> Remove-Item -Recurse -Force "$env:LOCALAPPDATA\copilot\marketplaces\DimaBir--connect-summary-copilot"
> copilot plugin install DimaBir/connect-summary-copilot
> ```

Verify it loaded:

```bash
copilot plugin list
```

You should see `connect-summary-copilot` with the `connect-summary` skill.

### Updating

```bash
copilot plugin update connect-summary-copilot
```

### Uninstalling

```bash
copilot plugin uninstall connect-summary-copilot
```

### Local / dev install

Clone anywhere and point Copilot at it:

```bash
git clone https://github.com/DimaBir/connect-summary-copilot.git
copilot --plugin-dir ./connect-summary-copilot
```

## Usage

```
/connect-summary since 2026-03-01 org myorg project MyProduct
/connect-summary since 2025-07-01 org contoso project Platform
```

## License / Attribution

Personal-use plugin. Contains no secrets or PII. Pass your own `org` and `project` on every call.
