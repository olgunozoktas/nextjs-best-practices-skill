# Revalidation — keeping surfaces honest after a write

Sources: [`/getting-started/revalidating`](https://nextjs.org/docs/app/getting-started/revalidating) ·
[`/guides/how-revalidation-works`](https://nextjs.org/docs/app/guides/how-revalidation-works) ·
[`/guides/incremental-static-regeneration`](https://nextjs.org/docs/app/guides/incremental-static-regeneration)
Next.js `16.3.0`, read `2026-08-06`.

## The default failure is a stale screen after a successful write

The mutation worked, the response was fine, and the list still shows the old data. Users read
that as "it didn't save" and do it again — which is how you get duplicates from a working
feature.

**Every mutation should say what it invalidated.** If the answer is "nothing", that should be a
decision you can point at, not an omission.

## Path vs tag

| Use | When |
|---|---|
| **`revalidatePath('/projects')`** | You know the URL(s) affected and there are few |
| **`revalidateTag('project:123')`** | The data appears on several routes, or you do not know all of them |

**Prefer tags once the same entity appears in more than one place.** Paths do not scale: adding
a sixth route that shows a project means finding every mutation and adding a sixth
`revalidatePath`, and the one you miss is the stale one. Tagging is declared where the data is
read, so a new reader inherits invalidation for free.

**Tag by entity and id, not by page.** `project:123` survives a routing change; `/projects/123`
does not.

## Practices

- **Tag at read time or you cannot invalidate at write time.** This is the ordering people get
  wrong: the tag has to exist on the cached entry before the mutation can target it. See
  [`caching.md`](caching.md).
- **Revalidate after the write succeeds, not before.** Invalidating first serves a fresh read of
  unchanged data and then goes stale again.
- **Do not revalidate everything.** A blanket invalidation on every mutation makes your cache a
  cold cache, and the next visitor pays for it.
- **Time-based revalidation is a fallback, not a plan.** It bounds staleness for data that
  changes without your app knowing (an external feed). For your own writes, invalidate
  explicitly — a 60-second window is 60 seconds of a user seeing their own edit missing.
- **Revalidation does not reach a browser cache** you granted with a long `max-age`, and it does
  not reach a CDN layer above the framework unless configured to. If a page is still stale after
  a successful revalidate, ask which layer you are actually looking at.
- **After a mutation that invalidates the caller's session, redirect — do not revalidate.**
  Revalidating with a dead session renders a signed-out page with no explanation.
