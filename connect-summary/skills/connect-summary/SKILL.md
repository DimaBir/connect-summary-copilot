---
name: connect-summary
description: "Generate a Connect performance review summary from your Azure DevOps contributions (PRs created, PRs reviewed, completed work items). Auto-detects available ADO tools and works with any setup."
---

# Connect Performance Review Summary Generator

Generate a structured Connect performance review summary by analyzing your Azure DevOps contributions.

## When to Use This Skill

Use this skill when the user asks to:
- Generate a Connect summary
- Prepare for a Connect performance review
- Summarize their ADO contributions
- Draft a performance review based on PRs and work items

## Input Parameters

Parse these from the user's request:

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `since_date` | **Yes** | — | Start date in YYYY-MM-DD format (e.g., 2025-01-01) |
| `org` | **Yes** | — | Azure DevOps organization name (ask the user if not provided) |
| `project` | **Yes** | — | Azure DevOps project name (ask the user if not provided) |

> **Portability note**: The WIQL queries below use `Microsoft.VSTS.Common.ClosedDate`, which is the public ADO field name available in Agile/Scrum/CMMI process templates. If the user's project uses the **Basic** process template, substitute `System.ChangedDate` (less precise but universally available).

If the user does not provide a `since_date`, `org`, or `project`, ask for each one before proceeding. For `since_date`, suggest "the start of the current review period" as guidance.

## Workflow

Execute these steps **in order**. Report progress to the user after each major step.

**Important**: Data source selection is **per query type**, not one-source-for-all. For example, PRs may come from MCP while work items use CLI (because WIQL gives reliable date-filtered history that `ado-wit_my_work_items` does not).

### Step 1 — Detect Available Data Sources

Probe each method once and record what's available. Use the best-available method for each query type in Steps 2–4.

**Probe 1 — ADO MCP Tools** (best for PRs):
- Try calling `ado-repo_list_pull_requests_by_repo_or_project` with `project` and `top: 1`
- If it returns (even empty array), mark `MCP_AVAILABLE = true`

**Probe 2 — `az devops` CLI** (best for work items via WIQL):
- Run via PowerShell: `az devops project show --project "{project}" --org "https://dev.azure.com/{org}" --output json 2>&1`
- If this succeeds (returns JSON, no error), mark `CLI_AVAILABLE = true`
- If `az` is installed but not logged in, the user may need to run `az login` first — suggest this rather than asking for a PAT

**Probe 3 — REST API** (last-resort, needs auth):
- Only offer this if both MCP and CLI probes fail
- Two auth options, in priority order:
  - **Option A (preferred when `az login` has succeeded): bearer token from `az`** — works even when `az boards` / `az repos` are broken by keyring errors. Grab a token scoped to ADO's first-party app ID:
    ```powershell
    $token = az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query accessToken -o tsv
    $headers = @{ Authorization = "Bearer $token"; Accept = "application/json" }
    ```
    The resource GUID `499b84ac-1321-427f-aa17-267ca6975798` is the universal Azure DevOps app ID (not machine-specific). This avoids needing a PAT entirely.
  - **Option B: Personal Access Token** — ask the user to set `$env:ADO_PAT` locally, then say "done". Do NOT ask them to paste the PAT into chat. If they insist on pasting, accept it but warn about the security risk and remind them to revoke it after. Scopes required: **Code (Read)** and **Work Items (Read)**. Build headers:
    ```powershell
    $headers = @{ Authorization = "Basic $([Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$env:ADO_PAT")))" }
    ```
- Known issue on Windows: `az boards` and `az repos` sometimes fail with `ServiceResponseError ... Could not load or create the keyring`. When this happens, **skip the CLI probe outcome** and use Option A above — the bearer token path bypasses the broken keyring-dependent subcommands while still reusing your `az login`.

**Probe 4 — Manual Input** (always works):
- If all above fail or the user declines auth, ask them to paste data. Prefer URLs:
  1. PRs you created — paste **PR URLs** (one per line) from ADO, or provide repo + PR number
  2. PRs you reviewed — paste **PR URLs** (one per line)
  3. Completed work items — paste **work item URLs** (one per line), or `WI-{id}` + title + type
- URLs are strongly preferred because the skill's output requires clickable links for every ID.

Tell the user which methods are active, e.g.:
> "✅ PRs: ADO MCP • Work Items: az devops CLI"

### Step 1b — Resolve Current User Identity

REST and some CLI queries require an **identity ID** (a GUID) for `{me}`, `{userId}`. Resolve it once:

- **MCP**: use `ado-core_get_identity_ids` with the user's email/alias — returns an identity GUID
- **CLI**: `az ad signed-in-user show --query id -o tsv` (Entra object ID) or run `az devops user show --user "{email}" --org "https://dev.azure.com/{org}"` for ADO identity
- **REST**: call `https://vssps.dev.azure.com/{org}/_apis/connectionData?api-version=7.1` with auth — the response includes `authenticatedUser.id`

If identity cannot be auto-resolved, ask the user for the email associated with their ADO account (e.g., `user@example.com`) and pass it as `creator`/`reviewer` to the CLI — the CLI accepts both emails and GUIDs. For REST, you **must** have the GUID.

### Step 2 — Fetch PRs Created by User

Fetch completed pull requests **created by the user** since `since_date`.

**Via ADO MCP:**
```
ado-repo_list_pull_requests_by_repo_or_project:
  project: {project}
  created_by_me: true
  status: "Completed"
  top: 200
```

**Via az devops CLI** (PowerShell):
```powershell
az repos pr list --project "{project}" --org "https://dev.azure.com/{org}" --creator "{me}" --status completed --top 200 --output json
```

**Via REST API** (PowerShell) — PRs are **repo-scoped**, so enumerate repos first, then query each repo with pagination:
```powershell
# Headers: use whichever auth method was selected in Probe 3 (bearer token preferred, PAT as fallback)
# $headers = @{ Authorization = "Bearer $token" }                                         # Option A
# $headers = @{ Authorization = "Basic $([Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$env:ADO_PAT")))" }  # Option B

# 1. List all repositories in the project
$repos = (Invoke-RestMethod -Uri "https://dev.azure.com/{org}/{project}/_apis/git/repositories?api-version=7.1" -Headers $headers).value

# 2. For each repo, page through completed PRs by creator
$allPRs = @()
foreach ($repo in $repos) {
    $skip = 0
    do {
        $uri = "https://dev.azure.com/{org}/{project}/_apis/git/repositories/$($repo.id)/pullrequests?searchCriteria.status=completed&searchCriteria.creatorId={userId}&`$top=100&`$skip=$skip&api-version=7.1"
        $page = (Invoke-RestMethod -Uri $uri -Headers $headers).value
        $allPRs += $page
        $skip += 100
    } while ($page.Count -eq 100)
}
```

**Pagination for all methods:**
- MCP: use `skip` parameter, loop until result count < `top`
- CLI: `--skip {n}` + `--top 100`, loop similarly
- REST: `$skip` + `$top=100`, loop until page returns fewer than 100

**Post-processing:**
- Filter to only PRs where `closedDate >= since_date`
- Collect for each PR: `title`, `pullRequestId`, `repository.name`, URL (`https://dev.azure.com/{org}/{project}/_git/{repo}/pullrequest/{id}`)
- Count total PRs found

### Step 3 — Fetch PRs Reviewed by User

Fetch completed pull requests where the user was a **reviewer** since `since_date`.

**Via ADO MCP:**
```
ado-repo_list_pull_requests_by_repo_or_project:
  project: {project}
  i_am_reviewer: true
  status: "Completed"
  top: 200
```

**Via az devops CLI** (PowerShell):
```powershell
az repos pr list --project "{project}" --org "https://dev.azure.com/{org}" --reviewer "{me}" --status completed --top 200 --output json
```

**Via REST API** (PowerShell) — same repo-enumeration pattern as Step 2, but with `reviewerId`:
```powershell
$allReviews = @()
foreach ($repo in $repos) {
    $skip = 0
    do {
        $uri = "https://dev.azure.com/{org}/{project}/_apis/git/repositories/$($repo.id)/pullrequests?searchCriteria.status=completed&searchCriteria.reviewerId={userId}&`$top=100&`$skip=$skip&api-version=7.1"
        $page = (Invoke-RestMethod -Uri $uri -Headers $headers).value
        $allReviews += $page
        $skip += 100
    } while ($page.Count -eq 100)
}
```

**Post-processing:**
- Filter to only PRs where `closedDate >= since_date`
- Exclude PRs that also appear in "Created by me" list (user reviewed their own)
- Collect for each PR: `title`, `pullRequestId`, `repository.name`, URL
- Count total reviews

### Step 4 — Fetch Completed Work Items

Fetch work items **assigned to the user** that were completed/closed since `since_date`.

**For date-accurate history, prefer WIQL (via CLI or REST) over MCP.** The MCP tools `ado-wit_my_work_items` and `ado-search_workitem` are convenience wrappers without reliable date filtering and may miss or over-report items.

**Via `az devops` CLI (preferred for work items):**
```powershell
$wiql = "SELECT [System.Id] FROM WorkItems WHERE [System.AssignedTo] = @Me AND [System.State] IN ('Closed','Completed','Resolved','Done') AND [Microsoft.VSTS.Common.ClosedDate] >= '{since_date}' ORDER BY [Microsoft.VSTS.Common.ClosedDate] DESC"
$ids = (az boards query --wiql "$wiql" --project "{project}" --org "https://dev.azure.com/{org}" --output json | ConvertFrom-Json).id
```

**Via REST API** — WIQL returns IDs only; hydrate fields in a second call (batch up to 200 IDs per request):
```powershell
$body = @{ query = $wiql } | ConvertTo-Json
$ids = (Invoke-RestMethod -Method POST -Uri "https://dev.azure.com/{org}/{project}/_apis/wit/wiql?api-version=7.1" -Headers $headers -ContentType "application/json" -Body $body).workItems.id

# Hydrate fields for the returned IDs
$items = @()
for ($i = 0; $i -lt $ids.Count; $i += 200) {
    $batch = $ids[$i..([Math]::Min($i+199, $ids.Count-1))] -join ','
    $fields = 'System.Id,System.Title,System.WorkItemType,System.State,Microsoft.VSTS.Common.ClosedDate'
    $items += (Invoke-RestMethod -Uri "https://dev.azure.com/{org}/_apis/wit/workitems?ids=$batch&fields=$fields&api-version=7.1" -Headers $headers).value
}
```

For the CLI, hydrate similarly:
```powershell
$items = az boards work-item show --id $id --org "https://dev.azure.com/{org}" --output json
# or batch via: az rest --uri "https://dev.azure.com/{org}/_apis/wit/workitems?ids=..."
```

**Via ADO MCP (fallback only, when neither CLI nor REST is available):**
```
ado-wit_get_work_items_batch_by_ids:
  project: {project}
  ids: [list of ids from a prior WIQL]
  fields: ["System.Id","System.Title","System.WorkItemType","System.State","Microsoft.VSTS.Common.ClosedDate"]
```
If no WIQL is possible, fall back to `ado-wit_my_work_items` with `includeCompleted: true` and filter client-side by `ClosedDate >= since_date`. Note that this may be incomplete for long lookbacks — warn the user.

**Post-processing:**
- Collect for each work item: `id`, `title`, `workItemType`, `state`
- Build URL: `https://dev.azure.com/{org}/{project}/_workitems/edit/{id}`
- Count by type (User Story, Task, Bug, etc.)

### Step 5 — Present Data Summary

Before generating the Connect summary, show the user what was collected:

```markdown
## 📊 Data Collected

| Category | Count |
|----------|-------|
| PRs Created | {count} |
| PRs Reviewed | {count} |
| Work Items Completed | {count} |

**Period**: {since_date} to today
**Project**: {org}/{project}

Generating your Connect summary...
```

### Step 6 — Generate Connect Summary

Now generate the Connect performance review summary using the data collected. Use the prompt template below, filling in the data sections with the actual data from Steps 2-4.

**IMPORTANT**: You ARE the AI model — do not try to call an external AI. Just generate the summary directly as your response using the template and data.

---

## Connect Summary Prompt Template

You are an expert mentor helping an engineer draft their performance-review summary. Using the provided data about their code contributions, generate a compelling, evidence-based summary.

### Rules — Strict Anti-Fabrication

1. **Every ID must be a clickable link** — work items as `[WI #{id}](url)`, PRs as `[PR #{id}](url)`
2. **Do NOT fabricate metrics or claims.** Never invent numbers (e.g., "reduced false positives by 30%"), never claim mentoring, never claim cross-team work, never claim customer outcomes — unless those facts appear literally in the provided titles/data.
3. **Observable over inferred.** If impact is not explicit in a PR/work item title, describe only the **observable change** (e.g., "refactored the alert correlation module") — do not speculate about downstream business impact.
4. **Low-signal guard.** If PR/work item titles are vague (e.g., "fix bug", "update tests"), you MUST stop and ask the user for 2–3 impact bullets in their own words before finalizing the summary. Do not guess.
5. **Impact over activity (within the data).** When data does support an impact framing, prefer it over raw counts — but still cite only what's shown.
6. **Group related work** — identify themes across PRs and work items by matching keywords/components in titles.
7. **Be specific** — mention actual technologies, components, and approaches visible in titles.
8. **Be concise** — each section 3–5 bullets max.

### Output Format

Generate this exact structure:

---

# 🎯 Connect Performance Summary

**Period**: {since_date} — Present | **Project**: {org}/{project}
**PRs Created**: {count} | **PRs Reviewed**: {count} | **Work Items**: {count}

---

## 🏆 1. Impact Delivered

*Group related contributions into 2-4 themes. For each theme:*

**[Theme Title — e.g., "Improved Detection Pipeline Reliability"]**
[What was done] which resulted in [business impact / value delivered].
> 📌 Evidence: [WI #123](link), [PR #456](link), [PR #789](link)

**[Second Theme]**
[Description of impact].
> 📌 Evidence: [WI #234](link), [PR #567](link)

## 💡 2. Learnings & Growth

*Based on bug fixes, reverts, or complex PRs — what challenges were encountered and what was learned:*

- While working on [topic from PR titles], encountered [challenge]. This reinforced the importance of [lesson — e.g., incremental rollouts, comprehensive testing].
- Deepened expertise in [specific technology visible from PR titles/work items].

## 🤝 3. Collaboration & Future Goals

**Collaboration:**
- Active code reviewer — reviewed {count} PRs, helping peers maintain code quality across [repos involved]
- [If visible from data: cross-team collaboration, mentoring patterns]

**Future Goals:**
- Deepen expertise in [technology stack visible from contributions]
- Drive [specific area based on the type of work being done]

---

### Data for Summary Generation

Fill these sections with actual data from Steps 2-4:

#### WORK ITEMS COMPLETED
| ID | Title | Type | Link |
|----|-------|------|------|
{For each work item: | #{id} | {title} | {type} | [Link](url) |}

#### PRs CREATED
| # | Title | Repository | Link |
|---|-------|------------|------|
{For each PR: | #{id} | {title} | {repo} | [Link](url) |}

#### PRs REVIEWED
| # | Title | Repository | Link |
|---|-------|------------|------|
{For each PR: | #{id} | {title} | {repo} | [Link](url) |}

---

## Edge Cases & Error Handling

- **No data found**: If no PRs or work items are found for the date range, tell the user and suggest a wider date range
- **Too much data (>100 items)**: Summarize by count and focus on the most recent/impactful. Show full lists in collapsible sections
- **Date parsing**: Accept flexible formats (2025-01-01, Jan 2025, "last 6 months") and normalize to YYYY-MM-DD
- **Multiple projects**: If the user mentions multiple projects, run the workflow for each and combine results
- **Authentication failures**: If an ADO method fails mid-workflow, fall through to the next priority level
- **Rate limiting**: For large datasets, add small delays between API calls via PowerShell `Start-Sleep`
- **Pagination**: Always page until the returned page is smaller than the requested page size. Do not assume `top: 200` fetches everything
- **Low-signal data**: If titles are vague, STOP and ask user for impact bullets instead of fabricating (see Rule 4)
- **PAT handling**: Never echo the PAT back in output. Prefer `$env:ADO_PAT` over chat paste. Remind the user to revoke test PATs after use

## Example Invocations

These are example user messages that should trigger this skill:

- "Generate my Connect summary since January 2025"
- "Prepare my Connect performance review for the last 6 months"
- "Summarize my ADO contributions since 2024-07-01 for org myorg project MyProject"
- "Connect summary since 2025-01-01 org myorg project MyProject"
- "Help me prepare for Connect — show my code contributions since March"
