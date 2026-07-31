## Final Output

---

### Step 1 — Publish RCA to knowledgebase

> **Guardrail — RCA Publish Gate:** Before publishing, apply all checks in `@modules/guardrails/11-rca-publish-gate.md`: completeness check, security ticket restriction, secret/PII scrub, Production Gate G5, and target accessibility verification. Do not publish until all checks pass.

> **Guardrail — Audit Trail:** After publish (success or failure), log `RCA_PUBLISHED` or `RCA_PUBLISH_FAILED` via `@modules/guardrails/12-audit-trail.md`. On workflow completion, log `WORKFLOW_COMPLETED` with the PR URL, RCA URL, and duration.

If `knowledgebase.enabled = true`, publish the RCA document generated in Phase 6 to the configured platform before producing the resolution report. Choose the path based on `knowledgebase.platform`.

---

#### Confluence (`platform = "confluence"`)

Choose the method based on `mcp.atlassian.enabled`:

**If `mcp.atlassian.enabled = true`** — use MCP:

```
<toolPrefix>__createConfluencePage({
  spaceKey: "<confluence.spaceKey>",
  title:    "RCA: <TICKET-ID> — <Ticket Summary>",
  body:     "<full RCA markdown converted to Confluence storage format>",
  parentId: "<id of 'Incidents' or 'RCA' parent page, if it exists>"
})
```

**If `mcp.atlassian.enabled = false`** — use Confluence REST API via WebFetch:

```
POST <confluence.baseUrl>/rest/api/content
Authorization: Basic base64(<confluence.userEmail>:<CONFLUENCE_API_TOKEN>)
Content-Type: application/json

{
  "type": "page",
  "title": "RCA: <TICKET-ID> — <Ticket Summary>",
  "space": { "key": "<confluence.spaceKey>" },
  "ancestors": [{ "id": "<parent page id, if known>" }],
  "body": {
    "storage": {
      "value": "<RCA content in Confluence XHTML storage format>",
      "representation": "storage"
    }
  }
}
```

If a parent page named "Incidents" or "RCAs" exists in the space, nest the page under it. If it does not exist, create the page at the space root.

---

#### SharePoint (`platform = "sharepoint"`)

Always uses WebFetch via Microsoft Graph (no MCP server for SharePoint):

```
# 1. Get auth token
POST https://login.microsoftonline.com/<tenantId>/oauth2/v2.0/token
  client_id=<SHAREPOINT_CLIENT_ID>
  client_secret=<SHAREPOINT_CLIENT_SECRET>
  scope=https://graph.microsoft.com/.default
  grant_type=client_credentials

# 2. Upload RCA as a new file in the configured site's document library
PUT https://graph.microsoft.com/v1.0/sites/<siteId>/drive/root:
    /Incidents/RCA-<TICKET-ID>-<DATE>.md:/content
Authorization: Bearer <access_token>
Content-Type: text/markdown

<full RCA markdown content>
```

If the `Incidents/` folder does not exist, create it first:
```
POST https://graph.microsoft.com/v1.0/sites/<siteId>/drive/root/children
{ "name": "Incidents", "folder": {}, "@microsoft.graph.conflictBehavior": "rename" }
```

---

#### If knowledgebase publish fails

```
⚠️ KNOWLEDGEBASE PUBLISH FAILED

  Platform : <confluence | sharepoint>
  Error    : <HTTP status> — <error message>

  The RCA document was NOT published to the knowledgebase.
  It remains available in the fix branch:
    RCA-<TICKET-ID>-<DATE>.md in fix/<ticket-id>-<slug>

  To publish manually:
    Confluence : paste the markdown into a new page under <confluence.spaceKey>
    SharePoint : upload RCA-<TICKET-ID>-<DATE>.md to <sharepoint.siteUrl>/Incidents/
```

Continue to the resolution report — do not stop for a publish failure.

---

### Step 2 — Agent self-metrics

After the knowledgebase publish attempt (success or failure), collect and persist agent self-metrics for this run:

@modules/09-agent-metrics.md

This step is non-blocking — if metrics collection fails it must not prevent the resolution report from being produced.

---

### Step 3 — Resolution report

After completing all phases and metrics collection, produce the consolidated **Resolution Report**:

```
✅ RESOLUTION COMPLETE — <TICKET-ID>

📋 Ticket:        <summary>
🧠 Root Cause:    <one-liner>
🔧 Fix:           <one-liner>
📄 RCA Doc:       RCA-<ID>-<DATE>.md (attached to PR)
🚀 PR:            <PR URL>
📚 Knowledgebase: <confluence page URL | sharepoint file URL | skipped (disabled) | failed (see above)>
⏱️ Total Time:    <duration>
```

---

### Step 4 — Completion notification

If `notifications.enabled = true`, send notifications after Step 2. Choose the event type:

- **Normal completion** (PR created, RCA published): notify channels whose `events` includes `"completion"`.
- **Workflow failure or hard STOP** (e.g., unresolved service, pre-flight failure, injection abort): notify channels whose `events` includes `"failure"`.

For each matching channel, send a webhook POST to the env var named by `webhookEnvVar`:

#### Slack (`type = "slack"`)

```
POST <SLACK_WEBHOOK_URL>
Content-Type: application/json

{
  "text": "<icon> *Support Ticket Resolver — <TICKET-ID>*",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "<icon> *<TICKET-ID> — <ticket summary>*\n*Status:* <Resolution complete | Workflow failed>\n*Root Cause:* <one-liner or failure reason>\n*PR:* <<PR URL>|View PR>\n*RCA:* <<RCA URL>|View RCA> | _not published_\n*Duration:* <duration>"
      }
    }
  ]
}
```

Replace `<icon>` with ✅ for completion, ❌ for failure. Omit the PR/RCA lines for failure events where those were not produced.

#### Microsoft Teams (`type = "teams"`)

```
POST <TEAMS_WEBHOOK_URL>
Content-Type: application/json

{
  "@type": "MessageCard",
  "@context": "http://schema.org/extensions",
  "themeColor": "<00AA00 for completion | FF0000 for failure>",
  "summary": "Support Ticket Resolver — <TICKET-ID>",
  "sections": [
    {
      "activityTitle": "<icon> <TICKET-ID> — <ticket summary>",
      "activitySubtitle": "Status: <Resolution complete | Workflow failed>",
      "facts": [
        { "name": "Root Cause", "value": "<one-liner or failure reason>" },
        { "name": "PR", "value": "<PR URL or N/A>" },
        { "name": "RCA", "value": "<RCA URL or not published>" },
        { "name": "Duration", "value": "<duration>" }
      ]
    }
  ]
}
```

#### On notification failure

If the webhook returns a non-2xx response or the env var is missing:

```
⚠️ NOTIFICATION FAILED — <channel.name> (<channel.type>)

  Env var : <webhookEnvVar> — <not set | HTTP <status>>
  Event   : <completion | failure>

  The workflow result has already been logged to the audit trail.
  Fix the webhook URL or disable this channel in notifications.channels.
```

**Do not STOP or re-attempt.** Notification failures never block or retry workflow steps.
