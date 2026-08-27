[🏠 Home](../README.md) · [Power Platform](README.md)

# ⚡ Microsoft Power Platform Automation Handbook

> Quick notes + basic/intermediate hands-on scenarios for learning Power Automate (the automation engine of Microsoft Power Platform) from a DevOps perspective.

---

## Table of Contents

1. [What Is Power Platform?](#1-what-is-power-platform)
2. [Power Automate — Core Concepts](#2-power-automate--core-concepts)
3. [Architecture — Mental Model](#3-architecture--mental-model)
4. [Flow Types & Triggers](#4-flow-types--triggers)
5. [Expression Language Cheatsheet](#5-expression-language-cheatsheet)
6. [Basic Scenarios](#6-basic-scenarios)
7. [Intermediate Scenarios](#7-intermediate-scenarios)
8. [DevOps Gotchas — Power Automate](#8-devops-gotchas--power-automate)
9. [Quick Reference Card](#9-quick-reference-card)

---

## 1. What Is Power Platform?

Microsoft Power Platform is a suite of low-code tools:

| Product | Purpose |
|---------|---------|
| **Power Automate** | Workflow automation (this handbook's focus) — "if this happens, do that" |
| **Power Apps** | Build custom business apps (canvas or model-driven) without full code |
| **Power BI** | Business intelligence dashboards and reports |
| **Power Virtual Agents / Copilot Studio** | Build chatbots without code |
| **Dataverse** | The common data platform underneath Power Apps/Automate |

> ⚠️ **Gotcha:** "Low-code" does not mean "no governance needed." Flows created by business users outside IT's visibility ("shadow IT automation") are a real operational risk — a single person's departure can silently break a critical approval flow with no one else aware it exists.

---

## 2. Power Automate — Core Concepts

| Concept | Description |
|---------|-------------|
| **Flow** | An automated workflow — the top-level object you build |
| **Trigger** | The event that starts a flow (schedule, HTTP request, Teams message, SharePoint item created, etc.) |
| **Action** | A single step in the flow (send email, create record, call API, condition, loop) |
| **Connector** | Pre-built integration to an external service (Office 365, SharePoint, Azure, GitHub, Twitter, SQL) |
| **Connection** | An authenticated instance of a connector (your login/token to that service) |
| **Environment** | Isolated workspace — typically Dev / Test / Prod, each with its own Dataverse & flows |
| **Solution** | A packaged, versioned bundle of flows/apps/connections used to promote across environments (like a deployment artifact) |
| **Premium connector** | Requires a paid license (HTTP, SQL Server, custom connectors, most Azure services) |
| **Desktop flow (RPA)** | Automates legacy desktop apps via UI simulation (Power Automate Desktop) |

---

## 3. Architecture — Mental Model

```
        ┌───────────────┐
        │    TRIGGER     │   "When X happens..."
        │ (event source) │   e.g. new SharePoint item, HTTP request, schedule, Teams message
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │    ACTION 1    │   e.g. Get data / Parse JSON
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │   CONDITION     │──── NO ───► Action branch B (e.g. reject, notify)
        │  (if/else)      │
        └───────┬───────┘
              YES
                ▼
        ┌───────────────┐
        │    ACTION 2    │   e.g. Send email / Post to Teams / Call API
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │  ACTION N...   │   Chain continues until flow ends
        └───────────────┘

  Each run is logged independently → visible under "Run history" with
  full input/output of every step (critical for debugging failures).
```

**Connector layer** (how Power Automate reaches other systems):

```
   Power Automate ──► Office 365 connector ──► Outlook / SharePoint / Teams
                 ──► Azure connector      ──► DevOps / Key Vault / Functions / Logic Apps
                 ──► HTTP connector (Premium) ──► Any REST API (GitHub, custom services)
                 ──► On-premises data gateway ──► SQL Server / file shares behind firewall
```

> ⚠️ **Gotcha:** Connectors that reach on-prem resources (SQL Server, file shares) require an **On-premises Data Gateway** installed on a machine inside your network — if that gateway machine reboots or the service account's password expires, every dependent flow fails silently until someone checks.

---

## 4. Flow Types & Triggers

| Flow Type | Use Case |
|-----------|----------|
| **Automated cloud flow** | Triggered by an event (item created, email received, webhook) |
| **Instant cloud flow** | Manually triggered by a button (mobile app, Teams, SharePoint) |
| **Scheduled cloud flow** | Runs on a timer (daily report, nightly cleanup) |
| **Business process flow** | Guides users through a multi-stage process in Dataverse |
| **Desktop flow (RPA)** | Automates a legacy desktop application via simulated clicks/keystrokes |

**Common triggers:**
```
- Recurrence (schedule)
- When an item is created/modified (SharePoint, Dataverse)
- When a new email arrives (Outlook)
- When a HTTP request is received (webhook — Premium)
- When a new response is submitted (Microsoft Forms)
- For a selected item (manual/instant, from SharePoint/Teams UI)
- When a Teams message contains a keyword
```

---

## 5. Expression Language Cheatsheet

Power Automate uses Workflow Definition Language expressions (similar syntax to Azure Logic Apps):

```
utcNow()                                    → current UTC datetime
formatDateTime(utcNow(), 'yyyy-MM-dd')      → formatted date string
addDays(utcNow(), -7)                       → date 7 days in the past
triggerBody()?['fieldName']                 → safely read a field from the trigger payload
body('Step_name')?['fieldName']             → safely read a field from a previous action's output
outputs('Step_name')                        → full output of a named step
if(equals(1,1), 'yes', 'no')                → inline conditional
concat('Hello, ', triggerBody()?['Name'])   → string concatenation
coalesce(triggerBody()?['Email'], 'none')   → fallback value if null
```

> The `?[ ]` (safe navigation) syntax is important — using plain `[ ]` throws a hard error if the field is missing, while `?[ ]` returns `null` instead.

---

## 6. Basic Scenarios

### 6.1 Scenario — Scheduled Email Report

**Goal:** Every Monday at 8 AM, email a status summary to the team.

```
Trigger: Recurrence
  Frequency: Week, Interval: 1, On: Monday, At: 08:00

Action: Office 365 Outlook → Send an email (V2)
  To:      team@yourcompany.com
  Subject: Weekly Report - @{formatDateTime(utcNow(), 'dddd, MMMM d')}
  Body:    <static text or dynamic content pulled from a SharePoint list>
```

**Steps:**
1. [make.powerautomate.com](https://make.powerautomate.com) → **Create** → **Scheduled cloud flow**
2. Set start time + repeat interval as above
3. **+ New step** → search "Send an email (V2)" → fill in fields, using **Dynamic content** for the subject date
4. Save → **Test** → Manually trigger to confirm

**Verify:** Check "Run history" — should show a green ✅ Succeeded run with the email delivered.

---

### 6.2 Scenario — Auto-Respond to a Microsoft Forms Submission

**Goal:** When someone submits a form, log it to SharePoint and send them a confirmation email.

```
Trigger: Microsoft Forms → When a new response is submitted
    │
    ▼
Action 1: Microsoft Forms → Get response details
    (input: Response Id from the trigger)
    │
    ▼
Action 2: SharePoint → Create item
    Site: your SharePoint site
    List: "Form Submissions"
    Map fields: Name → @{body('Get_response_details')?['r123456']}
    │
    ▼
Action 3: Office 365 Outlook → Send an email (V2)
    To: @{body('Get_response_details')?['r_email_field_id']}
    Subject: "Thanks for your submission!"
```

> ⚠️ **Gotcha:** Forms field references look like `r123456` (an internal ID), not the human-readable question text. If you rename or delete a question in the form later, the flow's field mapping silently breaks — always re-test the flow after editing the form.

---

## 7. Intermediate Scenarios

### 7.1 Approval Workflow (SharePoint + Teams)

**Goal:** Employee submits a leave request via a SharePoint list → manager approves/rejects via Teams → status updates automatically.

```
┌─────────────────────────┐
│ SharePoint: item created │  (Leave Requests list)
└────────────┬─────────────┘
             ▼
┌─────────────────────────────────────┐
│ Start and wait for an approval        │
│  Type: Approve/Reject – First to      │
│         respond                       │
│  Assigned to: Manager's email          │
│  Title: "Leave Request from @{title}" │
└────────────┬─────────────────────────┘
             ▼
        ┌─────────┐
        │Condition │  outcome == "Approve"?
        └────┬────┘
     YES │         │ NO
         ▼         ▼
 ┌───────────┐ ┌───────────┐
 │ Update item │ │ Update item │
 │ Status=     │ │ Status=     │
 │ "Approved"  │ │ "Rejected"  │
 └─────┬──────┘ └─────┬──────┘
       ▼               ▼
 Email employee   Email employee
 (approved)       (rejected + comments)
```

**Key points:**
- The approval action **pauses** the flow until the manager responds — this can take hours or days, and that's expected/fine.
- Use **Dynamic content** to pull the manager's email and request details from the triggering SharePoint item.

---

### 7.2 HTTP Webhook → Post to Teams (Premium Connector)

**Goal:** Receive a GitHub webhook (PR opened) and post a formatted message into a Teams channel.

```
Trigger: When a HTTP request is received
  → Power Automate generates a unique URL
  → Paste that URL into GitHub: Settings → Webhooks → Add webhook
  → Paste a sample GitHub PR payload into "Use sample payload to generate schema"

Action 1: Parse JSON
  Content: triggerBody()
  Schema: (auto-generated from the sample payload)

Action 2: Microsoft Teams → Post message in a chat or channel
  Message: "PR opened: @{body('Parse_JSON')?['pull_request']?['title']}
            by @{body('Parse_JSON')?['pull_request']?['user']?['login']}"
  Link:    @{body('Parse_JSON')?['pull_request']?['html_url']}
```

> ⚠️ **Gotcha:** The "When a HTTP request is received" trigger is a **Premium** connector. Using it in a flow will block the flow from running (or require an upgrade) for users/environments without a Premium license — always confirm licensing before designing around it.

---

### 7.3 Azure DevOps Pipeline → Teams Notification

**Goal:** When a deployment pipeline completes, notify a Teams channel with the result — a common ChatOps pattern.

**Option A — from the pipeline directly (no Power Automate needed):**
```yaml
- task: PowerShell@2
  condition: always()
  inputs:
    targetType: inline
    script: |
      $status = "$(Agent.JobStatus)"
      $body = @{
        text = "Pipeline: $(Build.DefinitionName) — $status"
      } | ConvertTo-Json
      Invoke-RestMethod -Uri "$(TEAMS_WEBHOOK_URL)" -Method Post -Body $body -ContentType "application/json"
```

**Option B — via Power Automate (if you need approvals/branching logic beyond a simple webhook):**
```
Trigger: Azure DevOps → When a build/release completes (requires the Azure DevOps connector)
    ▼
Condition: status == "succeeded"?
    ▼
Action: Microsoft Teams → Post message with pipeline name, status, and a link back to the run
```

> Prefer **Option A** for simple one-way notifications — it avoids a dependency on Power Automate entirely. Use Power Automate when you need conditional logic, approvals, or fan-out to multiple systems (e.g., Teams **and** SharePoint **and** an email digest) from one trigger.

---

## 8. DevOps Gotchas — Power Automate

> ⚠️ **Gotcha: Flows are owned by a person, not a team, by default.** If the flow creator leaves the company and their account is disabled, all their flows stop running — always co-own critical flows or use a **service/application account** as the connection owner for production automation.

> ⚠️ **Gotcha: Connections use delegated (user) auth by default.** A flow calling SharePoint/Outlook "as" a specific user breaks the moment that user's password changes, MFA is reconfigured, or the account is offboarded. Prefer **service principals / application permissions** for production connectors where supported.

> ⚠️ **Gotcha: No built-in environment promotion without Solutions.** Building flows directly in an environment (rather than inside a **Solution**) means there's no clean way to promote Dev → Test → Prod — you end up manually recreating flows, which drifts over time. Always build inside a Solution from day one for anything beyond a personal quick-fix flow.

> ⚠️ **Gotcha: Throttling limits are per-connector, not obvious until you hit them.** Office 365 connectors have request-per-minute limits; a flow processing a large batch (e.g., looping over 5,000 SharePoint items) can silently start failing mid-run with `429 Too Many Requests`. Add pagination and `Delay` actions for bulk operations.

> ⚠️ **Gotcha: "For a selected item" (instant flows) only shows up if permissions align.** If a user doesn't have at least Edit permission on the underlying list/library, the flow won't appear in the context menu — a common "why isn't my flow showing up" support ticket.

> ⚠️ **Gotcha: Infinite loop risk with "when item is modified" triggers.** If a flow triggers on item modification and then itself updates that same item, it can re-trigger itself — always add a condition to check whether the relevant field actually changed, or use "Trigger Conditions" to filter.

> ⚠️ **Gotcha: Premium connectors silently gate features.** A flow built by someone with a premium license may fail for everyone else in the environment once that license lapses — Power Automate doesn't loudly announce "this flow needs Premium," it just starts erroring at runtime.

> ⚠️ **Gotcha: No native source control.** Flows aren't stored as diffable text by default like a Terraform file or GitHub Action YAML. Use **Solutions + Power Platform CLI (`pac`)** to export/version flows as part of a real CI/CD pipeline if you want auditable change history.

---

## 9. Quick Reference Card

| Item | Where to find it |
|------|-------------------|
| Flow designer | [make.powerautomate.com](https://make.powerautomate.com) |
| Run history / debugging | My flows → (flow name) → 28-day run history |
| Connections | Data → Connections |
| Solutions (for CI/CD) | Solutions → New solution → add flows to it |
| Admin center (governance) | [admin.powerplatform.microsoft.com](https://admin.powerplatform.microsoft.com) |
| CLI for automation/export | `pac` (Power Platform CLI) |
| On-prem gateway | Data → Gateways |
| Licensing check | Power Platform Admin Center → Billing → Licenses |

**Typical automation building blocks:**
```
Trigger      → Recurrence / Item created / HTTP request / Forms submitted / Teams message
Get data     → Get item, Get response details, HTTP GET, SQL query
Transform    → Parse JSON, Compose, Select, Filter array
Decide       → Condition, Switch, Apply to each (loop)
Act          → Send email, Post to Teams, Create/Update item, Call HTTP API
Wait for human → Start and wait for an approval
```

---

[🏠 Back to Home](../README.md) · [Power Platform Index](README.md)
