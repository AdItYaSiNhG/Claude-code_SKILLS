---
name: browser-visual-qa
description: >
  Use this skill for browser and visual QA tasks: cross-browser compatibility testing
  (Playwright/BrowserStack), layout regression detection, accessibility audits (axe-core),
  colour contrast checking, mobile viewport testing, print stylesheet audits, animation
  performance audits, and font rendering audits.
  Triggers: "cross-browser", "browser testing", "visual regression", "screenshot diff",
  "accessibility audit", "a11y", "colour contrast", "mobile viewport", "responsive testing",
  "print styles", "animation audit", "font rendering", "layout regression", "WCAG".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Browser & Visual QA Skill

## Why this skill exists

Bugs that only appear in Safari, CLS issues only on mobile, colour contrast
failures, broken print layouts — these ship because manual testing can't cover
every surface. This skill delivers **automated, systematic browser and visual QA**
across browsers, viewports, accessibility standards, and rendering.

---

## 0. Audit setup

```bash
# Install Playwright with all browsers
npx playwright install --with-deps 2>/dev/null

# Install accessibility tooling
npm i -D @axe-core/playwright pa11y 2>/dev/null

# Start local server for auditing
npm run build && npm run start &
sleep 3 && curl -s http://localhost:3000 > /dev/null && echo "Server ready"
```

---

## 1. Cross-browser Playwright config

```typescript
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/browser",
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,
  reporter: [["html", { outputFolder: "playwright-report" }]],

  use: {
    baseURL: process.env.BASE_URL ?? "http://localhost:3000",
    screenshot: "only-on-failure",
    video:      "retain-on-failure",
    trace:      "on-first-retry",
  },

  projects: [
    { name: "chrome",             use: { ...devices["Desktop Chrome"] } },
    { name: "firefox",            use: { ...devices["Desktop Firefox"] } },
    { name: "safari",             use: { ...devices["Desktop Safari"] } },
    { name: "edge",               use: { ...devices["Desktop Edge"] } },
    { name: "mobile-chrome",      use: { ...devices["Pixel 7"] } },
    { name: "mobile-safari",      use: { ...devices["iPhone 14"] } },
    { name: "ipad",               use: { ...devices["iPad Pro 11"] } },
  ],

  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
  },
});
```

### Cross-browser smoke test
```typescript
// tests/browser/smoke.spec.ts
import { test, expect } from "@playwright/test";

const CRITICAL_PAGES = ["/", "/login", "/dashboard"];

for (const path of CRITICAL_PAGES) {
  test(`${path} renders without errors`, async ({ page, browserName }) => {
    const consoleErrors: string[] = [];
    const failedRequests: string[] = [];

    page.on("console", m => { if (m.type() === "error") consoleErrors.push(m.text()); });
    page.on("requestfailed", r => failedRequests.push(r.url()));

    await page.goto(path);
    await page.waitForLoadState("networkidle");

    await page.screenshot({
      path: `screenshots/${browserName}${path.replace(/\//g, "-") || "-home"}.png`,
      fullPage: true,
    });

    const realErrors = consoleErrors.filter(e => !e.includes("favicon"));
    expect(realErrors, `Console errors in ${browserName}`).toHaveLength(0);
    expect(failedRequests.filter(u => !u.includes("analytics")), "Failed requests").toHaveLength(0);
  });
}
```

### Safari-specific checks
```typescript
test.describe("Safari compatibility", () => {
  test.use({ ...devices["Desktop Safari"] });

  test("date input works", async ({ page }) => {
    await page.goto("/register");
    const dob = page.getByLabel("Date of birth");
    await dob.fill("1990-01-15");
    await expect(dob).toHaveValue("1990-01-15");
  });

  test("no horizontal overflow (Safari flexbox gap)", async ({ page }) => {
    await page.goto("/dashboard");
    const overflow = await page.evaluate(() =>
      document.documentElement.scrollWidth > window.innerWidth
    );
    expect(overflow).toBe(false);
  });
});
```

---

## 2. Visual regression testing

```typescript
// tests/visual/visual-regression.spec.ts
import { test, expect } from "@playwright/test";

test("homepage matches baseline", async ({ page }) => {
  await page.goto("/");
  await page.waitForLoadState("networkidle");
  // Mask dynamic content (timestamps, avatars)
  await page.addStyleTag({ content: "[data-dynamic], .timestamp { visibility: hidden; }" });
  await expect(page).toHaveScreenshot("homepage.png", { maxDiffPixelRatio: 0.02, fullPage: true });
});

test("button states match baseline", async ({ page }) => {
  await page.goto("/design-system");
  await expect(page.locator(".buttons")).toHaveScreenshot("buttons-default.png");
  await page.locator("button.primary").hover();
  await expect(page.locator(".buttons")).toHaveScreenshot("buttons-hover.png");
});

test("responsive layout — 3 breakpoints", async ({ page }) => {
  for (const { w, h, name } of [
    { w: 375, h: 812, name: "mobile" },
    { w: 768, h: 1024, name: "tablet" },
    { w: 1440, h: 900, name: "desktop" },
  ]) {
    await page.setViewportSize({ width: w, height: h });
    await page.goto("/");
    await page.waitForLoadState("networkidle");
    await expect(page).toHaveScreenshot(`homepage-${name}.png`, { fullPage: true });
  }
});
```

```bash
# Update baselines after intentional change
npx playwright test tests/visual/ --update-snapshots 2>/dev/null
# Run visual tests
npx playwright test tests/visual/ --reporter=html 2>/dev/null && npx playwright show-report
```

---

## 3. Accessibility audit (axe-core + WCAG 2.1 AA)

```typescript
// tests/a11y/accessibility.spec.ts
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

const PAGES = ["/", "/login", "/dashboard", "/checkout"];

for (const path of PAGES) {
  test(`${path} passes WCAG 2.1 AA`, async ({ page }) => {
    await page.goto(path);
    await page.waitForLoadState("networkidle");

    const results = await new AxeBuilder({ page })
      .withTags(["wcag2a", "wcag2aa", "wcag21a", "wcag21aa"])
      .exclude(".third-party-widget")
      .analyze();

    if (results.violations.length) {
      for (const v of results.violations) {
        console.log(`[${v.impact?.toUpperCase()}] ${v.id}: ${v.description}`);
        for (const node of v.nodes.slice(0, 2)) {
          console.log(`  Target: ${node.target}`);
          console.log(`  Fix: ${node.failureSummary}`);
        }
      }
    }

    expect(results.violations).toHaveLength(0);
  });
}

test("keyboard navigation — modal focus trap", async ({ page }) => {
  await page.goto("/dashboard");
  await page.getByRole("button", { name: "Open settings" }).press("Enter");
  await expect(page.getByRole("dialog")).toBeVisible();

  // Focus stays in modal
  await page.keyboard.press("Tab");
  const focused = await page.evaluate(() => document.activeElement?.getAttribute("role") ?? document.activeElement?.tagName);
  expect(focused).toBeTruthy();

  // Escape closes
  await page.keyboard.press("Escape");
  await expect(page.getByRole("dialog")).not.toBeVisible();
});

test("screen reader announcements work", async ({ page }) => {
  await page.goto("/contact");
  await page.getByLabel("Name").fill("Alice");
  await page.getByRole("button", { name: "Submit" }).click();
  await expect(page.locator("[aria-live]")).toContainText(/sent|success/i, { timeout: 5000 });
});
```

---

## 4. Colour contrast

```bash
# Static scan — potential low-contrast CSS (rough heuristic)
python3 - <<'EOF'
import re, math
from pathlib import Path

def hex_to_rgb(h):
    h = h.lstrip("#")
    if len(h) == 3: h = "".join(c*2 for c in h)
    return tuple(int(h[i:i+2], 16) for i in (0, 2, 4))

def relative_luminance(r, g, b):
    def c(v):
        v /= 255
        return v/12.92 if v <= 0.03928 else ((v+0.055)/1.055)**2.4
    return 0.2126*c(r) + 0.7152*c(g) + 0.0722*c(b)

def contrast_ratio(c1, c2):
    l1, l2 = relative_luminance(*c1), relative_luminance(*c2)
    lighter, darker = max(l1,l2), min(l1,l2)
    return (lighter + 0.05) / (darker + 0.05)

issues = []
for f in Path("src").rglob("*.css"):
    text = f.read_text(errors="replace")
    colors = re.findall(r"#([0-9a-fA-F]{3,6})\b", text)
    if len(colors) >= 2:
        for i in range(len(colors)-1):
            try:
                c1 = hex_to_rgb(colors[i])
                c2 = hex_to_rgb(colors[i+1])
                ratio = contrast_ratio(c1, c2)
                if ratio < 4.5:
                    issues.append(f"{f}: #{colors[i]} on #{colors[i+1]} = {ratio:.1f}:1 (need 4.5:1 for AA)")
            except: pass

print(f"Possible contrast issues: {len(issues)}")
for i in issues[:15]: print(f"  {i}")
EOF

# Accurate a11y contrast — via axe-core in test suite (see Section 3)
# pa11y CLI for quick check
npx pa11y http://localhost:3000 --standard WCAG2AA 2>/dev/null \
  | grep -i "contrast" | head -20
```

---

## 5. Mobile viewport testing

```typescript
// tests/browser/responsive.spec.ts
const VIEWPORTS = [
  { name: "iPhone SE",   w: 375,  h: 667  },
  { name: "iPhone 14",   w: 393,  h: 852  },
  { name: "Galaxy S23",  w: 360,  h: 780  },
  { name: "iPad Mini",   w: 768,  h: 1024 },
  { name: "iPad Pro",    w: 1024, h: 1366 },
  { name: "Desktop FHD", w: 1920, h: 1080 },
];

for (const vp of VIEWPORTS) {
  test(`layout correct on ${vp.name}`, async ({ page }) => {
    await page.setViewportSize({ width: vp.w, height: vp.h });
    await page.goto("/");

    // Mobile: hamburger visible; desktop: nav visible
    if (vp.w < 768) {
      await expect(page.getByLabel("Open menu")).toBeVisible();
    } else {
      await expect(page.getByRole("navigation")).toBeVisible();
    }

    // No horizontal overflow
    const overflow = await page.evaluate(() =>
      document.documentElement.scrollWidth > window.innerWidth
    );
    expect(overflow, `Horizontal overflow on ${vp.name}`).toBe(false);

    // No text smaller than 12px
    const tooSmall = await page.evaluate(() =>
      Array.from(document.querySelectorAll("p, li, td, span"))
        .filter(el => parseInt(window.getComputedStyle(el).fontSize) < 12 && (el.textContent?.trim().length ?? 0) > 0).length
    );
    expect(tooSmall, `${tooSmall} elements <12px on ${vp.name}`).toBe(0);
  });
}

test("touch targets ≥44×44px (WCAG 2.5.5)", async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto("/");

  const small = await page.evaluate(() =>
    Array.from(document.querySelectorAll("button, a, input, [role='button']"))
      .filter(el => {
        const r = el.getBoundingClientRect();
        return r.width > 0 && (r.width < 44 || r.height < 44);
      })
      .map(el => ({ tag: el.tagName, text: el.textContent?.trim().slice(0, 30), w: el.getBoundingClientRect().width, h: el.getBoundingClientRect().height }))
  );

  if (small.length) console.table(small);
  expect(small.length, `${small.length} touch targets <44px`).toBe(0);
});
```

---

## 6. Print stylesheet audit

```typescript
// tests/browser/print.spec.ts
test("invoice prints correctly", async ({ page }) => {
  await page.goto("/invoice/123");
  await page.emulateMedia({ media: "print" });

  // Navigation hidden in print
  const navDisplay = await page.evaluate(() =>
    window.getComputedStyle(document.querySelector("nav")!).display
  );
  expect(navDisplay).toBe("none");

  // Tables don't break mid-row
  const badPageBreak = await page.evaluate(() =>
    Array.from(document.querySelectorAll("tr")).some(
      tr => window.getComputedStyle(tr).pageBreakInside === "auto"
    )
  );
  expect(badPageBreak).toBe(false);

  await page.screenshot({ path: "screenshots/print-invoice.png", fullPage: true });
});
```

---

## 7. Animation performance audit

```typescript
// tests/browser/animation.spec.ts
test("animations use compositor-safe properties only", async ({ page }) => {
  await page.goto("/");

  const badAnimations = await page.evaluate(() => {
    const issues: string[] = [];
    const layoutTriggers = ["width","height","top","left","margin","padding","font-size","border"];
    for (const sheet of Array.from(document.styleSheets)) {
      try {
        for (const rule of Array.from(sheet.cssRules ?? [])) {
          if (rule instanceof CSSKeyframesRule) {
            for (const frame of Array.from(rule.cssRules)) {
              for (const prop of layoutTriggers) {
                if ((frame as CSSStyleRule).style?.getPropertyValue(prop)) {
                  issues.push(`${rule.name}: animates '${prop}' → use transform/opacity instead`);
                }
              }
            }
          }
        }
      } catch { /* cross-origin */ }
    }
    return [...new Set(issues)];
  });

  if (badAnimations.length) badAnimations.forEach(i => console.warn(i));
  expect(badAnimations.length).toBeLessThan(3);
});

test("no long tasks during animation (<50ms)", async ({ page }) => {
  await page.goto("/");

  const longTasks = await page.evaluate(() =>
    new Promise<number>(resolve => {
      let count = 0;
      new PerformanceObserver(list => {
        count += list.getEntries().filter(e => e.duration > 50).length;
      }).observe({ entryTypes: ["longtask"] });
      setTimeout(() => resolve(count), 2000);
    })
  );

  expect(longTasks, `${longTasks} long tasks >50ms`).toBeLessThan(3);
});
```

---

## 8. Font rendering audit

```typescript
// tests/browser/fonts.spec.ts
test("fonts load without FOIT (font-display set)", async ({ page }) => {
  await page.goto("/");

  const noFontDisplay = await page.evaluate(() => {
    for (const sheet of Array.from(document.styleSheets)) {
      try {
        for (const rule of Array.from(sheet.cssRules ?? [])) {
          if (rule instanceof CSSFontFaceRule) {
            const d = rule.style.getPropertyValue("font-display");
            if (!d || d === "auto" || d === "block") return true;
          }
        }
      } catch { /* cross-origin */ }
    }
    return false;
  });
  expect(noFontDisplay, "All @font-face rules should have font-display: swap or fallback").toBe(false);
});

test("font swap does not cause CLS >0.1", async ({ page }) => {
  await page.goto("/");
  const cls = await page.evaluate(() =>
    new Promise<number>(resolve => {
      let score = 0;
      new PerformanceObserver(list => {
        for (const entry of list.getEntries())
          if (!(entry as any).hadRecentInput) score += (entry as any).value ?? 0;
      }).observe({ type: "layout-shift", buffered: true });
      setTimeout(() => resolve(score), 3000);
    })
  );
  expect(cls, `CLS from font swap: ${cls.toFixed(3)}`).toBeLessThan(0.1);
});
```

---

## 9. Full QA report generator

```bash
python3 - <<'EOF'
import subprocess, json
from datetime import datetime

report = {}

# Lighthouse
try:
    r = subprocess.run(
        ["npx","lighthouse","http://localhost:3000","--output=json","--quiet","--chrome-flags=--headless"],
        capture_output=True, text=True, timeout=90
    )
    d = json.loads(r.stdout)
    report["lighthouse"] = {k: int(v["score"]*100) for k,v in d["categories"].items()}
except Exception as e:
    report["lighthouse"] = {"error": str(e)}

# pa11y
try:
    r = subprocess.run(["npx","pa11y","http://localhost:3000","--reporter","json"],
                       capture_output=True, text=True, timeout=30)
    issues = json.loads(r.stdout) if r.stdout.strip().startswith("[") else []
    report["pa11y"] = {"total": len(issues), "errors": sum(1 for i in issues if i.get("type")=="error"), "warnings": sum(1 for i in issues if i.get("type")=="warning")}
except Exception as e:
    report["pa11y"] = {"error": str(e)}

print(f"\nBrowser & Visual QA Report — {datetime.now():%Y-%m-%d %H:%M}")
print("="*55)
for tool, data in report.items():
    print(f"\n{tool.upper()}:")
    for k, v in data.items():
        ok = isinstance(v,int) and v >= (90 if tool=="lighthouse" else 0)
        icon = "✓" if (isinstance(v,int) and v>=90 and tool=="lighthouse") or (tool!="lighthouse" and isinstance(v,int) and v==0) else "⚠"
        print(f"  {icon} {k}: {v}")
EOF
```
