---
name: integrations
description: Integration skills covering MCP server configuration, GitHub automation (Conventional Commits, release pipelines, CHANGELOG), and project management toolchain (Jira, Linear, Notion, Slack). Use for connecting Claude Code to external tools and automating cross-tool workflows.
license: Internal enterprise use
---

Three sub-skills cover the integrations surface area:

| Sub-skill file | Invoke when… |
|---|---|
| `mcp.md` | configuring or building MCP servers to connect Claude Code to external tools |
| `github.md` | automating commits, releases, CHANGELOGs, or PR workflows |
| `project-mgmt.md` | integrating with Jira, Linear, Notion, Slack, or Figma |

## Routing

- "Connect Claude to {tool}" → `mcp.md`
- "Conventional commits / CHANGELOG / release notes" → `github.md`
- "Create a Jira ticket / sync to Notion / post to Slack" → `project-mgmt.md`
- Building a full release workflow → `github.md` + `project-mgmt.md`

## Enterprise Context

Integrations are the connective tissue of a Claude Code enterprise deployment. Without them, engineers context-switch constantly between tools. With them, Claude Code becomes the single interface for code, tasks, communication, and documentation.
