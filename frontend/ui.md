---
name: frontend-ui
description: >
  Use this skill for all frontend UI tasks: React component architecture, Next.js App
  Router patterns, Vue 3/Nuxt, SvelteKit, Angular, Astro, Core Web Vitals optimisation,
  bundle analysis (Webpack/Vite), image optimisation, lazy loading, PWA/offline support,
  and micro-frontend architecture. Covers component design patterns, state management,
  performance profiling, and accessibility-first implementation.
  Triggers: "React component", "Next.js", "Vue", "Svelte", "Angular", "Astro",
  "frontend performance", "bundle size", "lazy load", "PWA", "micro-frontend",
  "Core Web Vitals", "component architecture", "state management".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Frontend UI Skill

## Why this skill exists

Frontend code written without architectural discipline degrades faster than
any other layer — prop-drilling, God components, uncontrolled re-renders,
and unoptimised bundles compound quickly. This skill provides **framework-specific
patterns, performance tooling, and component architecture guidance** that scales
to enterprise codebases.

---

## 0. Project audit

```bash
# Detect framework and versions
cat package.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
all_deps = {**d.get('dependencies',{}), **d.get('devDependencies',{})}
frameworks = ['react','next','vue','nuxt','svelte','@sveltejs/kit','@angular/core','astro']
for f in frameworks:
    if f in all_deps: print(f'{f}: {all_deps[f]}')
"

# Bundle size baseline
du -sh dist/ .next/ build/ 2>/dev/null

# Core Web Vitals via Lighthouse CLI
npx lighthouse http://localhost:3000 --only-categories=performance \
  --output=json --quiet 2>/dev/null \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
cats = d.get('categories', {})
audits = d.get('audits', {})
print(f\"Performance: {int(cats.get('performance',{}).get('score',0)*100)}\")
for key in ['first-contentful-paint','largest-contentful-paint',
            'total-blocking-time','cumulative-layout-shift','speed-index']:
    a = audits.get(key, {})
    print(f'  {key}: {a.get(\"displayValue\",\"?\")}')
" 2>/dev/null
```

---

## 1. React — component architecture patterns

### Compound component pattern
```tsx
// components/Select/index.tsx
import { createContext, useContext, useState } from "react";

const SelectContext = createContext<SelectContextValue | null>(null);

function useSelect() {
  const ctx = useContext(SelectContext);
  if (!ctx) throw new Error("Select subcomponent used outside <Select>");
  return ctx;
}

function Select({ value, onChange, children }: SelectProps) {
  const [open, setOpen] = useState(false);
  return (
    <SelectContext.Provider value={{ value, onChange, open, setOpen }}>
      <div className="relative">{children}</div>
    </SelectContext.Provider>
  );
}

Select.Trigger = function Trigger({ children }: { children: React.ReactNode }) {
  const { open, setOpen, value } = useSelect();
  return (
    <button type="button" aria-haspopup="listbox" aria-expanded={open}
      onClick={() => setOpen(!open)}>
      {value || children}
    </button>
  );
};

Select.Options = function Options({ children }: { children: React.ReactNode }) {
  const { open } = useSelect();
  return open ? <ul role="listbox">{children}</ul> : null;
};

Select.Option = function Option({ value, children }: { value: string; children: React.ReactNode }) {
  const { onChange, setOpen, value: selected } = useSelect();
  return (
    <li role="option" aria-selected={selected === value}
      onClick={() => { onChange(value); setOpen(false); }}>
      {children}
    </li>
  );
};

export { Select };
```

### Smart / dumb component split
```tsx
// DUMB — pure presentation, fully testable, zero side effects
export function UserCard({ user, onEdit, onDelete }: UserCardProps) {
  return (
    <article aria-label={`User: ${user.name}`}>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={() => onEdit(user.id)}>Edit</button>
      <button onClick={() => onDelete(user.id)}>Delete</button>
    </article>
  );
}

// SMART — data fetching, mutations, side effects only
export function UserCardContainer({ userId }: { userId: string }) {
  const { data: user, isLoading } = useQuery({
    queryKey: ["users", userId],
    queryFn: () => fetchUser(userId),
  });
  const { mutate: editUser }   = useMutation({ mutationFn: updateUser });
  const { mutate: deleteUser } = useMutation({ mutationFn: removeUser });

  if (isLoading) return <UserCardSkeleton />;
  if (!user) return <NotFound />;
  return <UserCard user={user} onEdit={editUser} onDelete={deleteUser} />;
}
```

### Custom hook — headless logic
```tsx
// hooks/useVirtualList.ts
export function useVirtualList<T>({ items, itemHeight, containerHeight }: VirtualListOptions<T>) {
  const [scrollTop, setScrollTop] = useState(0);
  const startIndex  = Math.max(0, Math.floor(scrollTop / itemHeight) - 1);
  const visibleCount = Math.ceil(containerHeight / itemHeight) + 2;
  const visibleItems = items.slice(startIndex, startIndex + visibleCount);

  return {
    visibleItems,
    totalHeight: items.length * itemHeight,
    offsetY: startIndex * itemHeight,
    onScroll: (e: React.UIEvent<HTMLDivElement>) => setScrollTop(e.currentTarget.scrollTop),
  };
}
```

---

## 2. State management

### Zustand — client state
```typescript
// store/useCartStore.ts
import { create } from "zustand";
import { devtools, persist } from "zustand/middleware";
import { immer } from "zustand/middleware/immer";

export const useCartStore = create<CartStore>()(
  devtools(persist(immer((set) => ({
    items: [] as CartItem[],
    total: 0,

    addItem: (product: Product, qty: number) => set((state) => {
      const existing = state.items.find(i => i.productId === product.id);
      if (existing) { existing.qty += qty; }
      else { state.items.push({ productId: product.id, name: product.name, price: product.price, qty }); }
      state.total = state.items.reduce((s, i) => s + i.price * i.qty, 0);
    }),

    removeItem: (productId: string) => set((state) => {
      state.items = state.items.filter(i => i.productId !== productId);
      state.total = state.items.reduce((s, i) => s + i.price * i.qty, 0);
    }),

    clear: () => set({ items: [], total: 0 }),
  })), { name: "cart-store" }))
);
```

### TanStack Query — server state (never manage server data in Zustand)
```typescript
// lib/query/users.ts
export const userKeys = {
  all:     ["users"] as const,
  lists:   () => [...userKeys.all, "list"] as const,
  list:    (f: UserFilters) => [...userKeys.lists(), f] as const,
  detail:  (id: string) => [...userKeys.all, "detail", id] as const,
};

export function useUsers(filters: UserFilters) {
  return useQuery({
    queryKey:    userKeys.list(filters),
    queryFn:     () => fetchUsers(filters),
    staleTime:   5 * 60 * 1000,
    placeholderData: (prev) => prev,   // no loading flash on filter change
  });
}

export function useUpdateUser() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: updateUser,
    onSuccess: (updated) => {
      qc.setQueryData(userKeys.detail(updated.id), updated);
      qc.invalidateQueries({ queryKey: userKeys.lists() });
    },
  });
}
```

---

## 3. Next.js App Router

### Server vs client components
```tsx
// SERVER component (default) — zero JS bundle cost, direct DB access
// app/users/page.tsx
export default async function UsersPage() {
  const users = await db.user.findMany({ where: { active: true }, take: 20 });
  return (
    <main>
      <UserList users={users} />    {/* server — no bundle */}
      <UserSearch />                {/* client — has "use client" */}
    </main>
  );
}

// CLIENT component
// components/UserSearch.tsx
"use client";
import { useTransition } from "react";
import { useRouter } from "next/navigation";

export function UserSearch() {
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  return (
    <input
      placeholder={isPending ? "Searching…" : "Search users"}
      onChange={e => startTransition(() => router.push(`/users?q=${e.target.value}`))}
    />
  );
}
```

### Loading / error boundaries
```tsx
// app/users/loading.tsx — automatic Suspense
export default function Loading() { return <UserListSkeleton count={10} />; }

// app/users/error.tsx — automatic Error boundary
"use client";
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div role="alert">
      <p>Failed to load: {error.message}</p>
      <button onClick={reset}>Retry</button>
    </div>
  );
}
```

### Server Actions (mutations without API routes)
```tsx
// app/users/actions.ts
"use server";
import { revalidatePath } from "next/cache";

export async function deleteUser(formData: FormData) {
  const id = formData.get("id") as string;
  await db.user.delete({ where: { id } });
  revalidatePath("/users");
}

// app/users/DeleteButton.tsx — use in form
"use client";
export function DeleteButton({ userId }: { userId: string }) {
  return (
    <form action={deleteUser}>
      <input type="hidden" name="id" value={userId} />
      <button type="submit">Delete</button>
    </form>
  );
}
```

---

## 4. Vue 3 / Nuxt

```typescript
// composables/useUsers.ts
export function useUsers(filters: Ref<UserFilters>) {
  const { data, isLoading, error } = useQuery({
    queryKey: computed(() => ["users", filters.value]),
    queryFn:  () => fetchUsers(filters.value),
  });
  const sorted = computed(() => [...(data.value ?? [])].sort((a, b) => a.name.localeCompare(b.name)));
  return { users: sorted, isLoading, error };
}
```

```vue
<!-- components/UserList.vue -->
<script setup lang="ts">
const filters = ref({ active: true });
const { users, isLoading } = useUsers(filters);
</script>
<template>
  <ul v-if="!isLoading" aria-label="User list">
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
  <UserListSkeleton v-else />
</template>
```

---

## 5. SvelteKit

```typescript
// routes/users/+page.server.ts
export const load: PageServerLoad = async () => {
  const users = await db.user.findMany({ where: { active: true } });
  return { users };
};
```

```svelte
<!-- routes/users/+page.svelte -->
<script lang="ts">
  import type { PageData } from "./$types";
  export let data: PageData;
</script>

{#each data.users as user (user.id)}
  <UserCard {user} />
{/each}
```

---

## 6. Bundle analysis and code splitting

```bash
# Next.js bundle analyser
ANALYZE=true npx next build 2>/dev/null

# Vite — rollup visualiser
npx vite build 2>/dev/null
# (configure vite-plugin-rollup-visualizer in vite.config.ts)

# Find largest JS chunks
find .next/static/chunks dist/assets -name "*.js" 2>/dev/null \
  | xargs wc -c | sort -rn | head -15

# Detect heavy deps that should be replaced
grep -rn "from 'lodash'\|from 'moment'\|from 'underscore'" \
  --include="*.tsx" --include="*.ts" src/ 2>/dev/null | head -10
# lodash → native JS  |  moment → date-fns  |  underscore → native
```

```tsx
// Dynamic import — code split at component level
import dynamic from "next/dynamic";
const HeavyChart = dynamic(() => import("./HeavyChart"), {
  loading: () => <ChartSkeleton />,
  ssr: false,
});

// React.lazy equivalent
const DataGrid = lazy(() => import("./DataGrid"));
```

---

## 7. Image optimisation

```tsx
// Next.js Image — always use instead of <img>
import Image from "next/image";

// LCP image — add priority
<Image src="/hero.jpg" width={1200} height={600} alt="Hero" priority
  sizes="(max-width: 768px) 100vw, 1200px" />

// Dynamic images — use fill + aspect container
<div className="relative aspect-video w-full">
  <Image src={product.imageUrl} alt={product.name} fill
    sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
    className="object-cover" />
</div>
```

---

## 8. PWA / offline

```typescript
// vite.config.ts
import { VitePWA } from "vite-plugin-pwa";

VitePWA({
  registerType: "autoUpdate",
  workbox: {
    globPatterns: ["**/*.{js,css,html,ico,png,woff2}"],
    runtimeCaching: [
      {
        urlPattern: /^https:\/\/api\.myapp\.com\//,
        handler: "NetworkFirst",
        options: { cacheName: "api-cache", networkTimeoutSeconds: 5 },
      },
    ],
  },
  manifest: {
    name: "My App",
    short_name: "MyApp",
    theme_color: "#ffffff",
    icons: [
      { src: "/icon-192.png", sizes: "192x192", type: "image/png" },
      { src: "/icon-512.png", sizes: "512x512", type: "image/png", purpose: "maskable" },
    ],
  },
})
```

---

## 9. Micro-frontend (Module Federation)

```typescript
// apps/shell/webpack.config.ts
new ModuleFederationPlugin({
  name: "shell",
  remotes: {
    checkout:  "checkout@https://checkout.myapp.com/remoteEntry.js",
    catalogue: "catalogue@https://catalogue.myapp.com/remoteEntry.js",
  },
  shared: { react: { singleton: true, requiredVersion: "^18" }, "react-dom": { singleton: true } },
});

// apps/checkout/webpack.config.ts
new ModuleFederationPlugin({
  name: "checkout",
  filename: "remoteEntry.js",
  exposes: { "./CheckoutFlow": "./src/CheckoutFlow" },
  shared: { react: { singleton: true }, "react-dom": { singleton: true } },
});
```

---

## 10. Core Web Vitals fix guide

| Metric | Target | Top fixes |
|--------|--------|-----------|
| LCP | < 2.5s | `priority` on hero image, preload fonts, remove render-blocking JS |
| CLS | < 0.1 | Reserve space for images (`width`/`height` or `aspect-ratio`), avoid dynamic content injection above fold |
| INP | < 200ms | `useTransition` for non-urgent updates, debounce heavy handlers, move computation to Web Worker |
| FCP | < 1.8s | Inline critical CSS, preload key resources, reduce TTFB |
| TBT | < 200ms | Code-split large bundles, defer non-critical JS |

```bash
# Quick CLS audit — images missing dimensions
grep -rn "<Image\|<img" --include="*.tsx" --include="*.jsx" src/ \
  | grep -v "width=\|fill\|node_modules" | head -20

# Quick INP audit — heavy onChange handlers without debounce
grep -rn "onChange\|onInput\|onKeyUp" --include="*.tsx" src/ \
  | grep -v "debounce\|throttle\|useTransition\|node_modules" | head -20
```
