---
name: frontend-browser
description: Browser QA, visual regression testing, cross-browser compatibility, and accessibility auditing. Use before releases, after major UI refactors, or when a11y compliance is required. Covers Playwright, Percy, Chromatic, Axe, and Lighthouse CI.
---

The user needs to run, write, or interpret browser-level QA. This covers three concerns: visual regression, cross-browser compatibility, and accessibility.

## Visual Regression Testing

### Playwright + Chromatic (Storybook)
```ts
// playwright/visual.spec.ts
import { test, expect } from '@playwright/test'

test('Dashboard snapshot', async ({ page }) => {
  await page.goto('/dashboard')
  await page.waitForLoadState('networkidle')
  // Mask dynamic content (timestamps, user-specific data)
  await expect(page).toHaveScreenshot('dashboard.png', {
    mask: [page.locator('[data-testid="timestamp"]')],
    threshold: 0.02,   // 2% pixel diff tolerance
    animations: 'disabled',
  })
})
```

**Chromatic integration** (CI):
```yaml
# .github/workflows/chromatic.yml
- name: Run Chromatic
  uses: chromaui/action@v1
  with:
    projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
    exitZeroOnChanges: true   # Don't fail build — let reviewers approve
```

### Percy (BrowserStack alternative)
```ts
import percySnapshot from '@percy/playwright'
await percySnapshot(page, 'Dashboard loaded')
```

### When to use which
| Tool | Best for |
|---|---|
| Playwright screenshots | Full-page, pixel-level, CI-gated |
| Chromatic | Storybook component-level, design review workflow |
| Percy | Multi-browser, BrowserStack integration |

## Cross-Browser Test Matrix

Minimum enterprise matrix (adjust for analytics data):

```yaml
# playwright.config.ts
projects:
  - name: chromium
  - name: firefox
  - name: webkit           # Safari proxy
  - name: mobile-chrome    # Pixel 5
  - name: mobile-safari    # iPhone 12
```

### Common cross-browser issues to catch
- CSS `gap` in flex (Safari <14.1 needs fallback)
- `aspect-ratio` (IE11 requires JS polyfill if still supported)
- `subgrid` (Firefox 71+, Chrome 117+)
- CSS `:has()` selector (Safari 15.4+, Chrome 105+)
- `scrollbar-gutter` not in Safari
- Custom form controls behave differently in Firefox vs WebKit

## Accessibility Audit Workflow

### Automated (Axe + Playwright)
```ts
import AxeBuilder from '@axe-core/playwright'

test('Dashboard a11y', async ({ page }) => {
  await page.goto('/dashboard')
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
    .analyze()
  expect(results.violations).toEqual([])
})
```

### Lighthouse CI (Performance + a11y score gate)
```yaml
# lighthouserc.yml
ci:
  collect:
    url: ['http://localhost:3000/dashboard']
  assert:
    assertions:
      accessibility: ['error', { minScore: 0.9 }]
      performance:   ['warn',  { minScore: 0.8 }]
      best-practices: ['error', { minScore: 0.9 }]
```

### Manual a11y checklist (run quarterly or post-major-release)
- [ ] Screen reader walkthrough (NVDA + Chrome, VoiceOver + Safari)
- [ ] Keyboard-only navigation — can complete every user flow?
- [ ] Zoom to 200% — no content overflow or overlap
- [ ] Color blind simulation (Chromium DevTools → Rendering → Emulate vision)
- [ ] Focus order matches reading order
- [ ] Error messages associated with inputs via `aria-describedby`
- [ ] Modal/dialog traps focus, restores on close
- [ ] No positive `tabindex` (all `tabindex="0"` or `-1` only)

## CI Integration Pattern

```yaml
# Recommended stage order
jobs:
  unit-tests: ...
  visual-regression:
    needs: unit-tests
    runs-on: ubuntu-latest
    steps:
      - run: npx playwright install --with-deps
      - run: npx playwright test --project=chromium
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
  a11y-audit:
    needs: visual-regression
    steps:
      - run: npx lhci autorun
```

## Interpreting Results

When presenting audit output to the user:
1. Group violations by **severity** (critical → serious → moderate → minor)
2. For each critical/serious: show the failing element, WCAG rule, and a concrete fix
3. Prioritise fixes that affect the most users or highest-traffic pages
4. Provide a re-test command so the user can verify the fix immediately
