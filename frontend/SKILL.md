---
name: frontend
description: Enterprise frontend engineering skills covering UI frameworks, browser/visual QA, and design system management. Use when building, auditing, or scaling frontend applications. Sub-skills handle React/Vue/Angular patterns, visual regression, a11y audits, and design token pipelines.
license: Internal enterprise use
---

This category covers all frontend engineering concerns at enterprise scale. Three sub-skills are available — invoke the right one based on context.

| Sub-skill file | Invoke when… |
|---|---|
| `ui.md` | building or reviewing UI components, pages, or framework-specific patterns |
| `browser.md` | running visual regression, cross-browser QA, or accessibility audits |
| `design-system.md` | managing design tokens, Figma-to-code pipelines, or component library governance |

## When multiple sub-skills apply
Start with `ui.md` for implementation, then layer in `design-system.md` if token compliance is required, and `browser.md` for pre-release QA gates.

## Enterprise context
Large frontend codebases share these failure modes: token drift between Figma and code, untested visual regressions across browser versions, and a11y gaps that create legal exposure. These skills address all three systematically.
