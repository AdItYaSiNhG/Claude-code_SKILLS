---
name: design-system
description: >
  Use this skill for design system and component library tasks: Tailwind CSS design
  systems, shadcn/ui component integration, Storybook setup, design token management,
  dark mode and theming, and responsive layout audits. Covers token architecture,
  component variant systems, documentation, and consistency enforcement.
  Triggers: "design system", "design tokens", "Tailwind", "shadcn", "component library",
  "Storybook", "dark mode", "theming", "token", "typography scale", "colour palette",
  "spacing system", "component variants".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Design System Skill

## Why this skill exists

Design systems without rigorous token and variant architecture fracture into
inconsistency at scale — every dev ships their own shade of grey and margin.
This skill provides **token-first architecture, component variant patterns,
Storybook documentation, and theming infrastructure** that keeps UI consistent
as teams grow.

---

## 0. Audit existing design system

```bash
# Detect design system tooling
cat package.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
all_deps = {**d.get('dependencies',{}), **d.get('devDependencies',{})}
tools = ['tailwindcss','@shadcn/ui','@radix-ui/react-dialog','storybook',
         '@storybook/react','styled-components','@emotion/react','stitches',
         'vanilla-extract','panda-css']
for t in tools:
    if t in all_deps: print(f'{t}: {all_deps[t]}')
"

# Design token files
find . -name "tokens*.json" -o -name "design-tokens*" \
  -o -name "theme*.ts" -o -name "colors*.ts" \
  | grep -v node_modules | head -10

# Storybook stories
find . -name "*.stories.tsx" -o -name "*.stories.ts" \
  | grep -v node_modules | wc -l
```

---

## 1. Design token architecture

### Token layers (three-layer system)
```typescript
// tokens/primitive.ts — raw values, no semantic meaning
export const primitive = {
  color: {
    // Complete scale for each palette colour
    gray:   { 50:"#f9fafb", 100:"#f3f4f6", 200:"#e5e7eb", 300:"#d1d5db", 400:"#9ca3af", 500:"#6b7280", 600:"#4b5563", 700:"#374151", 800:"#1f2937", 900:"#111827", 950:"#030712" },
    blue:   { 50:"#eff6ff", 100:"#dbeafe", 200:"#bfdbfe", 300:"#93c5fd", 400:"#60a5fa", 500:"#3b82f6", 600:"#2563eb", 700:"#1d4ed8", 800:"#1e40af", 900:"#1e3a8a", 950:"#172554" },
    red:    { 50:"#fef2f2", 500:"#ef4444", 600:"#dc2626", 700:"#b91c1c" },
    green:  { 50:"#f0fdf4", 500:"#22c55e", 600:"#16a34a", 700:"#15803d" },
    amber:  { 50:"#fffbeb", 500:"#f59e0b", 600:"#d97706", 700:"#b45309" },
  },
  spacing: { 0:"0", 1:"4px", 2:"8px", 3:"12px", 4:"16px", 5:"20px", 6:"24px", 8:"32px", 10:"40px", 12:"48px", 16:"64px", 20:"80px", 24:"96px" },
  fontSize: { xs:"12px", sm:"14px", base:"16px", lg:"18px", xl:"20px", "2xl":"24px", "3xl":"30px", "4xl":"36px", "5xl":"48px" },
  fontWeight: { normal:400, medium:500, semibold:600, bold:700 },
  lineHeight: { tight:1.25, snug:1.375, normal:1.5, relaxed:1.625, loose:2 },
  borderRadius: { none:"0", sm:"2px", DEFAULT:"4px", md:"6px", lg:"8px", xl:"12px", "2xl":"16px", full:"9999px" },
  shadow: {
    sm:  "0 1px 2px 0 rgb(0 0 0 / 0.05)",
    DEFAULT: "0 1px 3px 0 rgb(0 0 0 / 0.1), 0 1px 2px -1px rgb(0 0 0 / 0.1)",
    md:  "0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1)",
    lg:  "0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1)",
  },
} as const;

// tokens/semantic.ts — meaning-bearing aliases (reference primitives only)
export const semantic = {
  color: {
    // Backgrounds
    bg: {
      default:   primitive.color.gray[50],
      subtle:    primitive.color.gray[100],
      muted:     primitive.color.gray[200],
      inverse:   primitive.color.gray[900],
    },
    // Text
    text: {
      primary:   primitive.color.gray[900],
      secondary: primitive.color.gray[600],
      disabled:  primitive.color.gray[400],
      inverse:   primitive.color.gray[50],
      link:      primitive.color.blue[600],
    },
    // Border
    border: {
      default:   primitive.color.gray[200],
      strong:    primitive.color.gray[400],
      focus:     primitive.color.blue[500],
    },
    // Feedback
    feedback: {
      error:     { bg: primitive.color.red[50],   text: primitive.color.red[700],   border: primitive.color.red[200] },
      success:   { bg: primitive.color.green[50],  text: primitive.color.green[700], border: primitive.color.green[200] },
      warning:   { bg: primitive.color.amber[50],  text: primitive.color.amber[700], border: primitive.color.amber[200] },
      info:      { bg: primitive.color.blue[50],   text: primitive.color.blue[700],  border: primitive.color.blue[200] },
    },
  },
} as const;

// tokens/component.ts — component-specific tokens (reference semantic only)
export const component = {
  button: {
    primary:   { bg: primitive.color.blue[600],  text: "#fff",                    hover: primitive.color.blue[700] },
    secondary: { bg: "transparent",               text: primitive.color.blue[600], border: primitive.color.blue[600], hover: primitive.color.blue[50] },
    danger:    { bg: primitive.color.red[600],    text: "#fff",                    hover: primitive.color.red[700] },
    ghost:     { bg: "transparent",               text: primitive.color.gray[700], hover: primitive.color.gray[100] },
  },
} as const;
```

### CSS custom properties output
```typescript
// tokens/css.ts — generates CSS variables from token layers
export function generateCSSVariables(theme: "light" | "dark"): string {
  const t = theme === "dark" ? darkTokens : lightTokens;
  return Object.entries(t)
    .map(([key, value]) => `  --${key}: ${value};`)
    .join("\n");
}

// In global CSS:
// :root { ...lightTokens }
// [data-theme="dark"] { ...darkTokens }
```

---

## 2. Tailwind design system configuration

```typescript
// tailwind.config.ts
import type { Config } from "tailwindcss";
import { primitive, semantic } from "./tokens";

export default {
  content: ["./src/**/*.{ts,tsx}"],
  darkMode: "class",    // or "media"

  theme: {
    // OVERRIDE (not extend) — prevents usage of Tailwind defaults
    colors: {
      transparent: "transparent",
      current:     "currentColor",
      white:       "#ffffff",
      black:       "#000000",
      gray:  primitive.color.gray,
      blue:  primitive.color.blue,
      red:   primitive.color.red,
      green: primitive.color.green,
      amber: primitive.color.amber,
    },
    spacing:      primitive.spacing,
    fontSize:     Object.fromEntries(Object.entries(primitive.fontSize).map(([k,v]) => [k,[v,{lineHeight:"1.5"}]])),
    fontWeight:   primitive.fontWeight,
    borderRadius: primitive.borderRadius,
    boxShadow:    primitive.shadow,

    extend: {
      // Semantic colour aliases as CSS var references (dark mode aware)
      backgroundColor: {
        DEFAULT:       "rgb(var(--bg-default) / <alpha-value>)",
        subtle:        "rgb(var(--bg-subtle) / <alpha-value>)",
      },
      textColor: {
        primary:       "rgb(var(--text-primary) / <alpha-value>)",
        secondary:     "rgb(var(--text-secondary) / <alpha-value>)",
      },
    },
  },

  plugins: [
    require("@tailwindcss/typography"),
    require("@tailwindcss/forms"),
    require("@tailwindcss/container-queries"),
  ],
} satisfies Config;
```

---

## 3. Component variant system (cva)

```typescript
// components/ui/Button/Button.tsx
import { cva, type VariantProps } from "class-variance-authority";
import { forwardRef } from "react";
import { cn } from "@/lib/utils";

const buttonVariants = cva(
  // Base — always applied
  "inline-flex items-center justify-center gap-2 rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        primary:   "bg-blue-600 text-white hover:bg-blue-700 focus-visible:ring-blue-500",
        secondary: "border border-blue-600 text-blue-600 hover:bg-blue-50 focus-visible:ring-blue-500",
        danger:    "bg-red-600 text-white hover:bg-red-700 focus-visible:ring-red-500",
        ghost:     "text-gray-700 hover:bg-gray-100 focus-visible:ring-gray-400",
        link:      "text-blue-600 underline-offset-4 hover:underline",
      },
      size: {
        sm:  "h-8  px-3 text-sm",
        md:  "h-10 px-4 text-sm",
        lg:  "h-12 px-6 text-base",
        icon:"h-10 w-10",
      },
    },
    defaultVariants: { variant: "primary", size: "md" },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  loading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, loading, leftIcon, rightIcon, children, disabled, ...props }, ref) => (
    <button
      ref={ref}
      className={cn(buttonVariants({ variant, size }), className)}
      disabled={disabled || loading}
      aria-busy={loading}
      {...props}
    >
      {loading && <span className="h-4 w-4 animate-spin rounded-full border-2 border-current border-t-transparent" aria-hidden />}
      {!loading && leftIcon}
      {children}
      {rightIcon}
    </button>
  )
);
Button.displayName = "Button";
```

---

## 4. shadcn/ui integration

```bash
# Initialise shadcn/ui in a Next.js project
npx shadcn-ui@latest init 2>/dev/null
# Options: New York style, CSS variables, Tailwind config

# Add components selectively
npx shadcn-ui@latest add button 2>/dev/null
npx shadcn-ui@latest add dialog 2>/dev/null
npx shadcn-ui@latest add form 2>/dev/null
npx shadcn-ui@latest add table 2>/dev/null

# List available components
npx shadcn-ui@latest add --help 2>/dev/null
```

```typescript
// components.json — shadcn config
{
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "src/app/globals.css",
    "baseColor": "slate",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils"
  }
}
```

### Extend shadcn component
```typescript
// Extend, don't fork — wrap the shadcn primitive
import { Button as ShadcnButton } from "@/components/ui/button";
import { Loader2 } from "lucide-react";

export function Button({ loading, children, ...props }: ButtonProps) {
  return (
    <ShadcnButton disabled={props.disabled || loading} {...props}>
      {loading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
      {children}
    </ShadcnButton>
  );
}
```

---

## 5. Storybook setup

```bash
# Initialise Storybook
npx storybook@latest init 2>/dev/null

# Run Storybook
npx storybook dev -p 6006 2>/dev/null
```

```typescript
// .storybook/main.ts
import type { StorybookConfig } from "@storybook/nextjs";

export default {
  stories: ["../src/**/*.stories.@(ts|tsx)"],
  addons: [
    "@storybook/addon-essentials",
    "@storybook/addon-a11y",         // accessibility panel
    "@storybook/addon-interactions", // play functions
    "@chromatic-com/storybook",      // visual regression
  ],
  framework: { name: "@storybook/nextjs", options: {} },
} satisfies StorybookConfig;
```

```typescript
// components/ui/Button/Button.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { userEvent, within, expect } from "@storybook/test";
import { Button } from "./Button";

const meta: Meta<typeof Button> = {
  title: "UI/Button",
  component: Button,
  tags: ["autodocs"],
  argTypes: {
    variant: { control: "select", options: ["primary","secondary","danger","ghost","link"] },
    size:    { control: "select", options: ["sm","md","lg","icon"] },
    loading: { control: "boolean" },
    disabled:{ control: "boolean" },
  },
};
export default meta;
type Story = StoryObj<typeof Button>;

export const Default: Story = { args: { children: "Click me" } };

export const AllVariants: Story = {
  render: () => (
    <div className="flex flex-wrap gap-3">
      {(["primary","secondary","danger","ghost"] as const).map(v => (
        <Button key={v} variant={v}>{v}</Button>
      ))}
    </div>
  ),
};

export const LoadingState: Story = {
  args: { children: "Saving…", loading: true },
};

// Interaction test
export const ClickFeedback: Story = {
  args: { children: "Submit", onClick: () => {} },
  play: async ({ canvasElement, args }) => {
    const canvas = within(canvasElement);
    await userEvent.click(canvas.getByRole("button"));
    expect(args.onClick).toHaveBeenCalled();
  },
};
```

---

## 6. Dark mode / theming

```typescript
// lib/theme/ThemeProvider.tsx
"use client";
import { createContext, useContext, useEffect, useState } from "react";

type Theme = "light" | "dark" | "system";
const ThemeContext = createContext<{ theme: Theme; setTheme: (t: Theme) => void } | null>(null);

export function ThemeProvider({ children, defaultTheme = "system" }: ThemeProviderProps) {
  const [theme, setThemeState] = useState<Theme>(() =>
    (typeof localStorage !== "undefined" && (localStorage.getItem("theme") as Theme)) || defaultTheme
  );

  useEffect(() => {
    const root = document.documentElement;
    const resolved = theme === "system"
      ? (window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light")
      : theme;

    root.classList.remove("light", "dark");
    root.classList.add(resolved);
    root.setAttribute("data-theme", resolved);
    localStorage.setItem("theme", theme);
  }, [theme]);

  return (
    <ThemeContext.Provider value={{ theme, setTheme: setThemeState }}>
      {children}
    </ThemeContext.Provider>
  );
}

export const useTheme = () => {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
};
```

```css
/* globals.css — CSS variable tokens per theme */
:root, .light {
  --color-bg:           #ffffff;
  --color-bg-subtle:    #f9fafb;
  --color-text:         #111827;
  --color-text-muted:   #6b7280;
  --color-border:       #e5e7eb;
  --color-primary:      #2563eb;
  --color-primary-hover:#1d4ed8;
}

.dark {
  --color-bg:           #0f172a;
  --color-bg-subtle:    #1e293b;
  --color-text:         #f1f5f9;
  --color-text-muted:   #94a3b8;
  --color-border:       #334155;
  --color-primary:      #60a5fa;
  --color-primary-hover:#93c5fd;
}
```

---

## 7. Typography scale

```typescript
// components/ui/Typography.tsx
import { cva, type VariantProps } from "class-variance-authority";

const typographyVariants = cva("", {
  variants: {
    variant: {
      h1:      "scroll-m-20 text-4xl font-bold tracking-tight lg:text-5xl",
      h2:      "scroll-m-20 text-3xl font-semibold tracking-tight",
      h3:      "scroll-m-20 text-2xl font-semibold tracking-tight",
      h4:      "scroll-m-20 text-xl font-semibold tracking-tight",
      p:       "leading-7",
      lead:    "text-xl text-gray-600 dark:text-gray-400",
      large:   "text-lg font-semibold",
      small:   "text-sm font-medium leading-none",
      muted:   "text-sm text-gray-500 dark:text-gray-400",
      code:    "relative rounded bg-gray-100 px-1 py-0.5 font-mono text-sm dark:bg-gray-800",
    },
  },
  defaultVariants: { variant: "p" },
});

const TAG_MAP: Record<string, string> = {
  h1:"h1", h2:"h2", h3:"h3", h4:"h4", p:"p", lead:"p",
  large:"p", small:"p", muted:"p", code:"code",
};

export function Typography({ variant = "p", className, children, ...props }: TypographyProps) {
  const Tag = (TAG_MAP[variant] ?? "p") as keyof JSX.IntrinsicElements;
  return <Tag className={cn(typographyVariants({ variant }), className)} {...props}>{children}</Tag>;
}
```

---

## 8. Design token audit

```bash
# Find hardcoded colour values in components (should use tokens/CSS vars)
grep -rn "#[0-9a-fA-F]\{3,6\}\|rgb(\|rgba(" \
  --include="*.tsx" --include="*.ts" src/components/ \
  | grep -v "node_modules\|\.stories\.\|test\|tokens\." | head -30

# Find hardcoded pixel values that should use spacing tokens
grep -rn "style={{" --include="*.tsx" src/ \
  | grep -E "padding|margin|gap|width|height" \
  | grep -v node_modules | head -20

# Find Tailwind arbitrary values (should be tokens)
grep -rn "\[#\|w-\[.\|h-\[.\|p-\[.\|m-\[." \
  --include="*.tsx" src/components/ \
  | grep -v node_modules | head -20

# Coverage: % of components with Storybook stories
total=$(find src/components -name "*.tsx" | grep -v "\.stories\.\|\.test\.\|index" | wc -l)
stories=$(find src/components -name "*.stories.tsx" | wc -l)
echo "Storybook coverage: $stories / $total components ($(( stories * 100 / (total+1) ))%)"
```

---

## 9. Responsive layout audit

```bash
# Find fixed pixel widths (anti-responsive)
grep -rn "width: [0-9]\+px\b" --include="*.css" --include="*.tsx" \
  src/ | grep -v "node_modules\|min-width\|max-width\|border" | head -20

# Find missing responsive variants in Tailwind
grep -rn "className=" --include="*.tsx" src/ \
  | grep -v node_modules \
  | python3 -c "
import sys, re
issues = []
for line in sys.stdin:
    classes = re.findall(r'className=[\"\'](.*?)[\"\'`]', line)
    for cls in classes:
        parts = cls.split()
        # Find layout classes without responsive prefix
        layout = [p for p in parts if re.match(r'^(flex|grid|hidden|block|inline)', p)
                  and not any(prefix in p for prefix in ['sm:','md:','lg:','xl:'])]
        if layout and len(parts) > 5:  # only flag in component-rich classNames
            issues.append(f'{line.strip()[:80]}')
for i in issues[:10]: print(i)
" 2>/dev/null
```
