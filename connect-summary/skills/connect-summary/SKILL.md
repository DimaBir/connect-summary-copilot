---
name: connect-summary
description: "Generate a Connect performance review summary from your Microsoft Azure DevOps contributions (PRs created, PRs reviewed, completed work items). Hardcoded for Microsoft org (dev.azure.com/microsoft). Uses Entra ID via Azure CLI as the primary auth path, with PAT fallback."
---

# Connect Performance Review Summary Generator (Microsoft)

Generate a structured Connect performance review summary by analyzing your Azure DevOps contributions on **Microsoft's internal ADO** (`dev.azure.com/microsoft`).

## When to Use This Skill

Use this skill when the user asks to:
- Generate a Connect summary
- Prepare for a Connect performance review
- Summarize their ADO contributions
- Draft a performance review based on PRs and work items

## Input Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `since_date` | **Yes** | — | Start date in YYYY-MM-DD format (e.g., 2026-01-01) |
| `project` | **Yes** | — | Microsoft ADO project name (e.g., `WDATP`, `OS`, `Edge`) |

**Org is hardcoded to `microsoft`** — `https://dev.azure.com/microsoft`.

If the user does not provide `since_date` or `project`, ask for them before proceeding.

## Authentication — Entra ID (Primary) with PAT Fallback

This skill uses **Entra ID tokens via Azure CLI** as the primary auth path. The user only needs to be logged in to `az` with their Microsoft corp account — no PAT to create, no secret to manage.

**Why Entra ID is preferred:**
- No long-lived secret on disk
- Tied to corp account lifecycle (Conditional Access, MFA, account disable all apply automatically)
- Nothing to revoke when finished
- Works for everyone in the Microsoft tenant out of the box

**Prerequisite (one-time):**
```powershell
az login                         # opens browser, sign in with @microsoft.com
az account show --query user.name   # confirm you're signed in as you@microsoft.com
```

**Token lifetime:** ~1 hour. The skill re-fetches per session as needed.

If `az` is not installed or `az login` has not been run, the skill falls back to PAT (see "Fallback: PAT" at the bottom of this file).

## Workflow

Execute in order. Report progress after each major step.

### Step 0 — Acquire Entra ID Token

```powershell
# 499b84ac-1321-427f-aa17-267ca6975798 is the well-known Azure DevOps resource ID — do not change.
$adoResource = "499b84ac-1321-427f-aa17-267ca6975798"
$token = az account get-access-token --resource $adoResource --query accessToken -o tsv 2>$null
if (-not $token) {
    throw "Could not get Entra ID token. Run 'az login' first, or set `$env:ADO_PAT to use the PAT fallback."
}
$headers = @{
    Authorization = "Bearer $token"
    Accept        = "application/json"
}
$org      = "microsoft"
$project  = "{project}"          # from user input
$since    = "{since_date}"       # from user input, YYYY-MM-DD
$sinceIso = ([DateTime]$since).ToUniversalTime().ToString("o")
$nowIso   = (Get-Date).ToUniversalTime().ToString("o")
```

> 🔄 **PAT fallback:** If `az` is unavailable, use a PAT instead — see "Fallback: PAT" at the end. The only line that changes is `$headers`; everything below is identical.

### Step 1 — Resolve User Identity

```powershell
$me = (Invoke-RestMethod -Uri "https://vssps.dev.azure.com/$org/_apis/connectionData?api-version=7.1-preview" -Headers $headers).authenticatedUser
$userId   = $me.id                       # GUID for REST searchCriteria
$userMail = $me.properties.Mail.'$value' # e.g. user@microsoft.com
```

If `$userId` is empty, the token is invalid — ask the user to re-run `az login`.

### Step 2 — PRs Created by User

PRs are repo-scoped. List repos, then page through each.

```powershell
$repos = (Invoke-RestMethod -Uri "https://dev.azure.com/$org/$project/_apis/git/repositories?api-version=7.1" -Headers $headers).value

# Server-side time filter via queryTimeRangeType=closed + minTime/maxTime ⇒ ADO returns only PRs closed in [since, now].
$allCreated = @()
foreach ($repo in $repos) {
    $skip = 0
    do {
        $uri  = "https://dev.azure.com/$org/$project/_apis/git/repositories/$($repo.id)/pullrequests?searchCriteria.status=completed&searchCriteria.creatorId=$userId&searchCriteria.minTime=$sinceIso&searchCriteria.maxTime=$nowIso&searchCriteria.queryTimeRangeType=closed&`$top=100&`$skip=$skip&api-version=7.1"
        $page = (Invoke-RestMethod -Uri $uri -Headers $headers).value
        $allCreated += $page
        $skip += 100
    } while ($page.Count -eq 100)
}
$createdPRs = $allCreated
```

Collect per PR: `title`, `pullRequestId`, `repository.name`, URL `https://dev.azure.com/$org/$project/_git/<repo>/pullrequest/<id>`.

### Step 3 — PRs Reviewed by User

Same pattern, with `reviewerId` instead of `creatorId`:

```powershell
$allReviewed = @()
foreach ($repo in $repos) {
    $skip = 0
    do {
        $uri  = "https://dev.azure.com/$org/$project/_apis/git/repositories/$($repo.id)/pullrequests?searchCriteria.status=completed&searchCriteria.reviewerId=$userId&searchCriteria.minTime=$sinceIso&searchCriteria.maxTime=$nowIso&searchCriteria.queryTimeRangeType=closed&`$top=100&`$skip=$skip&api-version=7.1"
        $page = (Invoke-RestMethod -Uri $uri -Headers $headers).value
        $allReviewed += $page
        $skip += 100
    } while ($page.Count -eq 100)
}
$reviewedPRs = $allReviewed | Where-Object { $_.createdBy.id -ne $userId }   # exclude own PRs
```

### Step 4 — Completed Work Items (WIQL)

# Captures items the user owned (AssignedTo = @Me) that landed in a terminal state
# during the review window. ClosedDate is set on most Microsoft process templates
# (Agile, Scrum, CMMI). ChangedDate is included as a guard for templates that don't
# populate ClosedDate. This is the standard "what I shipped" view used for Connect.
```powershell
$wiql = @"
SELECT [System.Id] FROM WorkItems
WHERE [System.AssignedTo] = @Me
  AND [System.State] IN ('Closed','Completed','Resolved','Done')
  AND ( [Microsoft.VSTS.Common.ClosedDate] >= '$since'
        OR [System.ChangedDate]            >= '$since' )
ORDER BY [System.ChangedDate] DESC
"@
$body = @{ query = $wiql } | ConvertTo-Json
$ids  = (Invoke-RestMethod -Method POST -Uri "https://dev.azure.com/$org/$project/_apis/wit/wiql?api-version=7.1" -Headers $headers -ContentType "application/json" -Body $body).workItems.id

$items = @()
for ($i = 0; $i -lt $ids.Count; $i += 200) {
    $batch  = $ids[$i..([Math]::Min($i+199, $ids.Count-1))] -join ','
    $fields = 'System.Id,System.Title,System.WorkItemType,System.State,Microsoft.VSTS.Common.ClosedDate'
    $items += (Invoke-RestMethod -Uri "https://dev.azure.com/$org/_apis/wit/workitems?ids=$batch&fields=$fields&api-version=7.1" -Headers $headers).value
}
```

Build URL per item: `https://dev.azure.com/$org/$project/_workitems/edit/<id>`.

### Step 5 — Present Data Summary

```markdown
## 📊 Data Collected

| Category | Count |
|----------|-------|
| PRs Created | {n} |
| PRs Reviewed | {n} |
| Work Items Completed | {n} |

**Period**: {since_date} → today
**Project**: microsoft/{project}
**Auth**: Entra ID (signed in as {userMail})

Generating your Connect summary...
```

### Step 6 — Generate Connect Summary

You (the AI model) generate the summary directly — do not call any external AI. Use the template + rules below.

---

## Connect Summary Prompt Template

You are an expert mentor helping a Microsoft engineer draft their "Connect" performance review. Generate a compelling, evidence-based summary from the data collected.

### Rules — Strict Anti-Fabrication

1. **Every ID must be a clickable link** — work items `[WI #{id}](url)`, PRs `[PR #{id}](url)`.
2. **Do NOT fabricate metrics or claims.** Never invent numbers (e.g., "reduced false positives by 30%"), never claim mentoring, cross-team work, or customer outcomes unless stated in the data.
3. **Observable over inferred.** If impact isn't explicit in a title, describe the observable change only.
4. **Low-signal guard.** If PR/work item titles are vague ("fix bug", "update tests"), STOP and ask the user for 2–3 impact bullets in their own words before finalizing.
5. **Impact over activity** — when data supports it, prefer impact framing over raw counts.
6. **Group related work** into 2–4 themes by keyword/component.
7. **Be specific** — name technologies and components that appear in titles.
8. **Be concise** — each section 3–5 bullets max.

### Output Format

```markdown
# 🎯 Connect Performance Summary

**Period**: {since_date} — Present | **Project**: microsoft/{project}
**PRs Created**: {n} | **PRs Reviewed**: {n} | **Work Items**: {n}

---

## 🏆 1. Impact Delivered

**[Theme 1 — e.g., "Improved Detection Pipeline Reliability"]**
[What was done] which resulted in [observable change / value].
> 📌 Evidence: [WI #123](link), [PR #456](link), [PR #789](link)

**[Theme 2]**
[Description].
> 📌 Evidence: [WI #234](link), [PR #567](link)

## 💡 2. Learnings & Growth

- While working on [topic], encountered [challenge] — reinforced [lesson].
- Deepened expertise in [tech visible from titles].

## 🤝 3. Collaboration & Future Goals

**Collaboration:**
- Reviewed {n} PRs across [repos], helping peers maintain code quality.

**Future Goals:**
- Deepen expertise in [tech visible from contributions].
- Drive [area based on work type].

---

### Appendix — Raw Data

#### Work Items Completed
| ID | Title | Type | Link |
|----|-------|------|------|
| … | … | … | [Link](url) |

#### PRs Created
| # | Title | Repo | Link |
|---|-------|------|------|
| … | … | … | [Link](url) |

#### PRs Reviewed
| # | Title | Repo | Link |
|---|-------|------|------|
| … | … | … | [Link](url) |
```

## Edge Cases

- **No data**: Tell the user and suggest a wider date range.
- **>100 items per category**: Summarize themes; put the full table in a collapsible `<details>` block.
- **Date formats**: Accept `2026-01-01`, `Jan 2026`, "last 6 months" — normalize to YYYY-MM-DD.
- **Multiple projects**: Run Steps 2–4 per project, combine results, note per-project counts.
- **Rate limits**: If you hit HTTP 429, `Start-Sleep -Seconds 5` and retry.
- **Vague titles**: See Rule 4 — stop and ask.
- **Token expired (401 mid-run)**: Re-run `az account get-access-token` and refresh `$headers`.
- **`az` not installed / not logged in**: Drop to the PAT fallback below.

## Example Invocations

- "Generate my Connect summary since 2026-01-01 project WDATP"
- "Connect summary since March 2026 project OS"
- "Prepare my Connect review for the last 6 months, project Edge"

---

## Fallback: PAT (only if `az` is unavailable)

If `az login` cannot be used (e.g., automation environment, no Azure CLI), fall back to a Personal Access Token:

1. https://dev.azure.com/microsoft/_usersSettings/tokens → **New Token**
   - Name: `connect-summary`, Expiration: 90 days
   - Scopes: **Code (Read)** + **Work Items (Read)**
2. ```powershell
   $env:ADO_PAT = "<paste-token-here>"
   ```
3. Replace Step 0's `$headers` block with:
   ```powershell
   $headers = @{
       Authorization = "Basic $([Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$env:ADO_PAT")))"
       Accept        = "application/json"
   }
   ```

Everything else in the workflow is identical. Revoke the PAT at the same URL when finished.
