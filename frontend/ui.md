---
name: frontend-ui
description: UI component and page implementation across React, Vue, Angular, and Svelte. Use for component architecture decisions, performance patterns (lazy loading, code splitting, memoization), and framework-specific best practices.
---

The user wants to build, review, or refactor UI components or pages. Apply the following guidance.

## Framework Selection Heuristics

When framework is not mandated, recommend based on team size and use-case:

| Scenario | Recommended |
|---|---|
| Large team, enterprise SPA | React + TypeScript |
| Full-stack with SSR/SSG | Next.js (React) or Nuxt (Vue) |
| Lean team, progressive enhancement | Vue 3 + Vite |
| Highly interactive data-heavy UI | SvelteKit |
| Existing Angular codebase | Angular 17+ with Signals |

## React Patterns (Enterprise Grade)

### Component Architecture
```tsx
// ✅ Compound component pattern — keeps API surface small
const DataTable = ({ children }: { children: React.ReactNode }) => { ... }
DataTable.Header = TableHeader
DataTable.Body   = TableBody
DataTable.Footer = TableFooter

// ✅ Render prop for cross-cutting concerns
<DataTable renderEmpty={() => <EmptyState />} />

// ❌ Avoid prop drilling beyond 2 levels — use context or Zustand slice
```

### Performance Checklist
- `React.memo` only on components that re-render frequently with unchanged props
- `useMemo` for expensive derived data (>1ms computation), not for simple transformations
- `useCallback` when passing callbacks to memoized children
- Dynamic `import()` + `React.lazy` for routes and heavy widgets (>50 KB)
- `startTransition` for non-urgent state updates (search, filter)
- Virtualize lists >100 rows with `@tanstack/react-virtual`

### State Architecture
```
Local state     → useState / useReducer
Server state    → TanStack Query (React Query)
Global UI state → Zustand (flat slices, no nesting beyond 2 levels)
Form state      → React Hook Form + Zod validation
URL state       → nuqs (type-safe search params)
```

### TypeScript Strictness
Always use `strict: true` in `tsconfig.json`. Prefer:
```ts
type Props = Readonly<{ label: string; count: number }>  // immutable props
type Status = 'idle' | 'loading' | 'success' | 'error'  // discriminated unions over enums
```

## Vue 3 Patterns

```vue
<!-- ✅ Composables over mixins -->
const { data, isLoading } = useUserList()

<!-- ✅ defineModel for two-way binding in Vue 3.4+ -->
const model = defineModel<string>()

<!-- ✅ Async components for code splitting -->
const HeavyChart = defineAsyncComponent(() => import('./HeavyChart.vue'))
```

## Angular 17+ Patterns

```ts
// ✅ Signals — replaces ChangeDetectionStrategy.OnPush boilerplate
count = signal(0)
doubleCount = computed(() => this.count() * 2)

// ✅ Standalone components — no NgModule required
@Component({ standalone: true, imports: [CommonModule, RouterLink] })
```

## Code Splitting Strategy

```
Route-level splits   → always (enforced by framework router)
Feature-level splits → components >40 KB or used on <20% of page views
Library splits       → chart libs, rich text editors, PDF renderers
Preload strategy     → prefetch on hover/focus, not eagerly
```

## Accessibility (a11y) Baseline

Every component must pass:
1. Keyboard navigation (focus trap in modals, arrow keys in menus)
2. ARIA roles and labels on interactive elements
3. Color contrast ≥ 4.5:1 (WCAG AA)
4. `alt` text on all meaningful images
5. No `outline: none` without a visible focus replacement

Automate with `eslint-plugin-jsx-a11y` (React) or `eslint-plugin-vuejs-accessibility` (Vue).

## Output Format

When generating component code:
1. Show the component file with TypeScript types
2. Show a usage example
3. Note any peer dependencies
4. Flag any a11y or performance concerns inline as `// ⚠️` comments
