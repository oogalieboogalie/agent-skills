# View Transitions in Next.js

## Setup

`<ViewTransition>` works out of the box for `startTransition`/`Suspense` updates. To also animate `<Link>` navigations:

```js
// next.config.js
const nextConfig = {
  experimental: { viewTransition: true },
};
module.exports = nextConfig;
```

This wraps every `<Link>` navigation in `document.startViewTransition`. Any VT with `default="auto"` fires on **every** link click — use `default="none"` to prevent competing animations.

Do **not** install `react@canary` — see SKILL.md "Availability" for details.

---

## Next.js Implementation Additions

When following `implementation.md`, apply these additions:

**After Step 2:** Enable the experimental flag above.

**Step 4:** Use `transitionTypes` on `<Link>` — see "The `transitionTypes` Prop" section below for usage and availability.

**After Step 6:** For same-route dynamic segments (e.g., `/collection/[slug]`), use the `key` + `name` + `share` pattern — see Same-Route Dynamic Segment Transitions below.

---

## Layout-Level ViewTransition

**Do NOT add a layout-level VT wrapping `{children}` if pages have their own VTs.** Nested VTs never fire enter/exit when inside a parent VT — page-level enter/exit will silently not work. Remove the layout VT entirely.

A bare `<ViewTransition>` in layout works only if pages have **no** VTs of their own.

**Layouts persist across navigations** — `enter`/`exit` only fire on initial mount, not on route changes. Don't use type-keyed maps in layouts.

---

## The `transitionTypes` Prop on `next/link`

No wrapper component needed, works in Server Components:

```tsx
<Link href="/products/1" transitionTypes={['transition-to-detail']}>View Product</Link>
```

Replaces the manual pattern of `onNavigate` + `startTransition` + `addTransitionType` + `router.push()`. Reserve manual `startTransition` for non-link interactions (buttons, forms).

**Availability:** `transitionTypes` requires `experimental.viewTransition: true` and is available in Next.js 15+ canary builds and Next.js 16+. If unavailable, use `startTransition` + `addTransitionType` + `router.push()` (see Programmatic Navigation below). To check: `grep -r "transitionTypes" node_modules/next/dist/` — if no results, fall back to programmatic navigation.

---

## Programmatic Navigation

```tsx
'use client';

import { useRouter } from 'next/navigation';
import { startTransition, addTransitionType } from 'react';

function handleNavigate(href: string) {
  const router = useRouter();
  startTransition(() => {
    addTransitionType('nav-forward');
    router.push(href);
  });
}
```

---

## Server-Side Filtering with `router.replace`

For search/sort/filter that re-renders on the server (via URL params), use `startTransition` + `router.replace`. VTs activate because the state update is inside `startTransition`:

```tsx
'use client';

import { useRouter } from 'next/navigation';
import { startTransition } from 'react';

function handleSort(sort: string) {
  const router = useRouter();
  startTransition(() => {
    router.replace(`?sort=${sort}`);
  });
}
```

List items wrapped in `<ViewTransition key={item.id}>` will animate reorder. This is the server-component alternative to the client-side `useDeferredValue` pattern in `patterns.md`.

---

## Two-Layer Pattern (Directional + Suspense)

Directional slides + Suspense reveals coexist because they fire at different moments. Place the directional VT in the **page component** (not layout):

```tsx
<ViewTransition
  enter={{ "nav-forward": "slide-from-right", default: "none" }}
  exit={{ "nav-forward": "slide-to-left", default: "none" }}
  default="none"
>
  <div>
    <Suspense fallback={<ViewTransition exit="slide-down"><Skeleton /></ViewTransition>}>
      <ViewTransition enter="slide-up" default="none"><Content /></ViewTransition>
    </Suspense>
  </div>
</ViewTransition>
```

---

## `loading.tsx` as Suspense Boundary

Next.js `loading.tsx` is an implicit `<Suspense>` boundary. Wrap the skeleton in `<ViewTransition exit="...">` in `loading.tsx`, and the content in `<ViewTransition enter="..." default="none">` in the page:

```tsx
// loading.tsx
<ViewTransition exit="slide-down"><PhotoGridSkeleton /></ViewTransition>

// page.tsx
<ViewTransition enter="slide-up" default="none"><PhotoGrid photos={photos} /></ViewTransition>
```

Same rules as explicit `<Suspense>`: use simple string props (not type maps) since Suspense reveals fire without transition types.

---

## Shared Elements Across Routes

```tsx
// List page
{products.map((product) => (
  <Link key={product.id} href={`/products/${product.id}`} transitionTypes={['nav-forward']}>
    <ViewTransition name={`product-${product.id}`}>
      <Image src={product.image} alt={product.name} width={400} height={300} />
    </ViewTransition>
  </Link>
))}

// Detail page — same name
<ViewTransition name={`product-${product.id}`}>
  <Image src={product.image} alt={product.name} width={800} height={600} />
</ViewTransition>
```

---

## Same-Route Dynamic Segment Transitions

When navigating between dynamic segments of the same route (e.g., `/collection/[slug]`), the page stays mounted — enter/exit never fire. Use `key` + `name` + `share`:

```tsx
<Suspense fallback={<Skeleton />}>
  <ViewTransition key={slug} name={`collection-${slug}`} share="auto" default="none">
    <Content slug={slug} />
  </ViewTransition>
</Suspense>
```

- `key={slug}` forces unmount/remount on change
- `name` + `share="auto"` creates a shared element crossfade
- VT inside `<Suspense>` (without keying Suspense) keeps old content visible during loading

---

## Shared Elements Behind a Data-Fetching `<Suspense>` (the "it only morphs the skeleton" trap)

A shared-element morph needs **both** the source *and* the target named element present **at the navigation snapshot** — the instant the transition starts. If the destination's `<ViewTransition name={...}>` sits **inside** a `<Suspense>` that's waiting on a data fetch, then at snapshot time the destination shell contains only the *skeleton* — the named element doesn't exist yet, no pair forms, and you'll see only the skeleton's auto-named groups animate (never your `name`d group). This looks like "shared elements don't work here," but it's a **placement** problem, not a limitation.

**Fix: hoist the shared element into the shell, fed only by fast/cached data; stream the slow data inside it.** Put the named element in the route's `layout.tsx` (or above the data `<Suspense>`), keyed only by `params`:

```tsx
// app/thing/[slug]/layout.tsx — awaits ONLY params (no slow query)
export default async function Layout({ children, params }: LayoutProps<'/thing/[slug]'>) {
  const { slug } = await params;
  return (
    <ViewTransition name={`thing-${slug}`} share="morph" default="none">
      <article>{children}</article>   {/* the slow header/body stream INSIDE, with their own reveal */}
    </ViewTransition>
  );
}
```

The morph target exists in the shell the moment you navigate (it only needs `slug` from the URL), so it pairs with the list-side card and morphs immediately; the data fills in after. If the shared element is a *visual* (album art, product image) rather than the whole container, back it with a **fast, cached, param-only** query (e.g. just the cover color/URL) and keep the heavy per-entity fetch behind the inner `<Suspense>`.

### Interaction with instant / prerenderable routes (Cache Components)

Placing the shared element in the shell means reading `params` **outside** the data `<Suspense>`. On instant-prerenderable routes this can trip *"params accessed outside of `<Suspense>` prevents the route from being prerendered."* That's expected — the shared-element shell and a fully-streamed instant route pull in opposite directions. Resolve it by keeping the shell's read **param-only and cached/fast** (so the shell is cheap), or opt the route out of instant prerendering where the morph matters. Do **not** conclude the morph is "structurally impossible" — it's the shared element being mis-placed behind the fetch, plus this params-in-shell tradeoff.

## Nested enter/exit for Suspense reveals — `parentEnter` / `parentExit` (experimental, evolving)

The long-standing rule "nested `<ViewTransition>`s don't fire enter/exit inside a parent VT" is being lifted by the experimental **`parentEnter` / `parentExit`** props (and `onParentEnter` / `onParentExit` handlers): a nested VT can define how *it* animates when a **parent** enters/exits, and `parentEnter="none"` stops the propagation into a subtree. Client support exists today (behind `enableViewTransitionParentEnterExit`, i.e. experimental React); SSR/streaming support for **Suspense reveals** was added in React PR #36917 ([commit](https://github.com/react/react/commit/83840902c890f0eb85decda239ef6b1b14945779)). Treat it as a **future/experimental** capability — verify it's in your bundled React before relying on it (`grep -r "vt-parent-enter" node_modules/next/dist/compiled/react*`). It is *not* a fix for root-captured persistent/floating isolation or the scroll-hang.

## Server Components

- `<ViewTransition>` works in both Server and Client Components
- `<Link transitionTypes>` works in Server Components — no `'use client'` needed
- `addTransitionType` and `startTransition` for programmatic nav require Client Components
