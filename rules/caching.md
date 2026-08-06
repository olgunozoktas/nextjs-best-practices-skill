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

## `'use cache'`

Marks a function, component, or route as cacheable. Cached on its **arguments and captured
closure**, so:

- **Keep arguments serialisable and few.** A large object argument is a large cache key.
- **A cached component may not read request state.** `cookies()`, `headers()`, `draftMode()`
  and the session are all out — and if you need per-reader output, the varying part belongs
  outside the boundary, usually behind `<Suspense>`.
- **`'use cache: private'`** exists for per-request caching that *may* read request state; its
  entries are not shared between visitors. Reach for it deliberately, and know it is not a
  shared cache.

## `cacheLife` and `cacheTag`

- **`cacheLife`** sets the profile — how long the entry is fresh, when it revalidates, when it
  expires. **Read the profile's actual numbers rather than assuming from its name**: a profile
  called "days" may revalidate daily and *expire* after a week, which means a stale value can
  be served far longer than the name suggests.
- **`cacheTag`** labels an entry so `revalidateTag` can invalidate it later. **Tag anything you
  will need to invalidate on a write** — deciding after the fact means you cannot target it.
  Tag by entity and id (`project:123`), not by page.

## Static shell, dynamic hole

The App Router's shape is a prerendered shell with dynamic content streamed in. Reading request
state at the top of a page makes the whole route dynamic; reading it inside a `<Suspense>`
boundary keeps the shell static.

**This is the mechanism behind most "why is my page dynamic?" questions**, and the fix is
almost always to move the request-dependent read down into a boundary rather than to remove it.

A related trap: a value that is *cheap but request-dependent* (a locale, the current year) makes
a whole route dynamic if read at the top. Push it behind a boundary, or accept dynamic
rendering deliberately — but know which you chose.

## Two caching models exist right now

The docs ship both a Cache Components guide and a "Caching (Previous Model)" guide, plus a
migration guide. **Determine which model the project is on before changing anything** — advice
from the other model is not merely suboptimal, it is inert or wrong. Check the config and the
Next version, and say which model you are working in.
