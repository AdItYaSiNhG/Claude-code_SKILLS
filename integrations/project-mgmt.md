---
name: integrations-project-mgmt
description: Project management tool integrations — Jira, Linear, Notion, Slack, and Figma. Use to automate ticket creation from code comments/PRs, sync development status to project tools, post structured notifications, and link design assets to implementations.
---

The user needs to integrate Claude Code with project management or communication tools. Apply the relevant section.

## Jira Integration

### Create ticket from code (via MCP or script)
```python
# Using Jira REST API v3
import httpx, os

def create_jira_ticket(
    summary: str,
    description: str,
    issue_type: str = "Story",
    priority: str = "Medium",
    labels: list[str] | None = None,
) -> dict:
    response = httpx.post(
        f"{os.environ['JIRA_BASE_URL']}/rest/api/3/issue",
        auth=(os.environ["JIRA_EMAIL"], os.environ["JIRA_API_TOKEN"]),
        json={
            "fields": {
                "project": {"key": os.environ["JIRA_PROJECT_KEY"]},
                "summary": summary,
                "description": {
                    "type": "doc",
                    "version": 1,
                    "content": [
                        {"type": "paragraph", "content": [{"type": "text", "text": description}]}
                    ],
                },
                "issuetype": {"name": issue_type},
                "priority": {"name": priority},
                "labels": labels or [],
            }
        },
    )
    return response.json()

# Usage from Claude Code context:
# "Create a Jira ticket for the auth bug we just fixed"
ticket = create_jira_ticket(
    summary="Fix: token refresh race condition causing intermittent 401 errors",
    description="Multiple concurrent requests could each attempt a token refresh...",
    issue_type="Bug",
    priority="High",
    labels=["auth", "production-bug"],
)
print(f"Created: {os.environ['JIRA_BASE_URL']}/browse/{ticket['key']}")
```

### Link PR to Jira (via commit message)
```
# Smart Commit format — Jira auto-detects these in commit messages:
git commit -m "fix(auth): prevent token refresh race condition

Resolves AUTH-234
Time: 3h
"
# Jira transitions AUTH-234 to "In Review" and logs 3 hours
```

### Jira webhook → Slack notification
```python
# In your webhook handler (FastAPI/Express)
@app.post("/webhooks/jira")
async def jira_webhook(payload: dict):
    if payload["issue_event_type_name"] == "issue_created":
        issue = payload["issue"]
        await slack_notify(
            channel="#engineering-tickets",
            text=f"New {issue['fields']['issuetype']['name']}: "
                 f"<{jira_url}/browse/{issue['key']}|{issue['key']}> — "
                 f"{issue['fields']['summary']}"
        )
```

## Linear Integration

### Create issue via API
```ts
import { LinearClient } from "@linear/sdk"
const linear = new LinearClient({ apiKey: process.env.LINEAR_API_KEY })

// Create issue from code context
const issue = await linear.createIssue({
  title: "Refactor: extract OrderProcessor into separate service",
  description: "The OrderController has grown to 800 lines. Extract business logic into a dedicated OrderProcessor service following the service pattern established in UserService.",
  teamId: process.env.LINEAR_TEAM_ID,
  priority: 2,           // 1=Urgent, 2=High, 3=Medium, 4=Low
  labelIds: ["refactor", "tech-debt"],
  estimate: 3,           // story points
})

console.log(`Created: https://linear.app/issue/${issue.issue?.identifier}`)
```

### GitHub ↔ Linear sync
```yaml
# Linear GitHub integration (configure in Linear settings)
# Automatically:
# - Creates Linear issue when PR is opened with "Fixes LIN-XXX" in body
# - Moves issue to "In Review" when PR is opened
# - Moves to "Done" when PR is merged
# - Closes issue when PR is closed without merge

# Branch naming convention for auto-linking:
git checkout -b lin-234-refactor-order-processor
# Linear detects "lin-234" and links to issue LIN-234
```

## Notion Integration

### Create documentation from code (via Notion API)
```ts
import { Client } from "@notionhq/client"
const notion = new Client({ auth: process.env.NOTION_API_KEY })

// Create a page in the Engineering Wiki
async function createADRPage(adr: {
  title: string
  status: string
  content: string
  databaseId: string
}) {
  await notion.pages.create({
    parent: { database_id: adr.databaseId },
    properties: {
      Name:   { title: [{ text: { content: adr.title } }] },
      Status: { select: { name: adr.status } },
      Date:   { date: { start: new Date().toISOString().split("T")[0] } },
      Tags:   { multi_select: [{ name: "ADR" }, { name: "Architecture" }] },
    },
    children: [
      {
        type: "paragraph",
        paragraph: {
          rich_text: [{ type: "text", text: { content: adr.content } }],
        },
      },
    ],
  })
}
```

### Query Notion for context
```ts
// Find relevant docs before answering a question
const results = await notion.search({
  query: "authentication token refresh",
  filter: { property: "object", value: "page" },
  sort: { direction: "descending", timestamp: "last_edited_time" },
})
```

## Slack Integration

### Structured notifications
```ts
import { WebClient } from "@slack/web-api"
const slack = new WebClient(process.env.SLACK_BOT_TOKEN)

// Rich Block Kit message — never use plain text for alerts
await slack.chat.postMessage({
  channel: "#deployments",
  blocks: [
    {
      type: "header",
      text: { type: "plain_text", text: "🚀 Production Deploy — orders-api v2.1.0" },
    },
    {
      type: "section",
      fields: [
        { type: "mrkdwn", text: "*Status:*\n✅ Success" },
        { type: "mrkdwn", text: "*Duration:*\n3m 42s" },
        { type: "mrkdwn", text: "*Deployed by:*\n@alice" },
        { type: "mrkdwn", text: "*Commit:*\n<https://github.com/org/repo/commit/abc|abc1234>" },
      ],
    },
    {
      type: "section",
      text: {
        type: "mrkdwn",
        text: "*Changes:*\n• fix(auth): prevent token refresh race condition\n• feat(orders): bulk cancellation endpoint",
      },
    },
    {
      type: "actions",
      elements: [
        { type: "button", text: { type: "plain_text", text: "View Release" }, url: "https://github.com/.../releases/tag/v2.1.0" },
        { type: "button", text: { type: "plain_text", text: "Dashboard" }, url: "https://grafana.internal/d/orders" },
      ],
    },
  ],
})
```

### Slash command handler (Claude-powered)
```ts
// Slack slash command: /ask-claude <question>
app.post("/slack/commands/ask-claude", async (req, res) => {
  const { text, user_id, channel_id } = req.body
  res.json({ response_type: "in_channel", text: "⏳ Asking Claude..." })

  const response = await anthropic.messages.create({
    model: "claude-haiku-4-5-20251001",
    max_tokens: 500,
    system: "You are an engineering assistant. Answer concisely in Slack markdown.",
    messages: [{ role: "user", content: text }],
  })

  await slack.chat.postMessage({
    channel: channel_id,
    text: `<@${user_id}> asked: _${text}_\n\n${response.content[0].text}`,
  })
})
```

## Figma Integration

### Fetch design specs for implementation
```ts
// Get component properties from Figma (via MCP or direct API)
const componentData = await fetch(
  `https://api.figma.com/v1/files/${FILE_KEY}/nodes?ids=${NODE_ID}`,
  { headers: { "X-Figma-Token": process.env.FIGMA_TOKEN } }
).then(r => r.json())

// Extract design tokens from Figma Variables API
const variables = await fetch(
  `https://api.figma.com/v1/files/${FILE_KEY}/variables/local`,
  { headers: { "X-Figma-Token": process.env.FIGMA_TOKEN } }
).then(r => r.json())
```

### Link Figma frames to code (Code Connect)
```ts
// figma.config.ts — maps Figma components to React components
import { figma } from "@figma/code-connect"
import { Button } from "./src/components/Button"

figma.connect(Button, "https://www.figma.com/file/...?node-id=...", {
  props: {
    variant: figma.enum("Variant", { primary: "primary", secondary: "secondary" }),
    size:    figma.enum("Size", { sm: "sm", md: "md", lg: "lg" }),
    label:   figma.string("Label"),
    disabled: figma.boolean("Disabled"),
  },
  example: ({ variant, size, label, disabled }) => (
    <Button variant={variant} size={size} disabled={disabled}>{label}</Button>
  ),
})
```

## Cross-tool Workflow: PR → Jira → Slack

```yaml
# .github/workflows/pr-notifications.yml
on:
  pull_request:
    types: [opened, ready_for_review, closed]

jobs:
  notify:
    steps:
      - name: Extract Jira ticket
        id: jira
        run: |
          TICKET=$(echo "${{ github.event.pull_request.body }}" | grep -oP 'Closes [A-Z]+-\d+' | head -1)
          echo "ticket=$TICKET" >> $GITHUB_OUTPUT

      - name: Transition Jira ticket
        if: steps.jira.outputs.ticket != ''
        run: |
          curl -X POST "$JIRA_URL/rest/api/3/issue/${{ steps.jira.outputs.ticket }}/transitions" \
            -u "$JIRA_EMAIL:$JIRA_TOKEN" \
            -H "Content-Type: application/json" \
            -d '{"transition":{"id":"31"}}'   # 31 = In Review

      - name: Notify Slack
        run: |
          curl -X POST $SLACK_WEBHOOK \
            -d '{"text": "PR opened: ${{ github.event.pull_request.title }} — ${{ steps.jira.outputs.ticket }}"}'
```

## Output Format

1. Show the actual API call / SDK code — not pseudocode
2. Include environment variable names the user needs to set
3. For Slack: always use Block Kit (not plain text) for structured messages
4. For cross-tool workflows: show the full GitHub Actions workflow that ties tools together
