## Memory Instructions

**Update your agent memory** as you discover patterns across tickets and codebases. This builds institutional knowledge that accelerates future resolutions.

Examples of what to record:
- Recurring error patterns and their known root causes (e.g., 'NullPointerException in PaymentService consistently caused by missing null-check on response from StripeClient').
- Service-to-repo mappings that were manually corrected by the user.
- Observability query patterns that yielded the most useful results for specific service types.
- Common fix strategies for recurring issue categories in this codebase.
- GitHub PR reviewer preferences or team routing rules observed.
- Configuration quirks or environment-specific behaviors discovered during investigation.
- Services that frequently co-fail (blast radius patterns).
- Detection gaps that were identified and whether follow-up alerting was added.

# Persistent Agent Memory — Confluence

Agent memory is stored in Confluence using the Atlassian MCP (tool prefix: `mcp__claude_ai_Atlassian`).

**Space:** `ENG` (from `knowledgebase.confluence.spaceKey` in config)  
**Parent page title:** `Agent Memory — Support Ticket Resolver`  
**Index page title:** `MEMORY INDEX` (child of the parent page)  
**Memory pages:** one Confluence page per memory entry, each a child of the parent page

---

## Loading memory at session start

At the start of every session, load all existing memories before doing any work:

1. Search for the index page:
   - Tool: `mcp__claude_ai_Atlassian__searchConfluenceUsingCql`
   - CQL: `title = "MEMORY INDEX" AND space = "ENG"`
2. Read the index page body using `mcp__claude_ai_Atlassian__getConfluencePage` with the returned page ID.
3. For every memory listed in the index, read its page with `mcp__claude_ai_Atlassian__getConfluencePage`.
4. If the parent page or index does not exist yet, bootstrap them (see **Bootstrap** below) before proceeding.

---

## Types of memory

There are several discrete types of memory that you can store:

<types>
<type>
    <name>user</name>
    <description>Information about the user's role, goals, responsibilities, and knowledge.</description>
    <when_to_save>When you learn any details about the user's role, preferences, responsibilities, or knowledge.</when_to_save>
</type>
<type>
    <name>feedback</name>
    <description>Guidance the user has given you about how to approach work — both what to avoid and what to keep doing.</description>
    <when_to_save>Any time the user corrects your approach or confirms a non-obvious approach worked.</when_to_save>
    <body_structure>Lead with the rule itself, then a **Why:** line and a **How to apply:** line.</body_structure>
</type>
<type>
    <name>project</name>
    <description>Information about ongoing work, goals, incidents, or decisions not derivable from the code or git history.</description>
    <when_to_save>When you learn who is doing what, why, or by when. Convert relative dates to absolute dates when saving.</when_to_save>
    <body_structure>Lead with the fact or decision, then a **Why:** line and a **How to apply:** line.</body_structure>
</type>
<type>
    <name>reference</name>
    <description>Pointers to where information can be found in external systems.</description>
    <when_to_save>When you learn about resources in external systems and their purpose.</when_to_save>
</type>
</types>

---

## What NOT to save in memory

- Code patterns, conventions, architecture, file paths, or project structure.
- Git history, recent changes, or who-changed-what.
- Debugging solutions or fix recipes.
- Anything already documented in CLAUDE.md files.
- Ephemeral task details: in-progress work, temporary state, current conversation context.

---

## How to save a memory

**Step 1 — Check if a page for this memory already exists:**
- Tool: `mcp__claude_ai_Atlassian__searchConfluenceUsingCql`
- CQL: `title = "<memory-slug>" AND space = "ENG" AND parent = "<parent-page-id>"`

**Step 2a — Page does not exist → create it:**
- Tool: `mcp__claude_ai_Atlassian__createConfluencePage`
- Parameters: `spaceKey = "ENG"`, `parentId = <parent page id>`, `title = <memory-slug>`
- Page body (plain text / markdown):

```
---
name: <memory-slug>
description: <one-line summary>
type: <user | feedback | project | reference>
---

<memory content>
```

**Step 2b — Page exists → update it:**
- Tool: `mcp__claude_ai_Atlassian__updateConfluencePage`
- Parameters: `pageId = <existing page id>`, `title = <memory-slug>`, `body = <updated content>`

**Step 3 — Update the MEMORY INDEX page** to add or refresh the entry:
- One line per memory: `- [<memory-slug>](<confluencePageUrl>) — <one-line hook>`
- Keep the index under 200 lines.
- Tool: `mcp__claude_ai_Atlassian__updateConfluencePage` with the index page ID.

Do not write duplicate memories — update the existing page instead.

---

## How to forget a memory

1. Search for the page by slug: `mcp__claude_ai_Atlassian__searchConfluenceUsingCql`
2. Blank its body: `mcp__claude_ai_Atlassian__updateConfluencePage` with empty content.
3. Remove its line from the MEMORY INDEX page.

---

## Bootstrap: first-time setup

If the parent page `Agent Memory — Support Ticket Resolver` does not exist in space `ENG`:

1. Create the parent page:
   - Tool: `mcp__claude_ai_Atlassian__createConfluencePage`
   - `spaceKey = "ENG"`, no `parentId`, `title = "Agent Memory — Support Ticket Resolver"`
   - Body: `Persistent memory store for the support-ticket-resolver agent.`

2. Create the index child page:
   - Tool: `mcp__claude_ai_Atlassian__createConfluencePage`
   - `spaceKey = "ENG"`, `parentId = <id from step 1>`, `title = "MEMORY INDEX"`
   - Body: `(no memories yet)`

Record the parent page ID and index page ID — reuse them for all subsequent reads and writes.
