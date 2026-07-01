# Frontend (4-15)
### React, Next.js, Routing & Data Fetching, State, Styling, Forms, Build & Performance, Accessibility, Testing

---

# 4. React Fundamentals & Evolution

React has gone through several distinct paradigm shifts since 2016. Understanding this history isn't just trivia — interviewers expect you to articulate *why* things changed and what problem each version solved. The arc goes: class-component complexity → hooks-based composition → concurrent rendering → server-first components.

## React Fundamentals (Refresher)

Before the version-by-version history, here's the mental model the whole library rests on — the things that haven't changed since 2016 and that interviewers assume you can explain cold.

- **Components & JSX.** A component is a function that takes `props` and returns a description of UI (JSX). JSX is sugar for `React.createElement` calls — it produces plain objects (the "React element" tree), not DOM. React reads that tree and decides what to do.
- **Props vs state.** Props are inputs passed *down* from a parent — read-only to the receiver. State is data a component *owns* and can change over time (`useState`). Changing either triggers a re-render of that component and its descendants.
- **One-way data flow.** Data flows down via props; changes flow up via callbacks. There is no two-way binding — a child never mutates a parent's state directly, it calls a function the parent gave it.
- **Reconciliation & the virtual DOM.** On each render React builds a new element tree and *diffs* it against the previous one, then applies the minimal set of real DOM mutations. This diffing is the "virtual DOM." Keys tell React which list items are the same across renders — **never use array index as a key** for dynamic lists, or React will reuse the wrong DOM node and state.
- **Render vs commit.** Render = React calls your component to compute the new tree (must be pure, no side effects). Commit = React writes the changes to the DOM. `useEffect` runs *after* commit; this is why you read layout in `useLayoutEffect` (before paint) but do data fetching in `useEffect`.
- **Controlled vs uncontrolled.** A controlled input's value lives in React state; an uncontrolled input's value lives in the DOM (read via ref). Covered in depth in §12.
- **Composition over inheritance.** React has no component inheritance. You reuse logic via custom hooks and reuse UI via composition (`children`, render props, passing components as props).
- **Rules of hooks.** Hooks must be called at the top level of a component (never in loops/conditions) and only from React functions. React identifies hook state *by call order*, which is why the order must be stable across renders.

## Timeline

| Version    | Year | Key Changes                                                                                                                  |
| ---------- | ---- | ---------------------------------------------------------------------------------------------------------------------------- |
| React 15   | 2016 | Class components, string refs, stable foundation                                                                             |
| React 16   | 2017 | FIBER rewrite, Error Boundaries, Portals, Fragments, better SSR                                                              |
| React 16.3 | 2018 | New Context API, createRef, forwardRef, new lifecycles (deprecate cWM/cWU/cWRP)                                              |
| React 16.6 | 2018 | React.lazy + Suspense (code-splitting only), React.memo                                                                      |
| React 16.8 | 2019 | **HOOKS** — useState, useEffect, useReducer, useCallback, useMemo, useRef, useLayoutEffect                                   |
| React 17   | 2020 | Bridge release (no new features), new JSX Transform, event delegation at root                                                |
| React 18   | 2022 | **CONCURRENT** — createRoot, Auto Batching, useTransition, useDeferredValue, useId, Streaming SSR, Selective Hydration       |
| React 19   | 2024 | **ACTIONS** — useActionState, useFormStatus, useOptimistic, use() API, React Compiler, Server Components stable, ref as prop |

## The Major Eras Explained

**React 16 — Fiber (2017):** The entire rendering engine was rewritten. The old synchronous reconciler (Stack) could not pause work mid-render, causing dropped frames during heavy renders. Fiber replaced it with an interruptible, priority-based algorithm. This made Error Boundaries possible and laid the groundwork for everything that came after. At the time, the visible impact was incremental — the big payoff came in React 18.

**React 16.8 — Hooks (2019):** The most significant API shift in React's history. Class components had two fundamental problems: lifecycle methods forced unrelated logic to live together (`componentDidMount` doing both data fetching and event subscriptions), and stateful logic couldn't be reused between components without render props or higher-order components. Hooks solved both: each hook call is a focused concern, and custom hooks allow clean composition. After 16.8, class components became legacy code.

**React 18 — Concurrent Mode (2022):** Fiber's interruptibility was finally exposed to developers. React can now pause rendering to handle urgent updates (user typing) before finishing lower-priority ones (filtering a large list). This made `useTransition` and `useDeferredValue` possible. Streaming SSR lets the server send HTML progressively instead of waiting for the full page to be ready.

**React 19 — Actions and Server-First (2024):** Forms and mutations have always been awkward in React — you needed `useState` for pending state, `useEffect` for side effects, and manual try/catch for errors. React 19 adds `useActionState`, `useFormStatus`, and `useOptimistic` to handle the full async action lifecycle declaratively. The React Compiler (RC) makes manual `useMemo`/`useCallback` largely unnecessary by auto-memoizing at build time.

## Mental Model Shifts

| Era            | Mental Model                                                     |
| -------------- | ---------------------------------------------------------------- |
| **React 15**   | Classes, lifecycle spaghetti, mixins                             |
| **React 16**   | Fiber = interruptible rendering. Error Boundaries.               |
| **React 16.8** | Functions + Hooks replace classes. Custom hooks for composition. |
| **React 17**   | Nothing new. No more `import React` needed.                      |
| **React 18**   | Rendering can be interrupted. Urgent vs transition updates.      |
| **React 19**   | Server is first-class. Compiler removes manual memoization.      |

## Re-renders & the React Compiler

A component re-renders when (1) its state changes, (2) its parent re-renders, or (3) its context value changes. The common surprise is #2: **a parent re-render re-renders all children by default**, even children whose props didn't change. React doesn't compare props automatically — it just re-runs the function. Usually that's cheap (rendering is fast; only the *commit* touches the DOM). It becomes a problem when a child is expensive or when a new reference breaks an optimization downstream.

The root cause of most "too many re-renders" bugs is **reference equality**: objects, arrays, and functions created during render are new on every render, so `===` comparisons (and `React.memo`) see them as "changed."

```tsx
// ❌ `onClick` and `style` are new objects every render → memoized Child still re-renders
<Child onClick={() => doThing()} style={{ margin: 8 }} />

// ✅ stabilize references so React.memo can bail out
const handleClick = useCallback(() => doThing(), []);
const style = useMemo(() => ({ margin: 8 }), []);
<Child onClick={handleClick} style={style} />
```

The classic manual tools:
- **`React.memo(Component)`** — skips re-render if props are shallowly equal. Only helps if the props references are stable.
- **`useMemo(fn, deps)`** — caches an expensive computed value between renders.
- **`useCallback(fn, deps)`** — caches a function reference (it's `useMemo` for functions).

The trade-off: manual memoization is easy to get wrong (stale deps, memoizing cheap work, breaking the cache with an unstable dep) and it clutters code. Use the **React DevTools Profiler** to find *actual* re-render hotspots before reaching for it — don't memoize preemptively.

**React Compiler (React 19)** changes the calculus. It's a build-time compiler (formerly "React Forget") that analyzes your components and auto-inserts the equivalent of `useMemo`/`useCallback`/`memo` where safe — so you write straightforward code and get fine-grained memoization for free. It relies on your components following the Rules of Hooks and being pure. Where it's adopted, manual `useMemo`/`useCallback` becomes largely unnecessary (though still valid).

## Error Boundaries

An error boundary is a component that catches JavaScript errors *during render* in its child tree, logs them, and shows a fallback UI instead of crashing the whole app (unmounting the entire tree). They remain the one thing **only class components can do** — there is no hook equivalent — so this is the standard reason you still see a class component in modern code.

```tsx
class ErrorBoundary extends React.Component<{ fallback: React.ReactNode }, { hasError: boolean }> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }  // render fallback
  componentDidCatch(error, info) { logToService(error, info); }     // side effect: report it
  render() { return this.state.hasError ? this.props.fallback : this.props.children; }
}
```

What boundaries **do not** catch: errors in event handlers (use try/catch), async code (promises/`setTimeout`), SSR, and errors thrown inside the boundary itself. In practice most teams use `react-error-boundary` (a maintained wrapper with a `resetKeys`/`onReset` API) rather than hand-rolling.

**Pairing with Suspense:** Suspense handles the *loading* state (a component is waiting on data/code); an error boundary handles the *failed* state. The idiomatic data-fetching shell wraps both: `<ErrorBoundary fallback={<Error/>}><Suspense fallback={<Spinner/>}>{children}</Suspense></ErrorBoundary>`. In the Next.js App Router these map to the `error.tsx` and `loading.tsx` files per route segment.

---


# 5. Every React Hook Explained

Hooks are functions that let you "hook into" React state and lifecycle from function components. Each hook is designed around a single concern — that's the key difference from class lifecycle methods, which bundled multiple concerns into a few callbacks.

## 5.1 Foundational (React 16.8)

### useState

`useState` is React's most fundamental building block. When you call `useState(initialValue)`, React stores the value outside the component function and returns the current value plus a setter. Calling the setter doesn't mutate the value — it tells React to schedule a re-render with the new value. This replace-don't-mutate model is the mental model everything else builds on.

**Why it matters:** Almost every interactive React component uses `useState`. The critical insight interviewers test: setting state is asynchronous — the current render still sees the old value, and the new value only appears on the next render.

```tsx
const [count, setCount] = useState(0);                          // initial value: 0
const [data, setData] = useState(() => expensiveCompute());     // lazy init: fn only runs once

// WRONG: count = 5       — mutating won't trigger a re-render
// RIGHT: setCount(5)     — schedules a re-render with the new value
// RIGHT: setCount(c => c + 1) — functional update when new value depends on old
```

React 18+: state updates are batched everywhere (setTimeout, promises, native events), not just inside React event handlers.

### useEffect

`useEffect` synchronises a component with an external system — a data source, a timer, a DOM subscription. The mental model is NOT "run code on mount/update/unmount". Think: "keep this side effect in sync with these dependencies." When dependencies change, React cleans up the old effect and runs the new one.

**Why it matters:** Misused `useEffect` is the most common source of React bugs. If you're using it to derive state from props, or to synchronise two pieces of React state, you're using it wrong. The dependency array isn't optional decoration — missing dependencies cause stale closures.

```tsx
useEffect(() => {
  const sub = subscribe(id);       // ← set up the subscription with the current id
  return () => sub.unsubscribe();  // ← cleanup runs before next effect or on unmount
}, [id]);                          // ← re-run whenever id changes
```

Runs **after** paint (asynchronous). Strict Mode fires it twice in development to expose missing cleanups. Avoid async functions directly — use an inner async function or `.then()`.

### useReducer

`useReducer` is `useState` for complex or interrelated state. Instead of calling multiple setters, you dispatch named actions to a pure reducer function that computes the next state. The pattern is identical to Redux: `(state, action) => nextState`.

**Why it matters:** When state transitions get complex — multiple values that change together, state that depends on previous state in non-trivial ways — `useReducer` makes the logic testable in isolation and readable at the call site.

```tsx
type Action = { type: "increment" } | { type: "set"; payload: number };

function reducer(state: { count: number }, action: Action) {
  switch (action.type) {
    case "increment":
      return { ...state, count: state.count + 1 };   // ← return a new object, never mutate
    case "set":
      return { ...state, count: action.payload };
    default:
      return state;
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0 });
// dispatch({ type: 'increment' })  ← trigger a named transition
```

Use when: multiple related values, complex transitions, testable state logic.

### useContext

`useContext` reads a value from a React Context — a mechanism for sharing values through the component tree without prop-drilling. When the context value changes, every component consuming it re-renders.

**Why it matters:** Context is suitable for infrequently changing values (theme, locale, current user). For high-frequency state (form inputs, counters), use Zustand or Jotai — every context change re-renders ALL consumers, even those that don't care about the changed part.

```tsx
const ThemeContext = createContext("light");

// React 19: <ThemeContext value="dark"> (no .Provider!)
// React 18 and below: <ThemeContext.Provider value="dark">

const theme = useContext(ThemeContext);  // ← re-renders whenever the context value changes
```

### useRef

`useRef` gives you a mutable container that persists across renders without triggering re-renders when changed. Two distinct use cases: holding a DOM element reference, and storing any mutable value that should survive re-renders but not cause them.

**Why it matters:** Understanding that `ref.current` is mutable and does NOT cause re-renders distinguishes it clearly from state. Useful for tracking previous values, managing focus, or storing timers.

```tsx
const inputRef = useRef<HTMLInputElement>(null);   // ← holds a DOM node; attach with ref={inputRef}
const renderCount = useRef(0);                     // ← mutable counter, zero re-renders when changed

// Access: inputRef.current?.focus()
// NOT: const el = inputRef  ← always read .current
```

React 19: `ref` is a regular prop on all elements — no more `forwardRef` wrapper needed.

### useMemo / useCallback

`useMemo` memoises a computed value; `useCallback` memoises a function. Both accept a dependency array — React skips recomputation when dependencies haven't changed.

**Why it matters:** These are performance tools, not correctness tools. They prevent referential instability — when a new function/object identity on every render breaks a child's `React.memo` or causes unnecessary `useEffect` re-runs. With the React 19 Compiler, these are largely automated and rarely needed manually.

```tsx
const sorted = useMemo(
  () => items.sort((a, b) => a.name.localeCompare(b.name)),  // ← expensive sort
  [items]                                                     // ← only re-sort when items changes
);

const handler = useCallback(
  (id: string) => select(id),  // ← stable function identity
  []                           // ← never changes (select is from outer scope, not reactive)
);
```

### useLayoutEffect

`useLayoutEffect` is identical to `useEffect` except it fires **synchronously** after DOM mutations and **before** the browser paints. This means you can measure the DOM and set state without the user seeing a flash.

**Why it matters:** The only legitimate use case is reading DOM layout (element dimensions, scroll position) and imperatively adjusting it before the user sees the painted result. Using it for data fetching or subscriptions is wrong — it blocks paint.

```tsx
useLayoutEffect(() => {
  const rect = el.current?.getBoundingClientRect();  // ← read layout BEFORE paint
  setPos({ x: rect.left, y: rect.top });             // ← state update before paint = no flash
}, [dep]);
```

## 5.2 Concurrent (React 18)

### useTransition / startTransition

`useTransition` marks a state update as non-urgent. React can interrupt the transition render to handle more urgent updates (like typing in an input). The `isPending` flag is `true` while the transition is in progress.

**Why it matters:** This is how you keep a UI responsive while computing expensive updates. Without `useTransition`, filtering a 10,000-item list on every keystroke blocks the UI. With it, React processes typing immediately and defers the filtering.

```tsx
const [isPending, startTransition] = useTransition();

// Urgent: the input value updates immediately
// Non-urgent: the filtered results update when React gets a chance
startTransition(() => setResults(filterHugeList(query)));

// isPending === true while the transition is still computing
{isPending && <Spinner />}
```

Standalone `startTransition` (imported from React) works outside components — used in routers and libraries.

### useDeferredValue

`useDeferredValue` defers re-rendering a value that's expensive to compute. The component renders with the old value until React has time to compute the new one — similar to `useTransition` but when you receive a prop or value from outside (you don't own the setter).

```tsx
const deferred = useDeferredValue(query);  // ← shows old value while new render is pending
// Use deferred (not query) to drive the expensive render
```

|          | useTransition          | useDeferredValue   |
| -------- | ---------------------- | ------------------ |
| Wraps    | State-setting function | A value            |
| Use when | You own the setter     | You receive a prop |
| Provides | isPending boolean      | Compare old vs new |

### useId

`useId` generates a stable, unique, SSR-safe ID for associating form elements with their labels. The ID is consistent between server and client renders, solving hydration mismatches.

```tsx
const id = useId();   // e.g. ":r3:" — deterministic, unique per component instance
// <label htmlFor={id}>Name</label>
// <input id={id} />
```

## 5.3 React 19 Hooks

### useActionState

`useActionState` manages the full lifecycle of an async action: pending state, result, and error. It replaces the pattern of `useState` + `useEffect` + try/catch for forms. Works natively with HTML `<form action={action}>` and Server Actions.

**Why it matters:** Forms and mutations were always awkward in React. This hook provides a single, consistent API for the async action pattern that's also compatible with progressive enhancement (works without JS).

```tsx
const [state, action, isPending] = useActionState(
  async (prev, formData) => {           // ← reducer: receives previous state + form data
    try {
      await save(formData.get("email"));
      return { error: null, ok: true };
    } catch {
      return { error: "Save failed", ok: false };
    }
  },
  { error: null, ok: false },            // ← initial state
);

<form action={action}>
  <input name="email" disabled={isPending} />
  {state.error && <p>{state.error}</p>}
  <button disabled={isPending}>{isPending ? "Saving..." : "Save"}</button>
</form>
```

### useFormStatus

`useFormStatus` reads the pending state of the nearest parent `<form>`. This lets you build a reusable `SubmitButton` component that automatically knows whether its form is submitting — without prop drilling.

```tsx
import { useFormStatus } from "react-dom";

function SubmitBtn() {
  const { pending } = useFormStatus();     // ← reads the PARENT form's state
  return <button disabled={pending}>{pending ? "..." : "Submit"}</button>;
}

// Works inside any <form action={...}> — no props needed
```

### useOptimistic

`useOptimistic` shows a temporary optimistic value immediately while an async operation is in flight. If the operation succeeds, the real value takes over. If it fails, React reverts to the original value automatically.

**Why it matters:** Optimistic UI makes apps feel instant. Without `useOptimistic`, you'd need complex state management to simulate the result before the server confirms it.

```tsx
const [optimisticTodos, addOptimistic] = useOptimistic(
  todos,                                          // ← the real value from server/state
  (currentTodos, newTodo) => [...currentTodos, newTodo]  // ← how to apply the optimistic update
);

// addOptimistic(newTodo)  — shows immediately
// If the server save fails, todos reverts automatically
```

### use() API

`use()` is a new primitive that reads the value of a Promise or Context anywhere in a render function — including inside `if` statements and loops (unlike all other hooks). For Promises, it suspends the component until the Promise resolves.

**Why it matters:** Unlocks conditional data fetching and conditional context reading, which were impossible with previous hook rules.

```tsx
const user = use(userPromise);    // ← suspends until resolved; wrap parent in <Suspense>
const theme = use(ThemeContext);  // ← like useContext but works in conditionals/loops

// Unlike hooks, use() can appear INSIDE conditionals:
if (showDetails) {
  const details = use(detailsPromise);  // ← valid!
}
```

Promise must be created outside the component (stable identity) — don't create it inline.

---


# 6. Server Components and Suspense

## What Are React Server Components?

React Server Components (RSC) are components that run exclusively on the server — they are never sent as JavaScript to the browser. Instead, they render to a special serialized format that the client can use to build the DOM without re-executing the component logic. The fundamental shift: components are server-first by default, and you opt specific subtrees into the client with `"use client"`.

**Why it matters:** RSC eliminates client-side waterfalls for data fetching (the component fetches its own data on the server, co-located with the rendering logic), reduces JavaScript bundle size (server components contribute zero JS to the client), and enables accessing server-only resources (databases, file system, secrets) directly inside components.

The boundary between server and client is not physical — it's conceptual. A server component can import and render a client component as a child, but a client component can only receive server components as `children` (passed from above), not import them directly.

## Server Components (RSC)

|                      | Server Component (default) | Client Component ("use client") |
| -------------------- | -------------------------- | ------------------------------- |
| DB/filesystem access | Yes                        | No                              |
| Secrets/env vars     | Yes                        | No                              |
| async/await in body  | Yes                        | No                              |
| JS sent to client    | None                       | Yes                             |
| useState/useEffect   | No                         | Yes                             |
| Event handlers       | No                         | Yes                             |
| Browser APIs         | No                         | Yes                             |

**Rules:** Server = default in App Router. `"use client"` marks the boundary where a subtree becomes client-rendered. Props crossing the boundary must be serializable (no functions, no class instances — except Server Actions). A client component can render server components passed as `children`.

## Suspense: React vs Next.js

Suspense is React's mechanism for declaratively specifying a loading state while content is being prepared.

**React (client):** When a component inside `<Suspense>` throws a Promise (via `use()` or lazy loading), React shows the `fallback` until the Promise resolves, then swaps in the real content.

**Next.js App Router (enhanced):** Suspense boundaries become HTTP streaming checkpoints. The server sends the `fallback` HTML immediately, then streams additional `<script>` tags that replace the fallback with the real content as server components finish rendering.

```tsx
<Suspense fallback={<Skeleton />}>
  {/* Server component: streams HTML when its async data resolves */}
  <SlowDataComponent />
</Suspense>
```

The server sends the skeleton HTML in the initial HTTP response (fast Time to First Byte), then streams the resolved content. The user sees a progressively loaded page — not a blank screen waiting for all data.

`loading.tsx` in the App Router is syntactic sugar for a route-level Suspense boundary.

---


# 7. Next.js Fundamentals & Evolution

Next.js has tracked React's evolution closely, with each major version unlocking a new capability from React's core. The Pages Router → App Router transition is the most significant architectural change — it shifted data fetching from route-level lifecycle methods to co-located, component-level async Server Components.

## Next.js Fundamentals (Refresher)

Next.js is a React *framework* — it adds the conventions React deliberately leaves out: routing, data fetching, bundling, and a server runtime. The App Router (the modern default) is built on these primitives:

- **File-based routing.** Folders under `app/` are URL segments; a `page.tsx` makes a segment routable. Dynamic segments use `[id]`, catch-all `[...slug]`, route groups `(group)` organize without affecting the URL.
- **Special files per segment.** `layout.tsx` (shared shell that persists across navigation, doesn't re-render), `page.tsx` (the route UI), `loading.tsx` (Suspense fallback), `error.tsx` (error boundary), `not-found.tsx`, `template.tsx` (like layout but remounts on nav).
- **Server vs Client components.** Everything under `app/` is a **Server Component by default** — runs only on the server, ships zero JS, can `async/await` data directly. Add `'use client'` at the top of a file to opt into a Client Component (needed for state, effects, event handlers, browser APIs). The boundary is one-way: a Client Component can't import a Server Component (but can receive one as `children`).
- **Route Handlers.** `route.ts` files export HTTP verbs (`GET`, `POST`, …) — this is the App Router's API layer, replacing `pages/api`.
- **Middleware.** `middleware.ts` runs on the Edge before a request completes — used for auth gates, redirects, rewrites, and i18n locale detection.
- **Metadata.** Export a `metadata` object or `generateMetadata()` for SEO tags instead of `next/head`.
- **Built-in optimization.** `next/image` (automatic resizing, lazy loading, AVIF/WebP), `next/font` (self-hosted fonts, no layout shift), and automatic code splitting per route.

| Version   | Year | Key Changes                                                                                                 |
| --------- | ---- | ----------------------------------------------------------------------------------------------------------- |
| Next 12   | 2021 | SWC compiler (17x faster), Middleware (Edge), React 18 experimental                                         |
| Next 13   | 2022 | **APP ROUTER (beta)** — app/ dir, layouts, loading/error files, RSC default, Streaming SSR, Turbopack alpha |
| Next 13.4 | 2023 | App Router STABLE, Server Actions alpha, Route Handlers                                                     |
| Next 14   | 2023 | Server Actions STABLE, PPR experimental, Turbopack 53% faster                                               |
| Next 15   | 2024 | React 19, async request APIs, **CACHING OFF BY DEFAULT**, Turbopack stable                                  |

## Pages Router vs App Router

The paradigm shift: Pages Router uses route-level data fetching functions (`getServerSideProps`, `getStaticProps`) that run before the component renders. App Router uses React Server Components — the component itself is `async` and fetches its own data. This means data fetching is co-located with rendering, which enables streaming, nested layouts, and component-level caching.

| Concept       | Pages Router                      | App Router                         |
| ------------- | --------------------------------- | ---------------------------------- |
| Data fetching | getServerSideProps/getStaticProps | async Server Components + fetch    |
| Layouts       | \_app.tsx (re-renders on nav)     | layout.tsx (persistent, no re-render) |
| Loading       | Manual useState/useEffect         | loading.tsx + Suspense             |
| Errors        | \_error.tsx (global)              | error.tsx (per segment)            |
| API routes    | pages/api/                        | app/\*/route.ts                    |
| Metadata      | next/head                         | export metadata / generateMetadata |
| Default       | Client components                 | Server Components                  |
| Mutations     | API route + client fetch          | Server Actions                     |

## Next.js 15 Caching Change

Next.js 14 cached `fetch` requests by default, which surprised many developers when stale data appeared in production. Next.js 15 reversed this: nothing is cached unless you explicitly opt in.

```tsx
// v14: cached by default (may show stale data)
await fetch(url);

// v15: NOT cached by default (always fresh — like SSR)
await fetch(url);

// v15: opt into caching explicitly
await fetch(url, { cache: "force-cache" });            // ← cache forever (SSG-like)
await fetch(url, { next: { revalidate: 3600 } });      // ← revalidate every hour (ISR-like)
```

## Caching Layers

Next.js has four distinct caching layers — understanding them prevents unexpected stale-data bugs:

1. **Request Memoization** — deduplicates identical `fetch()` calls within a single render tree (same URL + options = one network request)
2. **Data Cache** — persists `fetch` results across multiple requests and deployments (controlled by `cache:` / `revalidate:` options)
3. **Full Route Cache** — caches the RSC Payload + rendered HTML for static routes at build time
4. **Router Cache (client)** — keeps recently visited routes in memory on the client (v15 defaults: 0s for dynamic routes, 5 min for static)

## Internationalization (i18n)

The App Router has no built-in i18n config (the Pages Router's `i18n` key was removed). The convention is a `[locale]` dynamic segment plus middleware that detects the locale (from the URL, a cookie, or the `Accept-Language` header) and redirects/rewrites accordingly.

```
app/
  [locale]/
    layout.tsx      // reads params.locale, sets <html lang>, provides messages
    page.tsx
middleware.ts       // detect locale → redirect / rewrite
```

Libraries handle the message catalogs and formatting:
- **next-intl** — the de facto App-Router choice; works in Server Components, gives `useTranslations()` / `getTranslations()`, plus number/date/plural formatting via the ECMAScript `Intl` API.
- **react-intl (FormatJS)** — mature, framework-agnostic, but more client-oriented (uses context/providers, so more `'use client'`).
- **i18next / react-i18next** — large ecosystem, common in SPAs.

**Key concerns interviewers probe:** routing strategy (sub-path `/en/...` vs domain vs cookie), keeping translation work in Server Components to avoid shipping catalogs to the client, locale-aware formatting (`Intl.NumberFormat`, `Intl.DateTimeFormat`), and pluralization (ICU message syntax).

---


# 8. Rendering Methods Compared

Choosing the right rendering strategy is one of the most important architectural decisions in a Next.js application. The choice affects Time to First Byte (TTFB), SEO, content freshness, and infrastructure cost. These are not Next.js-specific concepts — they're general patterns, but Next.js makes them easy to mix on a per-route basis.

| Strategy               | HTML Generated                     | Freshness               | Speed         |
| ---------------------- | ---------------------------------- | ----------------------- | ------------- |
| **SSG**                | Build time                         | Stale until rebuild     | Fastest (CDN) |
| **ISR**                | Build + background regen           | Stale N sec, then fresh | Fast          |
| **SSR**                | Per request                        | Always fresh            | Slower TTFB   |
| **Streaming SSR**      | Per request, chunked               | Fresh, progressive      | Fast TTFB     |
| **CSR**                | Minimal shell                      | Fresh (client fetch)    | Slow FCP      |
| **PPR** (experimental) | Build (static) + request (dynamic) | Best of both            | Best          |

**When to use each:**
- **SSG**: Marketing pages, blog posts, documentation — content changes infrequently, CDN caching is valuable
- **ISR**: Product catalogues, news feeds — content changes but not every second; you can tolerate seconds/minutes of staleness
- **SSR**: Dashboards, user-specific pages, anything requiring request-time data (cookies, auth headers)
- **Streaming SSR**: Any SSR page with slow-loading sections — stream the shell immediately, load heavy parts progressively
- **CSR**: Highly interactive widgets, real-time UIs where SEO doesn't matter
- **PPR**: The future default — static outer shell, dynamic islands only where needed

**Triggers in App Router:**

- SSG: No dynamic functions, cached fetches, `generateStaticParams`
- ISR: `next: { revalidate: N }`
- SSR: `cookies()`, `headers()`, `searchParams`, `cache: 'no-store'`
- Streaming: `<Suspense>` boundaries around async components
- CSR: `'use client'` + `useEffect` for client-only fetching
- PPR: experimental flag + `<Suspense>` defines the static/dynamic boundary

---


# 9. Routing & Data Fetching

This section covers how users move between views and how data arrives. Two worlds exist: **client-side routing** in a single-page app (React Router) where navigation never hits the server, and **file-based / server routing** (Next.js App Router) where navigation can re-run server code. Data fetching follows the same split — client libraries (TanStack Query, SWR) vs Server Components.

## Client-Side Routing (React Router / Remix)

In a plain React SPA there are no routes — you add a router. **React Router** is the standard. Since v6.4 it added "data APIs" (loaders/actions) that fetch data *before* rendering a route, eliminating the classic fetch-in-`useEffect` waterfall.

```tsx
// React Router v6.4+ data router: loader runs before the component renders
const router = createBrowserRouter([
  {
    path: "/users/:id",
    element: <User />,
    loader: ({ params }) => fetch(`/api/users/${params.id}`),   // data ready at render
    action: async ({ request }) => { /* handle <Form> POST */ },
  },
]);

function User() {
  const user = useLoaderData();                 // no loading state to manage
  const navigate = useNavigate();
  return <button onClick={() => navigate("/")}>{user.name}</button>;
}
```

**Remix → React Router v7.** Remix (the full-stack framework built on React Router) merged back into **React Router v7**, which now offers a "framework mode" with SSR, loaders/actions, and file-based routes — a direct alternative to Next.js for teams that want React Router's data model with a server. React Router can still be used in pure "library mode" (client-only SPA).

| | React Router (SPA / library) | Next.js App Router | React Router v7 (framework) |
| --- | --- | --- | --- |
| Routing | Config or file-based | File-based | File-based |
| Rendering | CSR (SPA) | RSC + SSR/SSG | SSR + hydration |
| Data | loaders (client) | Server Components + fetch | loaders (server) |
| Best for | Dashboards behind auth, no SEO | SEO + server-first apps | Remix-style full-stack |

**Interview framing:** "When would you pick a SPA router over Next.js?" — A purely client-rendered app behind a login (internal dashboard, app shell) where SEO/SSR add no value and the simplicity of a static host + client routing wins. Reach for Next.js (or RR7 framework mode) when you need SSR/SSG for SEO, server-side data fetching, or streaming.

## App Router Compatibility

## What Problem Does This Solve?

The App Router introduced Server Components as the default, which broke many UI libraries that assumed they were always running in the browser (they used `window`, `localStorage`, or React hooks that only work client-side). Understanding this compatibility landscape matters when evaluating libraries and debugging `"you're importing a component that requires X"` errors.

The universal fix: wrap any client-only library in a `'use client'` boundary component that you import in your `layout.tsx` or at the point where it's needed.

## UI Libraries

| Library      | Status                                 | Notes                                      |
| ------------ | -------------------------------------- | ------------------------------------------ |
| Ant Design 5 | Works with @ant-design/nextjs-registry | Dot notation broken, FOUC without registry |
| MUI          | Works with @mui/material-nextjs        | Emotion cache needed                       |
| shadcn/ui    | Excellent                              | Built for App Router                       |
| Radix UI     | Excellent                              | Headless, many work as Server Components   |
| Tailwind CSS | Perfect                                | Zero runtime, ideal for RSC                |
| CSS Modules  | Perfect                                | Built-in                                   |

**Universal pattern:** Wrap client-only providers in a `'use client'` Providers component, import in `layout.tsx`:

```tsx
// app/providers.tsx
"use client";
import { ThemeProvider } from "some-ui-lib";
export function Providers({ children }: { children: React.ReactNode }) {
  return <ThemeProvider>{children}</ThemeProvider>;
}

// app/layout.tsx
import { Providers } from "./providers";
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>  {/* ← client boundary wraps everything that needs it */}
      </body>
    </html>
  );
}
```

## TanStack Query / SWR / RTK Query: Still Valid?

**Yes, but role changed.** Server Components handle static display data directly via `async/await`. Client data-fetching libraries handle the things Server Components can't:

- Polling / refetch intervals
- Infinite scroll / client pagination
- Complex optimistic updates
- Offline support
- Window focus refetching

**Hybrid pattern:** Server prefetches initial data via `queryClient.prefetchQuery()` → wraps in `<HydrationBoundary>` → Client component picks up the cached data with `useQuery()` and adds interactivity.

## Data Fetching & Caching Patterns

Beyond *where* you fetch, senior interviews probe *how* you fetch efficiently. The recurring villain is the **request waterfall** — sequential fetches that each wait for the previous one, multiplying latency.

```tsx
// ❌ Waterfall: user must resolve before posts even starts (latency = A + B)
const user = await getUser(id);
const posts = await getPosts(user.id);

// ✅ Parallel: kick both off, then await (latency = max(A, B)) — when they're independent
const [user, settings] = await Promise.all([getUser(id), getSettings(id)]);
```

In Server Components, fetches in *sibling* components run in parallel automatically, but an `await` in a parent blocks its children — so hoist independent fetches or use `Promise.all`. When a child genuinely depends on the parent's data, wrap the slow part in `<Suspense>` so the rest of the page streams immediately instead of blocking on it.

**Prefetching + hydration (TanStack Query).** Fetch on the server during render, dehydrate the cache, and let the client hydrate it — the user sees data instantly and the client cache takes over for refetching:

```tsx
// Server Component
const qc = new QueryClient();
await qc.prefetchQuery({ queryKey: ["todos"], queryFn: getTodos });
return <HydrationBoundary state={dehydrate(qc)}><Todos /></HydrationBoundary>;
// <Todos/> (client) calls useQuery(["todos"]) → no loading flash, then live refetching
```

**Optimistic updates.** Update the UI immediately on a mutation, then roll back if the server rejects it — essential for snappy UX. TanStack Query does this via `onMutate`/`onError`/`onSettled`; React 19 exposes `useOptimistic` (see §5.3).

**Cache invalidation.** The hard part of caching. TanStack Query keys cache entries by `queryKey`; after a mutation you `queryClient.invalidateQueries({ queryKey })` to mark them stale and trigger a refetch. In Next.js the equivalents are `revalidatePath()` / `revalidateTag()` (server) and the `next: { tags }` fetch option. **staleTime vs gcTime:** `staleTime` is how long data is considered fresh (no refetch); `gcTime` (formerly `cacheTime`) is how long unused data lingers in memory before garbage collection.

### Data Fetching Priority Summary

| Topic                                  | Priority      |
| -------------------------------------- | ------------- |
| Server vs client fetching boundary     | **Critical**  |
| Avoiding request waterfalls            | **Important** |
| Parallel vs sequential (`Promise.all`) | **Important** |
| Optimistic updates                     | **Important** |
| Cache invalidation (keys/tags)         | **Important** |
| Prefetch + HydrationBoundary           | **Learn**     |

---


# 10. State Management

State management in React separates into two distinct concerns: **server state** (data fetched from APIs — loading, caching, revalidation) and **client state** (UI state — open modals, selected tabs, wizard progress). Libraries like TanStack Query and SWR own server state in SPA architectures; Redux, Context, and MobX own client state. The important caveat for modern Next.js projects: React Server Components shift data fetching to the server, removing the need for client-side cache management for the initial render — which changes the value proposition of every library in this section. This section covers the full landscape and ends with an honest look at what still earns its place in an RSC world.

## 10.1 Built-in Solutions

### Context + useReducer

The built-in solution — no dependencies needed. `useReducer` provides predictable state transitions; `createContext` + `Provider` makes the state accessible to any descendant.

**When to use:** Global UI state (theme, auth), component-local complex state. **Avoid for:** Frequently changing state shared across many components — every context change re-renders all consumers.

```typescript
type State = { count: number; user: User | null };
type Action =
  | { type: 'INCREMENT' }
  | { type: 'SET_USER'; payload: User };

const AppContext = createContext<{
  state: State;
  dispatch: Dispatch<Action>;
} | null>(null);

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    case 'SET_USER':
      return { ...state, user: action.payload };
    default:
      return state;
  }
}

function AppProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(reducer, { count: 0, user: null });
  return (
    <AppContext.Provider value={{ state, dispatch }}>
      {children}
    </AppContext.Provider>
  );
}
```

## 10.2 Flux Architecture

Flux is the unidirectional data-flow pattern Facebook introduced in 2014 to tame the two-way binding that made MVC React apps hard to reason about as they scaled. The core insight: **data flows in one direction only** — Action → Dispatcher → Store → View — and the View never updates the Store directly. Redux is the canonical implementation of this pattern.

```
User Interaction
      ↓
   Action         { type: 'ADD_TO_CART', payload: item }
      ↓
  Dispatcher      routes the action to all registered stores
      ↓
    Store         pure function: (prevState, action) → nextState
      ↓
    View          re-renders from the new state
```

**Why Flux mattered:** Before it, a button click in one component could trigger cascading mutations across unrelated components through shared mutable objects — bugs that were nearly impossible to reproduce. Flux made every state change an explicit, inspectable event. The same principle drives React's own rendering model: you never mutate state in place, you always describe the next state.

**Interview framing:** "Explain Flux" is the gateway question to Redux. Interviewers want: one-way data flow, actions as plain objects describing what happened, reducers as pure functions that compute the next state, single store as the source of truth.

## 10.3 Redux & Redux Toolkit

Redux is the dominant implementation of Flux. At its core: a single immutable state tree (the **store**), **actions** (plain objects describing what happened), and **reducers** (pure functions computing the next state). Redux remains pervasive in large codebases started before ~2022, and Redux Toolkit significantly reduced its boilerplate.

### Core concepts

The three principles:
1. **Single source of truth** — the entire app state lives in one store.
2. **State is read-only** — the only way to change state is to dispatch an action.
3. **Changes via pure functions** — reducers take `(prevState, action)` and return `newState`; no side effects, no mutation.

```typescript
// Action — plain object describing what happened
const increment    = { type: 'counter/increment' };
const addUser      = { type: 'users/add', payload: { id: 1, name: 'Alice' } };

// Reducer — (state, action) → newState
function counterReducer(state = 0, action: { type: string }) {
  switch (action.type) {
    case 'counter/increment': return state + 1;
    case 'counter/decrement': return state - 1;
    default:                  return state;
  }
}

// Store
import { createStore } from 'redux';
const store = createStore(counterReducer);

store.dispatch({ type: 'counter/increment' });
console.log(store.getState()); // 1

// Subscribe to state changes
store.subscribe(() => console.log(store.getState()));
```

### Redux Toolkit (RTK)

Redux Toolkit is the official, opinionated way to write Redux. It eliminates boilerplate with `createSlice` and uses **Immer** under the hood — letting you write apparently mutable reducer code that actually produces a new state object.

```typescript
import { createSlice, configureStore, PayloadAction } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },          // Immer: safe apparent mutation
    incrementByAmount: (state, action: PayloadAction<number>) => {
      state.value += action.payload;
    },
  },
});

export const { increment, incrementByAmount } = counterSlice.actions;

const store = configureStore({
  reducer: { counter: counterSlice.reducer },
  middleware: (getDefault) => getDefault().concat(analyticsMiddleware),
});

export type RootState   = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

Async operations use `createAsyncThunk`:

```typescript
export const fetchUsers = createAsyncThunk('users/fetchAll', async () => {
  const res = await fetch('/api/users');
  return res.json();
});

const usersSlice = createSlice({
  name: 'users',
  initialState: { items: [] as User[], status: 'idle' as 'idle' | 'loading' | 'succeeded' | 'failed' },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending,   (state) => { state.status = 'loading'; })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items  = action.payload;
      })
      .addCase(fetchUsers.rejected,  (state) => { state.status = 'failed'; });
  },
});
```

### RTK Query

RTK Query is Redux Toolkit's built-in data fetching and caching layer. Define endpoints once; it auto-generates React hooks, manages cache, handles loading state, and invalidates related queries on mutation — no `createAsyncThunk` boilerplate.

```typescript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const usersApi = createApi({
  reducerPath: 'usersApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['User'],
  endpoints: (builder) => ({
    getUsers: builder.query<User[], void>({
      query: () => '/users',
      providesTags: ['User'],
    }),
    createUser: builder.mutation<User, Partial<User>>({
      query: (body) => ({ url: '/users', method: 'POST', body }),
      invalidatesTags: ['User'],   // auto-refetches getUsers after this mutation
    }),
  }),
});

export const { useGetUsersQuery, useCreateUserMutation } = usersApi;
```

**When Redux still makes sense:**
- Existing Redux codebase — migrating to RTK is low-risk; migrating away is high-effort
- Complex middleware needs: analytics pipelines, request deduplication, logging
- Large teams where strict unidirectional flow and action audit trails are required

## 10.4 MobX

MobX takes the opposite philosophy from Redux: instead of explicit actions and immutable state, it uses **observable reactive state** — you mutate objects directly, and MobX automatically tracks which components depend on which observables and re-renders only what changed.

```typescript
import { makeAutoObservable } from 'mobx';
import { observer } from 'mobx-react-lite';

class CartStore {
  items: CartItem[] = [];
  discount = 0;

  constructor() {
    makeAutoObservable(this);  // properties → observable, methods → actions, getters → computed
  }

  addItem(item: CartItem) {
    this.items.push(item);    // direct mutation — MobX handles re-renders automatically
  }

  get total() {
    return this.items.reduce((sum, i) => sum + i.price, 0) * (1 - this.discount);
  }
}

const cart = new CartStore();

const CartView = observer(() => (   // re-renders when any observed value changes
  <div>Total: {cart.total}</div>
));
```

**When MobX makes sense:** Domain-rich applications with complex object graphs (e-commerce, CRM), teams comfortable with OOP patterns, existing MobX codebases. **Tradeoff:** Less predictable than Redux — mutations can happen anywhere; implicit reactivity is ergonomic until something re-renders unexpectedly.

## 10.5 SWR

SWR (stale-while-revalidate) is Vercel's lightweight data-fetching hook. The name is the caching strategy: serve stale data immediately, revalidate in the background, update when fresh data arrives.

```typescript
import useSWR from 'swr';
import useSWRMutation from 'swr/mutation';

const fetcher = (url: string) => fetch(url).then(r => r.json());

function UserProfile({ id }: { id: string }) {
  const { data, error, isLoading, mutate } = useSWR(`/api/users/${id}`, fetcher, {
    revalidateOnFocus: true,
    dedupingInterval:  2000,
  });

  const { trigger } = useSWRMutation(
    `/api/users/${id}`,
    async (url, { arg }: { arg: Partial<User> }) => {
      await fetch(url, { method: 'PATCH', body: JSON.stringify(arg) });
      mutate();
    }
  );

  if (isLoading) return <Spinner />;
  if (error)     return <ErrorView />;
  return <div onClick={() => trigger({ name: 'Alice' })}>{data.name}</div>;
}
```

**SWR vs TanStack Query:**

| | SWR | TanStack Query |
|---|---|---|
| Bundle | ~4 KB | ~13 KB |
| Mutations | Basic (`useSWRMutation`) | Full (optimistic, rollback) |
| Infinite queries | Basic | First-class |
| DevTools | None | Yes |
| Offline support | No | Yes |

SWR wins on simplicity and bundle size. TanStack Query wins on feature breadth. Both become less relevant in RSC apps (see §10.7).

## 10.6 TanStack Query

TanStack Query manages the full lifecycle of server data: fetching, caching, background refetching, and synchronisation. Every piece of server data has a `queryKey` — TanStack Query owns the cache for that key.

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function Users() {
  const queryClient = useQueryClient();

  const { data, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn:  () => fetch('/api/users').then(r => r.json()),
    staleTime: 5000,
    gcTime:    10 * 60 * 1000,
  });

  const mutation = useMutation({
    mutationFn: (newUser: User) =>
      fetch('/api/users', { method: 'POST', body: JSON.stringify(newUser) }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['users'] }),
  });

  if (isLoading) return <Spinner />;
  if (error)     return <Error message={error.message} />;
  return (
    <div>
      {data.map(u => <div key={u.id}>{u.name}</div>)}
      <button onClick={() => mutation.mutate({ name: 'Alice' })}>Add</button>
    </div>
  );
}
```

**Key features:** Caching, request deduplication, background refetch on window focus, polling, optimistic updates with automatic rollback, infinite queries, pagination.

## 10.7 State Management in the RSC Era

React Server Components change the calculus for every library in this section, and it is worth being honest about which problems they no longer solve.

### What RSC removes

In a Next.js App Router app, **data fetching moves to the server**:

```typescript
// Server Component — fetch runs on the server, no client JS bundle needed
async function UserList() {
  const users = await fetch('/api/users', { next: { revalidate: 60 } }).then(r => r.json());
  return users.map((u: User) => <div key={u.id}>{u.name}</div>);
}
```

Server Actions replace client-side mutations:

```typescript
'use server'
async function createUser(formData: FormData) {
  await db.users.create({ name: formData.get('name') as string });
  revalidatePath('/users');
}

// No useQuery or useMutation needed here
<form action={createUser}><button>Add</button></form>
```

**What this eliminates client-side:** the entire `isLoading / data / error` trilogy for initial page data, client-side fetch caches, and most of what TanStack Query / SWR / RTK Query were solving for reads.

### What still needs client-side state

RSC eliminates *server state from the client* — it does not eliminate client state:

- **UI state** — open drawers, selected tabs, dark mode — always client
- **Optimistic UI** — `useOptimistic` for simple cases; TanStack Query's `useMutation` with rollback for complex cases where you need granular control
- **Client-side filtering/sorting** — after the server delivers a list, filtering it locally
- **Multi-step wizard state** — form data spanning multiple steps before submission
- **Real-time** — WebSocket or SSE subscriptions, live cursors, collaborative editing
- **Non-RSC apps** — SPAs built with Vite, React Native, apps that can't adopt App Router

### Practical guidance

| App type | Recommended approach |
|---|---|
| Next.js App Router | Server Components for fetches; Server Actions + `revalidatePath` for mutations; Context or Redux for global UI state only |
| SPA (Vite/CRA) | TanStack Query or SWR for server state; Redux or MobX for complex client state |
| Existing Redux codebase | Migrate to RTK; consider RTK Query for data fetching over raw `createAsyncThunk` |
| Complex domain model | MobX |

**Interview framing:** "When would you reach for TanStack Query vs Server Components?" — In an RSC app, Server Components handle reads and Server Actions handle mutations. TanStack Query earns its place when you need optimistic updates with rollback, real-time polling, or infinite scroll — cases where the client genuinely owns the data lifecycle rather than just displaying server-rendered HTML.

## 10.8 Comparison Table

| Library | Use Case | RSC-relevant? | Bundle |
|---|---|---|---|
| **Context + useReducer** | Global UI state, no deps | Yes — always relevant | 0 |
| **Redux / RTK** | Complex client state, middleware, large teams | Yes — for client state | 9 KB |
| **RTK Query** | CRUD + cache within a Redux app | Limited (RSC handles reads) | (in RTK) |
| **MobX** | OOP domain models, reactive state | Yes — client state | 16 KB |
| **SWR** | Lightweight API fetching (SPA) | No — RSC replaces for RSC apps | 4 KB |
| **TanStack Query** | Rich server state (SPA/CSR), optimistic UI | Partially — mutations + realtime | 13 KB |

## State Management Priority Summary

| Topic | Priority | Notes |
|---|---|---|
| **Context + useReducer** | **Important** | Built-in, always relevant |
| **Flux architecture** | **Deep** | Foundation of Redux; common interview question |
| **Redux core concepts** | **Deep** | Pervasive in existing codebases |
| **Redux Toolkit (RTK)** | **Important** | Modern Redux; Immer, createSlice, async thunks |
| **RTK Query** | **Learn** | Redux's built-in data fetching layer |
| **MobX** | **Know** | Less common but present in enterprise |
| **SWR** | **Know** | Lightweight fetching for SPAs |
| **TanStack Query** | **Deep** | Best-in-class for SPA server state; mutations in RSC apps |
| **RSC + Server Actions** | **Critical** | Replaces client-side data fetching in Next.js App Router |

---


# 11. Styling: Modern CSS, Tailwind, CSS-in-JS

Styling approaches have swung hard over the last decade: global stylesheets → CSS Modules → runtime CSS-in-JS (styled-components, Emotion) → utility-first (Tailwind) → and now *zero-runtime* CSS-in-JS driven by React Server Components. The arrival of RSC is the key plot point: any styling solution that needs to run JavaScript in the browser to inject styles is at odds with components that never ship JS, which is why the industry is converging on Tailwind, CSS Modules, and build-time CSS extraction.

## 11.1 Modern CSS

### Container Queries (2023+)

```css
.card-container {
  container-type: inline-size;
}

@container (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 1fr 2fr;
  }
}
```

**Why it matters:** Component-level responsive design (vs viewport-level media queries).

### :has() Selector (Parent Selector)

```css
/* Style form if it has an invalid input */
form:has(input:invalid) {
  border: 2px solid red;
}

/* Style card if it has a certain child */
.card:has(.badge) {
  padding-top: 2rem;
}
```

### @layer (Cascade Layers)

```css
@layer reset, base, components, utilities;

@layer reset {
  * {
    margin: 0;
    padding: 0;
  }
}

@layer utilities {
  .text-center {
    text-align: center;
  }
}
```

**Why it matters:** Control specificity without !important.

### View Transitions API

```javascript
document.startViewTransition(() => {
  // Update DOM
  updateDOM();
});
```

```css
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 0.5s;
}
```

**Use for:** Smooth page transitions (SPA navigation).

### CSS Variables + calc()

```css
:root {
  --spacing-unit: 8px;
  --max-width: 1200px;
}

.container {
  padding: calc(var(--spacing-unit) * 2);
  max-width: min(var(--max-width), 100% - 2rem);
}
```

## 11.2 Tailwind CSS v4

Tailwind dominates (~70% of job posts). v4 brings major changes.

### Key Features

- **Oxide engine** (Rust, 10x faster)
- **CSS-first config** (no more JS config)
- **Better @apply**
- **Container queries support**

```html
<div class="@container">
  <div class="@lg:grid @lg:grid-cols-2">
    <!-- Content -->
  </div>
</div>
```

### Tailwind Priority

| Topic                             | Priority     |
| --------------------------------- | ------------ |
| Utility-first CSS                 | **Critical** |
| Responsive design (sm:, md:, lg:) | **Critical** |
| Dark mode (dark:)                 | **Critical** |
| Custom plugins                    | **Learn**    |
| JIT mode                          | **Know**     |

## 11.3 CSS-in-JS, CSS Modules & Zero-Runtime

CSS-in-JS lets you co-locate styles with components and use JS values (props, theme) directly in styles. The trade-off is *when* the CSS is generated. **Runtime** libraries (styled-components, Emotion) generate and inject styles in the browser as components render — great DX, but a measurable runtime cost and a poor fit for Server Components. **Zero-runtime** libraries (vanilla-extract, Linaria, Panda CSS) extract styles to static `.css` files at build time — no JS shipped for styling.

### Emotion / styled-components (runtime)

```tsx
// styled-components: tagged-template API
import styled from "styled-components";

const Button = styled.button<{ $primary?: boolean }>`
  padding: 8px 16px;
  background: ${(p) => (p.$primary ? p.theme.colors.brand : "#eee")};
  color: ${(p) => (p.$primary ? "white" : "black")};
`;

// Emotion: css prop (needs the jsx pragma / @emotion/react)
/** @jsxImportSource @emotion/react */
import { css } from "@emotion/react";

const style = (primary: boolean) => css`
  padding: 8px 16px;
  background: ${primary ? "#0070f3" : "#eee"};
`;

<button css={style(true)}>Click</button>;
```

**RSC caveat:** both require `'use client'` — their style injection runs in the browser and relies on React context for theming. In the Next.js App Router you must wire up a style registry (`useServerInsertedHTML`) to avoid a flash of unstyled content (FOUC) during streaming. This friction is the main reason new App Router projects rarely choose runtime CSS-in-JS.

### CSS Modules (build-time, RSC-safe)

```tsx
// Button.module.css → class names are hashed/scoped at build time
import styles from "./Button.module.css";
export function Button() {
  return <button className={styles.primary}>Click</button>;
}
```

Locally-scoped class names, zero runtime, works in Server Components with no extra setup. Built into Next.js and Vite.

### Zero-runtime CSS-in-JS

```ts
// vanilla-extract: TypeScript styles, extracted to static CSS at build
import { style } from "@vanilla-extract/css";
export const button = style({
  padding: "8px 16px",
  background: "#0070f3",
  ":hover": { background: "#0051a8" },
});
```

- **vanilla-extract** — type-safe styles in `.css.ts` files; theming via CSS variables.
- **Linaria** — styled-components-like API, but extracted at build time.
- **Panda CSS** — style props + recipes, compiled to atomic CSS (Chakra team's RSC-friendly successor).

### Choosing an approach

| Approach            | Runtime cost | RSC-friendly | Best for                                  |
| ------------------- | ------------ | ------------ | ----------------------------------------- |
| **Tailwind**        | None         | Yes          | Most new projects; speed + consistency    |
| **CSS Modules**     | None         | Yes          | Teams that prefer plain CSS, scoped       |
| **vanilla-extract** | None         | Yes          | Type-safe design systems, theming         |
| **Emotion / SC**    | Yes          | No (client)  | Existing SPA/Pages Router; dynamic theming|

**Interview framing:** "Why did runtime CSS-in-JS fall out of favor?" — Two reasons: (1) it adds a serialization/injection cost on every render and ships a styling runtime to the client; (2) it fundamentally conflicts with Server Components, which aim to ship *zero* JS. The modern answer is utility-first (Tailwind) or build-time extraction (CSS Modules / vanilla-extract).

### Styling Priority Summary

| Topic                          | Priority      |
| ------------------------------ | ------------- |
| Tailwind utility-first         | **Critical**  |
| CSS Modules                    | **Important** |
| Modern CSS (container queries, :has, @layer) | **Learn** |
| Emotion / styled-components    | **Know**      |
| Zero-runtime (vanilla-extract) | **Know**      |
| RSC styling implications       | **Important** |


---


# 12. Form Management

Forms are where React's "UI is a function of state" model meets messy reality: validation, async submission, server errors, and performance with many fields. The first decision is **controlled vs uncontrolled**. A controlled input stores its value in React state (`value` + `onChange`) — every keystroke re-renders. An uncontrolled input lets the DOM hold the value, read via a `ref` or `FormData` on submit — fewer re-renders, less code. React 19 and the form libraries below all lean back toward uncontrolled for performance.

## 12.1 Controlled vs Uncontrolled

```tsx
// Controlled — React state is the source of truth (re-renders per keystroke)
function Controlled() {
  const [email, setEmail] = useState("");
  return <input value={email} onChange={(e) => setEmail(e.target.value)} />;
}

// Uncontrolled — the DOM holds the value; read it on submit
function Uncontrolled() {
  const ref = useRef<HTMLInputElement>(null);
  const onSubmit = () => console.log(ref.current?.value);
  return <input ref={ref} defaultValue="" />;
}
```

**Rule of thumb:** controlled when you need to react to every change (live validation, dependent fields, formatting); uncontrolled (or a library that wraps refs) for large forms where per-keystroke re-renders hurt.

## 12.2 Native Forms + React 19 Actions

React 19 makes plain HTML forms first-class. A `<form action={fn}>` receives a `FormData` object, works progressively (even before hydration), and pairs with `useActionState` for pending/error state and `useFormStatus` for nested submit buttons.

```tsx
"use client";
import { useActionState } from "react";

async function subscribe(prevState, formData: FormData) {
  const email = formData.get("email") as string;
  if (!email.includes("@")) return { error: "Invalid email" };
  await api.subscribe(email);
  return { success: true };
}

function NewsletterForm() {
  const [state, formAction, isPending] = useActionState(subscribe, {});
  return (
    <form action={formAction}>
      <input name="email" type="email" />          {/* uncontrolled — read from FormData */}
      <button disabled={isPending}>Subscribe</button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

The same `action` can be a **Server Action** (`'use server'`), letting the form submit and mutate server data without a client-side fetch.

## 12.3 React Hook Form

The dominant library (2024). It's uncontrolled by default — it registers inputs via refs, so typing doesn't re-render the whole form. Validation integrates with Zod/Yup via a resolver.

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email(),
  age: z.coerce.number().min(18),
});

function SignupForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm({ resolver: zodResolver(schema) });

  return (
    <form onSubmit={handleSubmit((data) => api.signup(data))}>
      <input {...register("email")} />
      {errors.email && <span role="alert">{errors.email.message}</span>}
      <input {...register("age")} />
      <button disabled={isSubmitting}>Sign up</button>
    </form>
  );
}
```

## 12.4 Formik vs React Hook Form

Formik was the standard for years but is **controlled** — it holds all values in state, re-rendering on every keystroke, which becomes a performance problem on large forms. It's also in low-maintenance mode. New projects should default to React Hook Form.

| Aspect          | React Hook Form         | Formik                       |
| --------------- | ----------------------- | ---------------------------- |
| Model           | Uncontrolled (refs)     | Controlled (state)           |
| Re-renders      | Minimal (per field)     | Whole form per keystroke     |
| Bundle size     | ~9 kB                   | ~13 kB + heavier             |
| Validation      | Zod/Yup resolver        | Yup (built-in support)       |
| Maintenance     | Active                  | Minimal                      |
| Verdict         | **Default choice**      | Legacy / existing codebases  |

## 12.5 Validation: Zod vs Yup

Schema validation is shared between forms and APIs. **Zod** is TypeScript-first — the schema *is* the type (`z.infer<typeof schema>`), so client and server validate against one source of truth. **Yup** predates Zod and is still common with Formik, but its type inference is weaker.

```ts
const userSchema = z.object({ name: z.string().min(1), email: z.string().email() });
type User = z.infer<typeof userSchema>;     // ← type derived from schema
userSchema.parse(input);                     // throws on invalid; .safeParse() returns a result
```

**Interview framing:** "How would you handle a 40-field form?" — Use an uncontrolled approach (React Hook Form or native FormData) so typing in one field doesn't re-render the other 39; validate with a Zod schema reused on the server; and surface server-side errors back into the form (`setError` in RHF, or returned state with `useActionState`).

### Form Management Priority Summary

| Topic                              | Priority      |
| ---------------------------------- | ------------- |
| Controlled vs uncontrolled         | **Critical**  |
| React Hook Form + Zod              | **Important** |
| Native forms + React 19 actions    | **Important** |
| Zod schema validation              | **Important** |
| Formik                             | **Know**      |
| Accessible errors (role="alert")   | **Important** |


---


# 13. Build Tools, Monorepos, Performance & Web APIs

The frontend toolchain is the foundation everything else sits on: how code is bundled (Vite, Turbopack, esbuild), how multi-package repos are organized (Turborepo, Nx), how you hit Core Web Vitals (LCP, INP, CLS), and which browser APIs you reach for (Intersection/Resize/Mutation Observers, Web/Service Workers, IndexedDB). Interviewers probe this to see whether you understand *why* the modern stack is fast, not just that it is.

## 13.1 Build Tools

### Vite (De Facto Standard)

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
        },
      },
    },
  },
});
```

**Why Vite won:**

- **Instant server start** (ESM, no bundling in dev)
- **Fast HMR** (granular updates)
- **Optimized builds** (Rollup)
- **Plugin ecosystem**

### Turbopack (Next.js)

- Built in Rust
- Incremental computation (caches everything)
- Next.js default in v15

### esbuild

- 10-100x faster than Webpack
- Used by Vite for deps pre-bundling
- Low-level (not a full build tool)

## 13.2 Monorepos

### Turborepo

```json
// turbo.json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    }
  }
}
```

**Features:**

- Task caching (local + remote)
- Parallel execution
- Dependency graph

### Nx

- More features (code generation, dependency graph viz)
- Steeper learning curve
- Better for very large monorepos (50+ packages)

### Workspace Structure

```
monorepo/
├── apps/
│   ├── web/          # Next.js app
│   ├── api/          # NestJS API
│   └── mobile/       # React Native
├── packages/
│   ├── ui/           # Shared component library
│   ├── utils/        # Shared utilities
│   ├── tsconfig/     # Shared TS configs
│   └── eslint-config/
├── package.json
├── turbo.json
└── pnpm-workspace.yaml
```

### Package Management

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

```json
// apps/web/package.json
{
  "dependencies": {
    "@repo/ui": "workspace:*",
    "@repo/utils": "workspace:*"
  }
}
```

### Changesets (Versioning)

```bash
pnpm changeset      # Create changeset
pnpm changeset version  # Bump versions
pnpm changeset publish  # Publish to npm
```

## 13.3 Performance

### Core Web Vitals (Critical for SEO)

| Metric                              | Target  | Measures                     |
| ----------------------------------- | ------- | ---------------------------- |
| **LCP** (Largest Contentful Paint)  | < 2.5s  | Loading performance          |
| **INP** (Interaction to Next Paint) | < 200ms | Interactivity (replaces FID) |
| **CLS** (Cumulative Layout Shift)   | < 0.1   | Visual stability             |

### Improving LCP

```tsx
// 1. Optimize images
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority  // Preload above-the-fold images
  placeholder="blur"
/>

// 2. Preload critical resources
<link rel="preload" href="/font.woff2" as="font" type="font/woff2" crossOrigin />

// 3. Use CDN
// Next.js automatic image optimization via CDN
```

### Improving INP

```tsx
// 1. Debounce expensive operations
import { useDeferredValue } from "react";

function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(
    () => expensiveFilter(items, deferredQuery),
    [deferredQuery],
  );
  // ...
}

// 2. Virtual lists for long lists
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ items }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });
  // Only renders visible items
}
```

### Improving CLS

```css
/* 1. Reserve space for images */
img {
  aspect-ratio: 16 / 9;
  width: 100%;
  height: auto;
}

/* 2. Use font-display */
@font-face {
  font-family: "CustomFont";
  src: url("/font.woff2") format("woff2");
  font-display: optional; /* or swap */
}
```

### Code Splitting

```tsx
// React.lazy
const Dashboard = lazy(() => import("./Dashboard"));

<Suspense fallback={<Loading />}>
  <Dashboard />
</Suspense>;

// Next.js dynamic imports
import dynamic from "next/dynamic";

const Chart = dynamic(() => import("./Chart"), {
  loading: () => <Skeleton />,
  ssr: false, // Client-only component
});
```

### Bundle Analysis

```bash
# Next.js
npm run build
# Generates .next/analyze/

# Vite
npm install -D rollup-plugin-visualizer
# Add to vite.config.ts
```

### Resource Hints

```html
<!-- DNS prefetch -->
<link rel="dns-prefetch" href="https://api.example.com" />

<!-- Preconnect (DNS + TCP + TLS) -->
<link rel="preconnect" href="https://cdn.example.com" />

<!-- Prefetch (low priority, for next navigation) -->
<link rel="prefetch" href="/page2" />

<!-- Preload (high priority, current page) -->
<link rel="preload" href="/critical.js" as="script" />

<!-- Modulepreload (for ES modules) -->
<link rel="modulepreload" href="/app.js" />
```

## 13.4 Web APIs

### Fetch + AbortController

```typescript
const controller = new AbortController();
const signal = controller.signal;

fetch("/api/data", { signal })
  .then((res) => res.json())
  .catch((err) => {
    if (err.name === "AbortError") {
      console.log("Fetch aborted");
    }
  });

// Cancel the request
controller.abort();

// React hook example
useEffect(() => {
  const controller = new AbortController();

  fetchData(controller.signal);

  return () => controller.abort(); // Cleanup on unmount
}, []);
```

### Intersection Observer (Lazy Loading)

```typescript
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        const img = entry.target as HTMLImageElement;
        img.src = img.dataset.src!;
        observer.unobserve(img);
      }
    });
  },
  {
    rootMargin: "100px", // Load 100px before entering viewport
  },
);

document.querySelectorAll("img[data-src]").forEach((img) => {
  observer.observe(img);
});
```

### Resize Observer

```typescript
const observer = new ResizeObserver((entries) => {
  entries.forEach((entry) => {
    console.log(
      "Element size:",
      entry.contentRect.width,
      entry.contentRect.height,
    );
  });
});

observer.observe(element);
```

### Mutation Observer

```typescript
const observer = new MutationObserver((mutations) => {
  mutations.forEach((mutation) => {
    console.log("DOM changed:", mutation.type);
  });
});

observer.observe(document.body, {
  childList: true,
  subtree: true,
  attributes: true,
});
```

### Web Workers

```typescript
// main.ts
const worker = new Worker("/worker.js");

worker.postMessage({ type: "CALCULATE", data: largeArray });

worker.onmessage = (event) => {
  console.log("Result from worker:", event.data);
};

// worker.js
self.onmessage = (event) => {
  if (event.data.type === "CALCULATE") {
    const result = heavyComputation(event.data.data);
    self.postMessage(result);
  }
};
```

**Use for:** CPU-intensive tasks (image processing, large data filtering).

### Service Workers & PWA

Service Workers are background scripts that run in a separate thread from the page — they can intercept network requests, manage caches, handle push notifications, and enable offline support. They are the technical foundation of **Progressive Web Apps (PWAs)**. The lifecycle is the most important concept to understand; most SW bugs come from misunderstanding when `install`, `activate`, and `fetch` fire.

**Lifecycle:**
```
Register → Install (pre-cache) → Activate (clean old caches) → Idle
                                                                  ↓
                                               Fetch / Push / Sync events
```

```typescript
// App entry point — register the SW
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    const reg = await navigator.serviceWorker.register('/sw.js');
    console.log('SW registered:', reg.scope);
  });
}
```

```typescript
// sw.js — install: pre-cache app shell
const CACHE = 'app-v2';
const PRECACHE = ['/index.html', '/app.js', '/styles.css'];

self.addEventListener('install', (event: ExtendableEvent) => {
  event.waitUntil(caches.open(CACHE).then(c => c.addAll(PRECACHE)));
  self.skipWaiting(); // activate immediately (careful: skips waiting for old tabs to close)
});

// activate: delete caches from previous SW versions
self.addEventListener('activate', (event: ExtendableEvent) => {
  event.waitUntil(
    caches.keys()
      .then(keys => Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k))))
  );
  self.clients.claim(); // take control of uncontrolled pages immediately
});
```

### Workbox caching strategies

[Workbox](https://developer.chrome.com/docs/workbox/) is Google's SW toolkit providing named strategies for common caching patterns:

| Strategy | Behaviour | Best for |
| --- | --- | --- |
| **CacheFirst** | Serve from cache; hit network only if not cached | Hashed JS/CSS/images (static assets) |
| **NetworkFirst** | Try network; fall back to cache on failure | HTML pages, API responses that need to be fresh |
| **StaleWhileRevalidate** | Serve stale immediately; update cache in background | Avatars, non-critical data where slight staleness is fine |
| **NetworkOnly** | Always network, never cache | Payments, analytics, mutations |
| **CacheOnly** | Always cache, never network | Pre-cached offline-only content |

```typescript
import { registerRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';

// Static assets — cache aggressively; content hash busts on deploy
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [new ExpirationPlugin({ maxEntries: 60, maxAgeSeconds: 30 * 24 * 60 * 60 })],
  })
);

// API responses — fresh when online, stale fallback when offline
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({ cacheName: 'api-cache', networkTimeoutSeconds: 3 })
);

// Non-critical UI resources — instant response, background refresh
registerRoute(
  ({ url }) => url.pathname.startsWith('/static/'),
  new StaleWhileRevalidate({ cacheName: 'static' })
);
```

### Background Sync

Background Sync defers actions until the user has connectivity. The browser retries the sync event even if the user closed the tab.

```typescript
// Page: save to IndexedDB then register a sync tag
async function submitOfflineForm(data: FormData) {
  const db = await openDB('pending', 1, {
    upgrade(db) { db.createObjectStore('forms', { autoIncrement: true }); },
  });
  await db.add('forms', data);
  await (await navigator.serviceWorker.ready).sync.register('submit-form');
}

// SW: handle sync event when connectivity resumes
self.addEventListener('sync', (event: SyncEvent) => {
  if (event.tag === 'submit-form') {
    event.waitUntil(flushPendingForms());
  }
});
```

### Push Notifications

Push requires **VAPID keys** (Voluntary Application Server Identification). The flow: subscribe in browser → POST subscription to server → server sends push via the browser's push service → SW receives `push` event and shows notification.

```typescript
// Browser: subscribe and send subscription to server
const reg = await navigator.serviceWorker.ready;
const sub = await reg.pushManager.subscribe({
  userVisibleOnly: true,
  applicationServerKey: urlBase64ToUint8Array(process.env.VAPID_PUBLIC_KEY),
});
await fetch('/api/push/subscribe', { method: 'POST', body: JSON.stringify(sub) });

// SW: receive and display
self.addEventListener('push', (event: PushEvent) => {
  const { title, body, url } = event.data?.json() ?? {};
  event.waitUntil(
    self.registration.showNotification(title, { body, icon: '/icon-192.png', data: { url } })
  );
});

self.addEventListener('notificationclick', (event: NotificationEvent) => {
  event.notification.close();
  event.waitUntil(clients.openWindow(event.notification.data.url));
});
```

### PWA manifest

A `manifest.json` linked in `<head>` is required for the "Add to Home Screen" prompt. With a registered SW + manifest + HTTPS, Chrome shows the install prompt automatically.

```json
{
  "name": "My App",
  "short_name": "App",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#0070f3",
  "background_color": "#ffffff",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

`display: standalone` removes browser chrome — the PWA launches like a native app.

**Interview framing:** "How would you add offline support to a web app?" — Register a SW, pre-cache the app shell with CacheFirst, use NetworkFirst with a stale fallback for API calls, Background Sync for queued writes. The misconception to avoid: "the SW caches everything automatically" — you must explicitly choose a strategy per route.

### IndexedDB

```typescript
const request = indexedDB.open("MyDatabase", 1);

request.onupgradeneeded = (event) => {
  const db = event.target.result;
  const store = db.createObjectStore("users", { keyPath: "id" });
  store.createIndex("email", "email", { unique: true });
};

request.onsuccess = (event) => {
  const db = event.target.result;
  const tx = db.transaction("users", "readwrite");
  const store = tx.objectStore("users");

  store.add({ id: 1, name: "Alice", email: "alice@example.com" });

  const getRequest = store.get(1);
  getRequest.onsuccess = () => {
    console.log(getRequest.result);
  };
};
```

**Use for:** Large client-side data (offline apps, cache).

### WebSockets

```typescript
const ws = new WebSocket("wss://example.com/socket");

ws.onopen = () => {
  ws.send(JSON.stringify({ type: "JOIN", room: "lobby" }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log("Message:", data);
};

ws.onerror = (error) => {
  console.error("WebSocket error:", error);
};

ws.onclose = () => {
  console.log("Connection closed");
};
```

### Server-Sent Events (SSE)

```typescript
const eventSource = new EventSource("/api/events");

eventSource.onmessage = (event) => {
  console.log("Message:", event.data);
};

eventSource.addEventListener("custom-event", (event) => {
  console.log("Custom event:", event.data);
});

eventSource.onerror = () => {
  console.error("Connection error");
};
```

**SSE vs WebSocket:**

- SSE: Server → Client only, automatic reconnection, HTTP-based
- WebSocket: Bidirectional, manual reconnection, TCP-based

### Web Crypto API

```typescript
// Generate random bytes
const array = new Uint8Array(16);
crypto.getRandomValues(array);

// Hash (SHA-256)
const data = new TextEncoder().encode("Hello");
const hashBuffer = await crypto.subtle.digest("SHA-256", data);
const hashArray = Array.from(new Uint8Array(hashBuffer));
const hashHex = hashArray.map((b) => b.toString(16).padStart(2, "0")).join("");
```

## 13.5 Micro-Frontends

Micro-frontends extend the microservices idea to the browser: instead of one large SPA owned by one team, you compose multiple independently deployed frontend fragments — each owned by a different team — into a single user interface. The central problem they solve is **team coupling at the UI layer**: when 10 teams all merge into one Next.js repository, every deploy becomes a coordination exercise and every dependency upgrade is a negotiation. This section covers the main composition strategies, Module Federation, and the genuine trade-offs that make micro-frontends a deliberate architectural choice rather than a default.

### Composition strategies

| Strategy | How | When to use |
| --- | --- | --- |
| **Module Federation** | Webpack 5 / Rspack loads remote modules at run-time | Independent deploys, different teams |
| **Build-time (npm packages)** | Each MFE published as a versioned package | Simple sharing, tight coupling acceptable |
| **Server-side composition** | Edge/ESI/proxy stitches HTML fragments | SSR-heavy, CDN-layer composition |
| **Iframe isolation** | Each MFE in an iframe | Maximum isolation, legacy wrapping |
| **Web Components** | Custom elements as the integration layer | Framework-agnostic team boundaries |

### Webpack Module Federation

Module Federation (Webpack 5 / Rspack) is the dominant run-time approach. A **remote** exposes components; the **host** loads them at run-time without bundling them at build time — each team deploys independently and the host picks up the latest version on the next page load.

```javascript
// product-catalog team — webpack.config.js (remote)
new ModuleFederationPlugin({
  name: 'productCatalog',
  filename: 'remoteEntry.js',
  exposes: {
    './ProductList': './src/components/ProductList',
  },
  shared: {
    react: { singleton: true, requiredVersion: '^18.0.0' },
    'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
  },
});

// shell team — webpack.config.js (host)
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    productCatalog: 'productCatalog@https://products.example.com/remoteEntry.js',
  },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
});
```

```tsx
// Host consumes the remote component lazily
const ProductList = React.lazy(() => import('productCatalog/ProductList'));

export function CatalogPage() {
  return (
    <Suspense fallback={<Spinner />}>
      <ProductList />
    </Suspense>
  );
}
```

**`singleton: true` is critical.** Without it, each MFE loads its own React instance. Two React copies on the same page break hooks with "invalid hook call" errors — hooks must be called from the React that rendered the component.

### Monorepo vs polyrepo for MFEs

| | Monorepo | Polyrepo |
| --- | --- | --- |
| **Dependency management** | Shared, consistent | Each team controls their own |
| **Cross-team atomic changes** | Single PR | PR per repo, coordination overhead |
| **Build isolation** | Needs Nx/Turbo to avoid full rebuilds | Natural per-repo |
| **Independent deploys** | Possible with caching; harder | Natural per-repo |

Most teams doing MFEs at scale choose polyrepo (one repo per domain team) with Module Federation for runtime composition — teams want full deployment autonomy including independent CI/CD pipelines.

### Trade-offs

**Why MFEs:** team autonomy, independent deploy cadence, technology flexibility (one team can use Vue while the shell uses React), isolated blast radius of deployments.

**Why NOT MFEs:** significant operational complexity, shared state across remotes requires explicit contracts, user experience fragmentation if teams diverge on design system, debugging cross-MFE issues is harder, and shared dependencies need careful versioning coordination.

**The key question to ask first:** is the problem team autonomy (a people/process problem) or technical coupling (a code problem)? Micro-frontends solve the former; a well-structured monorepo can solve the latter at much lower cost.

**Interview framing:** "When would you choose micro-frontends over a monorepo?" — When team autonomy and independent deploy cadence matter more than bundle efficiency and operational simplicity, typically at 50+ engineers across genuinely separate product domains. Monorepo with Turborepo/Nx solves most coordination problems at smaller scale.

## Build, Performance & Web API Priority Summary

| Topic                           | Priority                         |
| ------------------------------- | -------------------------------- |
| **Build Tools**                 |                                  |
| Vite                            | **Critical** (de facto standard) |
| Turbopack                       | **Know** (Next.js)               |
| Monorepos (Turborepo, Nx)       | **Important**                    |
| **Micro-Frontends**             |                                  |
| Module Federation               | **Important**                    |
| MFE composition strategies      | **Know**                         |
| Monorepo vs polyrepo for MFEs   | **Know**                         |
| **Performance**                 |                                  |
| Core Web Vitals (LCP, INP, CLS) | **Critical**                     |
| Code splitting / React.lazy     | **Deep**                         |
| Image optimization (WebP/AVIF)  | **Deep**                         |
| Bundle analysis                 | **Deep**                         |
| Resource hints                  | **Learn**                        |
| **Web APIs**                    |                                  |
| Fetch + AbortController         | **Deep**                         |
| Intersection Observer           | **Deep**                         |
| Resize Observer                 | **Learn**                        |
| Mutation Observer               | **Know**                         |
| Web Workers                     | **Learn**                        |
| Service Workers / PWA (Workbox, Background Sync, Push) | **Important**       |
| WebSockets / SSE                | **Important**                    |
| IndexedDB                       | **Know**                         |
| Web Crypto API                  | **Know**                         |


---


# 14. Accessibility (a11y)

Accessibility is no longer optional. Senior engineers are expected to build accessible applications.

## WCAG 2.1 Compliance

### Levels

- **A**: Minimum (must have)
- **AA**: Mid-range (legal requirement for many orgs)
- **AAA**: Highest (nice to have)

**Target for most projects: AA**

## Semantic HTML

```html
<!-- ❌ Bad -->
<div class="button" onclick="submit()">Submit</div>

<!-- ✅ Good -->
<button type="submit">Submit</button>

<!-- ❌ Bad -->
<div class="heading">Title</div>

<!-- ✅ Good -->
<h1>Title</h1>

<!-- ❌ Bad -->
<div class="nav">
  <div><a href="/">Home</a></div>
</div>

<!-- ✅ Good -->
<nav>
  <ul>
    <li><a href="/">Home</a></li>
  </ul>
</nav>
```

**Why it matters:** Screen readers rely on semantic HTML for navigation.

## ARIA (Accessible Rich Internet Applications)

### ARIA Roles

```html
<div role="button" tabindex="0" onclick="...">Click me</div>

<div role="alert">Error: Invalid input</div>

<nav role="navigation" aria-label="Main navigation">
  <!-- ... -->
</nav>
```

**Rule of thumb:** Use native HTML when possible. ARIA is a fallback.

### ARIA Labels

```html
<!-- aria-label -->
<button aria-label="Close dialog">
  <svg>...</svg>
</button>

<!-- aria-labelledby -->
<h2 id="dialog-title">Confirm Delete</h2>
<div role="dialog" aria-labelledby="dialog-title">
  <!-- ... -->
</div>

<!-- aria-describedby -->
<input type="email" aria-describedby="email-help" />
<span id="email-help">We'll never share your email.</span>
```

### ARIA Live Regions

```html
<!-- Announce changes to screen readers -->
<div aria-live="polite" aria-atomic="true">{statusMessage}</div>

<!-- assertive for urgent messages -->
<div aria-live="assertive">{errorMessage}</div>
```

**Use for:** Dynamic content updates (notifications, loading states).

### ARIA States

```html
<button aria-expanded="false" aria-controls="menu" onclick="toggleMenu()">
  Menu
</button>

<ul id="menu" aria-hidden="true">
  <!-- ... -->
</ul>

<input type="text" aria-invalid="true" aria-errormessage="email-error" />
<span id="email-error" role="alert"> Invalid email format </span>
```

## Keyboard Navigation

### Focus Management

```tsx
function Dialog({ isOpen, onClose }) {
  const firstFocusRef = useRef<HTMLButtonElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      previousFocusRef.current = document.activeElement as HTMLElement;
      firstFocusRef.current?.focus();
    } else {
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  return isOpen ? (
    <div role="dialog" aria-modal="true">
      <button ref={firstFocusRef} onClick={onClose}>
        Close
      </button>
      {/* ... */}
    </div>
  ) : null;
}
```

### Focus Trapping

```typescript
function trapFocus(element: HTMLElement) {
  const focusableElements = element.querySelectorAll(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])',
  );

  const firstElement = focusableElements[0] as HTMLElement;
  const lastElement = focusableElements[
    focusableElements.length - 1
  ] as HTMLElement;

  element.addEventListener("keydown", (e) => {
    if (e.key === "Tab") {
      if (e.shiftKey && document.activeElement === firstElement) {
        e.preventDefault();
        lastElement.focus();
      } else if (!e.shiftKey && document.activeElement === lastElement) {
        e.preventDefault();
        firstElement.focus();
      }
    }
  });
}
```

### Skip Links

```html
<a href="#main-content" class="skip-link"> Skip to main content </a>

<main id="main-content">
  <!-- Content -->
</main>
```

```css
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}
```

## Color Contrast

### WCAG AA Requirements

- **Normal text (< 18pt)**: Contrast ratio ≥ 4.5:1
- **Large text (≥ 18pt or 14pt bold)**: Contrast ratio ≥ 3:1

```css
/* ❌ Bad - contrast ratio 2.5:1 */
color: #767676;
background: #ffffff;

/* ✅ Good - contrast ratio 4.6:1 */
color: #595959;
background: #ffffff;
```

**Tools:** Chrome DevTools, Axe DevTools, Contrast Checker

### Don't Rely on Color Alone

```html
<!-- ❌ Bad -->
<span style="color: red;">Error</span>

<!-- ✅ Good -->
<span class="error">
  <svg aria-hidden="true">⚠️</svg>
  <span class="sr-only">Error:</span>
  Invalid input
</span>
```

## Screen Reader Testing

### VoiceOver (macOS)

```
Cmd + F5: Enable/disable
Ctrl + Option + U: Rotor (headings, links, etc.)
Ctrl + Option + Right Arrow: Next element
```

### NVDA (Windows, free)

```
Insert + Down Arrow: Read next line
Insert + F7: Elements list
```

### Screen Reader Only Text

```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

## Forms Accessibility

```html
<!-- Label association -->
<label for="email">Email</label>
<input id="email" type="email" required />

<!-- Fieldsets for groups -->
<fieldset>
  <legend>Shipping Address</legend>
  <label for="street">Street</label>
  <input id="street" type="text" />
  <!-- ... -->
</fieldset>

<!-- Error messages -->
<input
  id="email"
  type="email"
  aria-invalid="true"
  aria-describedby="email-error"
/>
<span id="email-error" role="alert"> Please enter a valid email </span>
```

## Testing Tools

| Tool                      | Priority  | Notes                                   |
| ------------------------- | --------- | --------------------------------------- |
| **Axe DevTools**          | **Learn** | Browser extension, finds ~57% of issues |
| **Lighthouse**            | **Learn** | Built into Chrome, automated audit      |
| **WAVE**                  | **Learn** | Visual feedback overlay                 |
| **Screen readers**        | **Learn** | VoiceOver (Mac), NVDA (Windows)         |
| **Keyboard-only testing** | **Deep**  | Unplug mouse, test navigation           |

## Accessibility Priority Summary

| Topic                            | Priority      |
| -------------------------------- | ------------- |
| WCAG 2.1 AA compliance           | **Important** |
| Semantic HTML                    | **Deep**      |
| ARIA roles, labels, live regions | **Deep**      |
| Keyboard navigation              | **Deep**      |
| Focus management                 | **Deep**      |
| Color contrast                   | **Know**      |
| Screen reader testing            | **Learn**     |
| Axe DevTools, Lighthouse a11y    | **Learn**     |



---


# 15. Testing

Testing is a critical skill for senior engineers. At the senior level, you're expected not just to write tests, but to design systems that are testable, define the right testing strategy for a codebase, and mentor others. Interviewers care about your philosophy as much as your syntax.

## 15.1 Testing Philosophy

### Test Pyramid

The test pyramid describes the ideal distribution of test types. The insight: unit tests are cheap and fast, E2E tests are expensive and brittle. Aim for many unit tests, some integration tests, and few E2E tests.

```
       /\
      /E2E\      ← Few (slow, brittle, expensive — test the most critical paths)
     /------\
    /Integr.\   ← Some (medium speed — test component + API boundaries)
   /----------\
  /   Unit     \ ← Many (fast, isolated — test business logic in isolation)
 /--------------\
```

**Anti-pattern:** Ice cream cone (many E2E, few unit tests) — slow CI, difficult debugging.

### AAA Pattern (Arrange-Act-Assert)

Every test should have three clear phases. This structure makes tests readable and debuggable.

```typescript
test("should add item to cart", () => {
  // Arrange: set up the system under test and its dependencies
  const cart = new ShoppingCart();
  const item = { id: 1, name: "Book", price: 10 };

  // Act: perform the single action being tested
  cart.addItem(item);

  // Assert: verify the outcome
  expect(cart.items).toHaveLength(1);
  expect(cart.total).toBe(10);
});
```

## 15.2 Unit Testing

### Vitest (Modern Jest Alternative)

Vitest is the test runner of choice for Vite-based projects (and increasingly everywhere). It's Jest-compatible (same API), but dramatically faster because it reuses Vite's transformer and module graph. Native TypeScript support, no configuration needed for modern projects.

**Why it matters:** Vitest is rapidly replacing Jest in the ecosystem. If you know Jest, you know Vitest — but understanding why it's faster (Vite's HMR-based module resolution) shows depth.

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { calculateDiscount } from "./pricing";

describe("calculateDiscount", () => {
  it("should apply 10% discount for members", () => {
    const result = calculateDiscount(100, { isMember: true });
    expect(result).toBe(90);
  });

  it("should not apply discount for non-members", () => {
    const result = calculateDiscount(100, { isMember: false });
    expect(result).toBe(100);
  });

  it("should handle edge case of 0 price", () => {
    const result = calculateDiscount(0, { isMember: true });
    expect(result).toBe(0);   // ← 10% of 0 is still 0
  });
});
```

### Test Doubles (Mocks, Stubs, Spies)

Test doubles replace real dependencies to make tests isolated and deterministic. The terminology matters: a spy tracks calls to a real function; a mock replaces a function entirely; a stub replaces a module.

```typescript
import { vi } from "vitest";

// Spy: real function executes, but calls are tracked
const spy = vi.spyOn(console, "log");
myFunction();
expect(spy).toHaveBeenCalledWith("Hello");

// Mock: fake implementation replaces the real one
const mockFn = vi.fn(() => 42);           // ← always returns 42
expect(mockFn()).toBe(42);
expect(mockFn).toHaveBeenCalledTimes(1);

// Stub: replace an entire module
vi.mock("./api", () => ({
  fetchUser: vi.fn(() => Promise.resolve({ id: 1, name: "Alice" })),
}));
```

### Testing Async Code

```typescript
it("should fetch user", async () => {
  const user = await fetchUser(1);
  expect(user.name).toBe("Alice");
});

it("should reject for missing user", async () => {
  await expect(fetchUser(-1)).rejects.toThrow("User not found");  // ← test the rejection
});
```

## 15.3 React Testing

### React Testing Library (RTL)

RTL's philosophy: test your components the way users use them. Query elements by their role, label, or text — not by CSS class or component internals. This makes tests resilient to implementation changes and doubles as accessibility validation (if you can query it by role, it's accessible).

**Why it matters:** RTL is the standard for React component testing. The key principle — test behaviour, not implementation — is what separates maintainable tests from brittle ones.

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserProfile } from './UserProfile';

describe('UserProfile', () => {
  it('should display user name', () => {
    render(<UserProfile user={{ name: 'Alice', age: 30 }} />);
    // getByText queries the DOM like a user scanning the page
    expect(screen.getByText('Alice')).toBeInTheDocument();
  });

  it('should toggle edit mode on button click', async () => {
    const user = userEvent.setup();    // ← userEvent simulates real browser events
    render(<UserProfile user={{ name: 'Alice', age: 30 }} />);

    // getByRole queries by ARIA role — the accessible name
    const editButton = screen.getByRole('button', { name: /edit/i });
    await user.click(editButton);

    expect(screen.getByRole('textbox')).toBeInTheDocument();
  });

  it('should call onSave with updated data', async () => {
    const user = userEvent.setup();
    const onSave = vi.fn();
    render(<UserProfile user={{ name: 'Alice', age: 30 }} onSave={onSave} />);

    await user.click(screen.getByRole('button', { name: /edit/i }));
    await user.clear(screen.getByRole('textbox'));
    await user.type(screen.getByRole('textbox'), 'Bob');    // ← type like a real user
    await user.click(screen.getByRole('button', { name: /save/i }));

    expect(onSave).toHaveBeenCalledWith({ name: 'Bob', age: 30 });
  });
});
```

**Key principles:** Query by role/label; avoid implementation details; test user behaviour.

### Testing Hooks

Custom hooks can be tested in isolation using `renderHook` — it mounts the hook without a visual component.

```typescript
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./useCounter";

it("should increment counter", () => {
  const { result } = renderHook(() => useCounter(0));

  act(() => {
    result.current.increment();    // ← act() wraps state updates
  });

  expect(result.current.count).toBe(1);
});
```

### Testing with TanStack Query

TanStack Query requires a `QueryClient` and `QueryClientProvider` wrapper. Create a fresh `QueryClient` per test to avoid cache bleed-through between tests.

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { renderHook, waitFor } from '@testing-library/react';
import { useUsers } from './useUsers';

it('should fetch users', async () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },  // ← don't retry on failure in tests
  });

  const wrapper = ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );

  const { result } = renderHook(() => useUsers(), { wrapper });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data).toHaveLength(3);
});
```

## 15.4 API Mocking (MSW)

*Backend API testing — Express with supertest (§16.2) and NestJS unit/module/e2e tiers (§16.3) — is covered in the Backend section.*

### MSW (Mock Service Worker)

MSW intercepts HTTP requests at the network layer (via a Service Worker in the browser, or Node's `http` module in tests). This means the same mock definitions work in both your tests and your development browser environment — no conditional fetch mocking logic needed.

**Why it matters:** MSW is the gold standard for mocking APIs. It mocks the network, not the fetch function — so any fetch/axios/whatever your code uses is intercepted transparently.

```typescript
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";

// Define handlers — same format works in browser (Storybook, dev) and Node (tests)
const server = setupServer(
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: 1, name: "Alice" },
      { id: 2, name: "Bob" },
    ]);
  }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());   // ← reset per-test overrides
afterAll(() => server.close());

it("should fetch users from API", async () => {
  const users = await fetchUsers();
  expect(users).toHaveLength(2);
});

it("should handle API error", async () => {
  // Override for this test only
  server.use(
    http.get("/api/users", () => {
      return new HttpResponse(null, { status: 500 });
    }),
  );

  await expect(fetchUsers()).rejects.toThrow();
});
```

## 15.5 E2E Testing

### Playwright

Playwright drives a real browser (Chromium, Firefox, or WebKit) to test your application end-to-end. It has surpassed Cypress in developer mindshare due to multi-browser support, better async handling, and faster test execution. Auto-waits for elements by default — no explicit `wait` calls needed for most interactions.

**Why it matters:** E2E tests are the only way to verify the full user journey works — authentication flows, form submissions, navigation, third-party integrations. Use them for the critical paths: login, checkout, core user workflow.

```typescript
import { test, expect } from "@playwright/test";

test.describe("User Login", () => {
  test("should login successfully", async ({ page }) => {
    await page.goto("http://localhost:3000/login");

    await page.fill('input[name="email"]', "user@example.com");
    await page.fill('input[name="password"]', "password123");
    await page.click('button[type="submit"]');

    // Playwright auto-waits for the URL to change
    await expect(page).toHaveURL("/dashboard");
    await expect(page.getByText("Welcome back!")).toBeVisible();
  });

  test("should show error for invalid credentials", async ({ page }) => {
    await page.goto("http://localhost:3000/login");

    await page.fill('input[name="email"]', "wrong@example.com");
    await page.fill('input[name="password"]', "wrong");
    await page.click('button[type="submit"]');

    await expect(page.getByText("Invalid credentials")).toBeVisible();
  });
});
```

**Playwright Features:** Multi-browser (Chromium/Firefox/WebKit), auto-wait, network interception, screenshots + videos on failure, parallel execution, codegen (record tests from browser).

### Visual Regression Testing

Screenshot-based testing catches unintended visual changes. Playwright's built-in screenshot comparison or dedicated services (Percy, Chromatic) store baseline screenshots and flag visual diffs in CI.

```typescript
test("homepage should match snapshot", async ({ page }) => {
  await page.goto("http://localhost:3000");
  await expect(page).toHaveScreenshot("homepage.png");   // ← fails if pixels differ
});
```

## 15.6 Database Testing

Database integration testing with Testcontainers (a real Docker PostgreSQL/Redis/MongoDB container per test run, destroyed after) is covered in **§16.3** alongside the NestJS module and e2e examples that use it.

## 15.7 Testing Best Practices

### DO

✅ Test behaviour, not implementation
✅ Use descriptive test names ("should X when Y")
✅ Keep tests independent (no shared mutable state between tests)
✅ Test edge cases and error paths
✅ Use factories/fixtures for test data
✅ Mock external dependencies (APIs, databases in unit tests)
✅ Run tests in CI/CD on every commit
✅ Aim for fast feedback (unit tests < 1s per test)

### DON'T

❌ Test private methods directly
❌ Chase 100% coverage (focus on critical paths)
❌ Make tests depend on each other (test order shouldn't matter)
❌ Test library code (React, Express internals)
❌ Over-mock (integration tests should use real code)
❌ Ignore flaky tests (they erode trust in the test suite)
❌ Skip E2E for critical user flows

## 15.8 Coverage

Coverage measures which lines/branches of your code were executed during tests. It's a useful signal but a poor goal — high coverage with shallow assertions can be worse than lower coverage with thorough assertions.

```bash
vitest --coverage
# or jest --coverage
```

**Healthy coverage targets:**
- Critical business logic: 90%+
- Overall codebase: 70-80%
- Don't chase 100% — the last 10-20% often covers error branches that are legitimately hard to trigger

### Mutation Testing (Advanced)

Mutation testing automatically modifies your code (flips `+` to `-`, removes `if` conditions, changes comparisons) and checks whether your tests catch the change. If tests still pass after a mutation, you have a coverage gap even if line coverage shows 100%.

**Tools:** Stryker (JS/TS)

## 15.9 Contract Testing

### Pact (Consumer-Driven Contracts)

In microservices, contract testing verifies that two services' assumptions about each other's API are compatible — without requiring both services to be running simultaneously. The consumer defines what it expects; the provider verifies it can fulfil those expectations.

**Why it matters:** Integration tests between services require both to be up. Contract tests are fast, isolated, and catch API breaking changes before they reach production.

```typescript
import { PactV3 } from "@pact-foundation/pact";

const provider = new PactV3({
  consumer: "UserService",
  provider: "AuthService",
});

it("should get user by id", async () => {
  provider
    .given("user 123 exists")
    .uponReceiving("a request for user 123")
    .withRequest({ method: "GET", path: "/users/123" })
    .willRespondWith({
      status: 200,
      body: { id: 123, name: "Alice" },
    });

  await provider.executeTest(async (mockServer) => {
    const user = await fetchUser(mockServer.url, 123);
    expect(user.name).toBe("Alice");   // ← consumer's assertion
  });
});
```

### Postman / Newman

Postman collections encode API contract expectations as automated tests that run via Newman (the Postman CLI runner) in CI. Use them for smoke and contract testing against real deployed environments (staging, preview deploys) — they validate the actual running API, not a mock.

```javascript
// Postman test tab (JavaScript, runs after each request)
pm.test("Status is 201", () => {
  pm.response.to.have.status(201);
});

pm.test("Response has required fields", () => {
  const body = pm.response.json();
  pm.expect(body).to.have.property("id").that.is.a("number");
  pm.expect(body.email).to.include("@");
});
```

```bash
# Run collection in CI against staging
npx newman run users-api.postman_collection.json \
  --environment staging.postman_environment.json \
  --reporters cli,junit \
  --reporter-junit-export results.xml
```

Pact (consumer-driven) verifies that two services agree on a contract at unit-test speed without both running. Postman/Newman verifies the *actual deployed endpoint* meets its contract — they are complementary: Pact catches breakage pre-merge, Newman catches it post-deploy.

## Testing Priority Summary

| Tool/Concept                         | Priority      | Notes                                   |
| ------------------------------------ | ------------- | --------------------------------------- |
| **Unit Testing**                     |               |                                         |
| Vitest / Jest                        | **Critical**  | Fast, isolated tests                    |
| Test doubles (mocks, stubs, spies)   | **Deep**      | Control dependencies                    |
| AAA pattern                          | **Refresh**   | Arrange-Act-Assert                      |
| **React Testing**                    |               |                                         |
| React Testing Library                | **Critical**  | Test user behavior                      |
| Testing hooks (renderHook, act)      | **Deep**      |                                         |
| **API Mocking**                      |               |                                         |
| MSW (Mock Service Worker)            | **Important** | Network-level mocking, FE tests         |
| **E2E Testing**                      |               |                                         |
| Playwright                           | **Critical**  | Modern E2E, winning market share        |
| Visual regression (Percy/Chromatic)  | **Learn**     |                                         |
| **Contract Testing**                 |               |                                         |
| Pact (consumer-driven)               | **Know**      | Pre-merge, unit-speed                   |
| Postman / Newman                     | **Know**      | Post-deploy, against real env           |
| **Advanced**                         |               |                                         |
| Mutation testing (Stryker)           | **Know**      |                                         |
| Backend testing (unit/module/e2e)    | **Deep**      | See §16.2 (Express) and §16.3 (NestJS)  |
| **Philosophy**                     |               |                                      |
| Test pyramid                       | **Deep**      | Many unit, some integration, few E2E |
| Coverage metrics                   | **Deep**      | 70-80% is good, not 100%             |

---

_(Continuing with sections 12–28 in PART_3–PART_5...)_

---

_End of Part 2. Continue to **Part 3** (Backend) in [`03_Backend_16-23.md`](./03_Backend_16-23.md), or return to the [README](./README.md)._
