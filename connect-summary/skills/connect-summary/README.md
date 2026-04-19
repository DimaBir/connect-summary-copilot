# Connect Summary Skill (Microsoft)

A GitHub Copilot CLI skill that drafts your **Connect** performance review by pulling your PRs and work items from Microsoft's internal Azure DevOps (`dev.azure.com/microsoft`).

It is **hardcoded to the `microsoft` ADO org** — you only need to pass `since_date` and `project`. Auth uses your **Entra ID** via Azure CLI as the primary path, with a **PAT fallback** for environments where `az` cannot be used.

## What it does

Given a start date and a project, the skill queries ADO for:
- PRs you **created** (completed, since date)
- PRs you **reviewed** (completed, since date — excluding your own)
- Work items **assigned to you** that were closed/completed/resolved/done since the date

…and then generates a structured Connect summary (Impact / Learnings / Collaboration) grouped into themes, with a clickable-links appendix of every PR and work item.

**It never fabricates metrics** — if titles are vague it will ask you for 2–3 impact bullets in your own words.

---

## Install

You need [GitHub Copilot CLI](https://github.com/github/copilot-cli) installed.

1. From the repo root (the `playground` clone), copy `SKILL.md` into your Copilot skills folder:

   **Windows (PowerShell):**
   ```powershell
   $dest = "$env:USERPROFILE\.copilot\skills\connect-summary"
   New-Item -ItemType Directory -Force -Path $dest | Out-Null
   Copy-Item .\plugins\connect-summary\skills\connect-summary\SKILL.md $dest\SKILL.md -Force
   ```

   **macOS / Linux:**
   ```bash
   mkdir -p ~/.copilot/skills/connect-summary
   cp ./plugins/connect-summary/skills/connect-summary/SKILL.md ~/.copilot/skills/connect-summary/SKILL.md
   ```

2. Restart Copilot CLI (`copilot`) so the skill is picked up.

That's it — no plugin install, no marketplace.

---

## One-time setup: `az login`

The skill authenticates to Microsoft ADO using your **Entra ID** token via Azure CLI. No PAT, no secret on disk.

1. Install Azure CLI if you don't have it: <https://learn.microsoft.com/cli/azure/install-azure-cli>
2. Sign in with your Microsoft corp account:
   ```powershell
   az login
   ```
   A browser opens — sign in as `you@microsoft.com`.
3. Verify:
   ```powershell
   az account show --query user.name
   # → you@microsoft.com
   ```

That's it. Tokens are short-lived (~1 hour) and refreshed automatically — there's nothing to manage or revoke.

> 🔒 **Security note:** Auth is tied to your corp account. Conditional Access, MFA, and account-disable apply automatically. Nothing sensitive is stored on disk.

### Fallback: PAT (only if you can't use `az`)

If you're in an environment where Azure CLI isn't available, you can fall back to a PAT — see the **"Fallback: PAT"** section at the bottom of `SKILL.md`.

---

## Usage

In Copilot CLI:

```
/connect-summary since 2026-01-01 project WDATP
```

Other examples:

```
Generate my Connect summary since March 2026, project OS
Connect summary for the last 6 months, project Edge
```

The skill will:
1. Acquire an Entra ID token (`az account get-access-token`).
2. Resolve your ADO identity.
3. Page through repos and collect PRs + work items.
4. Show you the counts, then generate the summary.

---

## Output

You'll get a Markdown document with three sections:
- 🏆 **Impact Delivered** — 2–4 themes grouped from your PRs/work items
- 💡 **Learnings & Growth**
- 🤝 **Collaboration & Future Goals**

…followed by an **Appendix** with every PR and work item as a clickable link, so you can verify and edit.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Could not get Entra ID token` | Run `az login` and try again. |
| `az: command not found` | Install Azure CLI, or use the PAT fallback in `SKILL.md`. |
| `authenticatedUser.id` is empty | Token is invalid — `az logout` then `az login` again. |
| 401 mid-run | Token expired — re-run `az account get-access-token`. |
| 404 on `/_apis/projects/…` | Check the project name spelling. Must match exactly (e.g., `WDATP`, `OS`, `Edge`). |
| HTTP 429 | Rate-limited — wait a minute and retry. |

---

## What it does NOT do

- ❌ Does not post anything back to ADO.
- ❌ Does not read comments, commits, or wiki pages.
- ❌ Does not fabricate numbers or claims — if your titles are vague it will ask you for context.
- ❌ Does not support non-Microsoft ADO orgs (hardcoded to `microsoft` — fork and change the `$org` line if you need to).

---

## Files

```
plugins/
└── connect-summary/
    └── skills/
        └── connect-summary/
            ├── SKILL.md    # The skill itself — copy to ~/.copilot/skills/connect-summary/
            └── README.md   # This file
```

---

## License

Share freely within Microsoft. No warranty.
