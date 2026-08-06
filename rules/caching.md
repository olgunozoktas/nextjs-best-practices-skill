# Caching — Cache Components, and the rule that prevents the expensive bug

Sources: [`/getting-started/caching`](https://nextjs.org/docs/app/getting-started/caching) ·
[`/guides/migrating-to-cache-components`](https://nextjs.org/docs/app/guides/migrating-to-cache-components) ·
[`/guides/caching-without-cache-components`](https://nextjs.org/docs/app/guides/caching-without-cache-components) ·
[`/guides/cdn-caching`](https://nextjs.org/docs/app/guides/cdn-caching) ·
[`/guides/incremental-static-regeneration-cache-components`](https://nextjs.org/docs/app/guides/incremental-static-regeneration-cache-components)
Next.js `16.3.0`, read `2026-08-06`.

## The rule that matters more than any API

> **A cached value must be a function of its cache key. Not the clock, not the request, not the
> reader.**

The reason this deserves top billing: **nothing errors.** A stale-but-plausible value is served
for the whole window, to everyone, and the only symptom is that a number is eventually noticed
to be wrong.

Two shapes, both of which pass review because the offending line is unremarkable:

- **Reader-dependent inside a shared entry** — a price formatted with the first visitor's
  locale, a date in the first visitor's timezone, a list filtered by their permissions.
- **Clock-dependent inside a shared entry** — `new Date().getFullYear()`, "3 hours ago", an
  expiry computed at cache-write time. Correct when written; wrong for the rest of the window.

**The fix is the same shape every time: resolve the varying input OUTSIDE the cached scope and
apply it after.** Cache the expensive invariant part; compute the cheap varying part per
request.

**Do not "fix" it by adding the varying value to the cache key** unless its cardinality is
bounded. A key derived from `Accept-Language`, a user id, or a search string is unbounded or
attacker-influenced, and turns a cache into a memory amplifier. Cache keys include the
function's arguments — which is exactly why an argument is a tempting and dangerous fix.

**Guard it mechanically.** A source-level test forbidding clock, randomness, request and session
reads inside cached scopes costs an hour and catches the class. Review does not: two instances
in two days, in one codebase, is an observed rate.

## `'use cache'` requires `cacheComponents: true` — this is opt-in

**Corrected 2026-08-06.** The first version of this file documented `'use cache'`, `cacheLife`
and `cacheTag` with no prerequisite. There is one:

> "`use cache` is a Cache Components feature. To enable it, add the `cacheComponents` option to
> your `next.config.ts` file."

**So everything in this section is inert on a default Next 16 app.** Check `next.config.ts`
before applying any of it — and see the two-models warning at the foot of this file, which the
earlier version of this section managed to violate.

## `'use cache'`

Marks a function, component, or route as cacheable. Cached on its **arguments and captured
closure**, so:

- **Keep arguments serialisable and few.** A large object argument is a large cache key.
- **A cached component may not read most request state.** `cookies()` and `headers()` are out.
  **`draftMode()` is the documented exception** — corrected here, because the first version of
  this file named it as forbidden: *"You **can** read `isEnabled` from `draftMode()` inside a
  `use cache` scope, however, other runtime APIs like `cookies()` and `headers()` are not
  allowed, even when Draft Mode is active."*
- **`'use cache: private'`** may read runtime request APIs, and its entries are not shared
  between visitors — but the mechanism is not "per-request" as the first version said. Per the
  docs, results are *"never stored on the server, they're cached only in the browser's memory and
  do not persist across page reloads"*, and it is **excluded from the static shell**. Those two
  facts, not the sharing, are what govern when it is usable.
- **`'use cache: remote'`** is the third directive and was missing entirely. It matters on
  serverless: the default in-memory store *"will not persist across instances"*, so `remote`
  moves the entry to a durable shared cache handler — *"a network roundtrip that pays off only at
  a **high hit rate**."*

## `cacheLife` and `cacheTag`

- **`cacheLife`** sets the profile — how long the entry is fresh, when it revalidates, when it
  expires. **Read the profile's actual numbers rather than assuming from its name**: a profile
  called "days" may revalidate daily and *expire* after a week, which means a stale value can
  be served far longer than the name suggests.
- **`cacheTag`** labels an entry so `updateTag` (immediate, Server Actions) or `revalidateTag`
  (stale-while-revalidate) can invalidate it later — see [`revalidation.md`](revalidation.md) for
  which. **Tag anything you
  will need to invalidate on a write** — deciding after the fact means you cannot target it.
  Tag by entity and id (`project:123`), not by page.

## Static shell, dynamic hole — **only with Cache Components enabled**

**This section was stated unconditionally and that was wrong.** With `cacheComponents: true`, a
request-state read inside a `<Suspense>` boundary keeps the shell static:

> "Reading `cookies()` here doesn't opt-in the whole route into dynamic rendering, **the way the
> previous rendering model did**."

**Without Cache Components — i.e. on a default Next 16 app — it does not work:**

> "**Without Cache Components**, reading `cookies()`, `headers()`, or `searchParams` opts **the
> whole route** into dynamic rendering."

So check the flag first. On the previous model the fix is different, and applying this one there
achieves nothing.

A related trap: a value that is *cheap but request-dependent* (a locale, the current year) makes
a whole route dynamic if read at the top. Push it behind a boundary, or accept dynamic
rendering deliberately — but know which you chose.

## Two caching models exist right now

The docs ship both a Cache Components guide and a "Caching (Previous Model)" guide, plus a
migration guide. **Determine which model the project is on before changing anything** — advice
from the other model is not merely suboptimal, it is inert or wrong. Check the config and the
Next version, and say which model you are working in.
