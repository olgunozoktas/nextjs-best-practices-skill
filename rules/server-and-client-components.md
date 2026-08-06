# The server/client boundary

Sources: [`/getting-started/server-and-client-components`](https://nextjs.org/docs/app/getting-started/server-and-client-components) ·
[`/guides/rendering-philosophy`](https://nextjs.org/docs/app/guides/rendering-philosophy) ·
[`/guides/interactive-apps`](https://nextjs.org/docs/app/guides/interactive-apps)
Next.js `16.3.0`, read `2026-08-06`.

## Server by default, and that is the decision to preserve

Components are Server Components unless a `'use client'` boundary puts them on the client. Keep
it that way: server rendering keeps data access, secrets and dependencies off the wire.

**`'use client'` is a boundary, not a file annotation.** Every module imported *into* a client
module becomes part of the client graph. So a `'use client'` at the top of a layout drags the
subtree with it.

**Push the boundary down to the smallest interactive leaf.** The interactive thing is usually a
button, a menu, a form field — not the page. A page marked `'use client'` because one control
needs an event handler ships the whole page and forfeits server data access for all of it.

## Composition beats conversion

When a server-rendered thing needs to live inside an interactive shell, **pass it as
`children`** rather than making the parent a Client Component:

```tsx
// Server Component page
<Collapsible>            {/* client: holds the open/closed state */}
  <ExpensiveServerList /> {/* stays on the server */}
</Collapsible>
```

The client component renders `{children}` without needing to know what they are. This is the
single most useful pattern in the App Router and it is the one most often missed — the instinct
is to convert the parent.

## What crosses the boundary is a wire format

Props to a Client Component are serialised and **shipped to the browser, where they are
readable.** Two consequences:

- **Serialisable only.** No functions (except Server Actions), no class instances, no `Map`/`Set`
  unless supported, no `Date` assumptions — check before relying on one.
- **Everything you pass is public.** This is a security property, not a performance one: pass
  DTOs, never database rows. See [`data-security.md`](data-security.md).

**A prop you pass "just in case" is data you published.** The `user` object with a
`stripe_customer_id` on it is the standard example.

## Hydration

The server HTML and the first client render must match. The usual culprits are exactly the
things a cached scope must also avoid — `Date.now()`, `Math.random()`, `localStorage`,
`window`, locale- or timezone-dependent formatting.

- **Render locale/timezone-dependent values with an explicit locale and zone** passed from the
  server, so both sides compute the same string. Letting `Intl` default means the server uses
  its locale and the browser uses theirs, and they differ.
- **Read browser-only state in an effect**, after mount, and render a stable placeholder first.
- Suppressing a hydration warning is a last resort for genuinely unavoidable mismatches (a
  timestamp), not a fix.

## Where to fetch

Fetch in Server Components, near where the data is used, and let the framework dedupe — rather
than fetching high and threading props down. `cache()` around a data function is the documented
way to get the same value in several places (see [`data-security.md`](data-security.md)).

For genuinely client-driven state — something that changes from user interaction after load,
with no navigation — a client fetch library is right. That is a narrower set of cases than it
first appears; a filter that can be a URL search param should be one, because then it is
server-renderable and shareable.
