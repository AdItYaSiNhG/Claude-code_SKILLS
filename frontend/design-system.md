---
name: frontend-design-system
description: Design token management, component library governance, and Figma-to-code pipelines. Use when setting up or auditing a design system, synchronising tokens between Figma and code, or managing a shared component library across multiple teams.
---

The user is working on design system infrastructure. This skill covers token pipelines, component governance, and multi-team distribution.

## Design Token Architecture

### Token Tier Model
```
Tier 1 — Primitive tokens   (raw values, never used directly in components)
  color.blue.500 = #3B82F6
  size.4 = 16px

Tier 2 — Semantic tokens    (intent-based aliases to primitives)
  color.action.primary = {color.blue.500}
  spacing.component.padding = {size.4}

Tier 3 — Component tokens   (component-specific overrides)
  button.padding.horizontal = {spacing.component.padding}
```

Never allow components to reference Tier 1 tokens directly. Enforcement: lint rule or Style Dictionary transform.

### Style Dictionary Setup

```js
// style-dictionary.config.js
module.exports = {
  source: ['tokens/**/*.json'],
  platforms: {
    css: {
      transformGroup: 'css',
      prefix: 'ds',
      buildPath: 'dist/tokens/',
      files: [{ destination: 'variables.css', format: 'css/variables' }],
    },
    js: {
      transformGroup: 'js',
      buildPath: 'dist/tokens/',
      files: [{ destination: 'tokens.js', format: 'javascript/es6' }],
    },
    ios: {
      transformGroup: 'ios-swift',
      buildPath: 'ios/DesignSystem/',
      files: [{ destination: 'Tokens.swift', format: 'ios-swift/class.swift' }],
    },
    android: {
      transformGroup: 'android',
      buildPath: 'android/res/values/',
      files: [{ destination: 'tokens.xml', format: 'android/resources' }],
    },
  },
}
```

## Figma → Code Pipeline

### Tokens Studio (Figma plugin) + GitHub Actions
```yaml
# .github/workflows/sync-tokens.yml
name: Sync Design Tokens
on:
  repository_dispatch:
    types: [figma-token-push]   # Tokens Studio webhook triggers this
jobs:
  sync:
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx token-transformer tokens/figma-export.json tokens/transformed.json
      - run: npx style-dictionary build
      - uses: peter-evans/create-pull-request@v6
        with:
          title: 'chore(tokens): sync from Figma'
          branch: 'chore/token-sync'
          commit-message: 'chore(tokens): auto-sync from Figma export'
```

### Figma Variables API (2024+)
```ts
// scripts/sync-figma-variables.ts
const res = await fetch(
  `https://api.figma.com/v1/files/${FILE_KEY}/variables/local`,
  { headers: { 'X-Figma-Token': process.env.FIGMA_TOKEN } }
)
const { meta: { variables, variableCollections } } = await res.json()
// Transform to Style Dictionary token format and write to tokens/
```

## Component Library Governance

### Versioning Strategy
- Follow semantic versioning: breaking visual changes = major, new components = minor, bug fixes = patch
- Changelog must reference the Figma component version that triggered the change
- Deprecation window: minimum 2 sprint cycles before removal

### Multi-team Distribution
```
Option A — NPM package (internal registry)
  @company/ui-core          → primitives (Button, Input, Modal)
  @company/ui-patterns      → composed patterns (DataTable, FormWizard)
  @company/ui-charts        → chart components (heavy, separate bundle)

Option B — Git submodule (mono-repo)
  packages/ui/src           → shared across all apps in monorepo

Option C — Micro-frontend component export
  Remote webpack module federation exposing components at runtime
```

### Component Acceptance Criteria (Definition of Done)
Every new component must have:
- [ ] Storybook story with all variants and states (default, hover, focus, disabled, error, loading)
- [ ] Visual snapshot tests in Chromatic
- [ ] a11y audit passing (axe-core, WCAG 2.1 AA)
- [ ] TypeScript props with JSDoc
- [ ] Usage guidelines in Storybook docs tab
- [ ] Token usage documented (which semantic tokens does it consume?)
- [ ] Dark mode support if design system has dark theme

### Token Drift Detection
Run periodically in CI to catch designers modifying tokens without syncing code:
```bash
npx style-dictionary build --output-references
git diff --exit-code dist/tokens/   # fail CI if tokens changed without PR
```

## Output Format

When helping with design system tasks:
1. Show token structure before code — token decisions affect everything downstream
2. Flag any Tier 1 token direct usage in component code as a violation
3. Provide the Style Dictionary transform or Figma plugin config, not just the concept
4. When reviewing a component for DS compliance, output a checklist with pass/fail per criterion
