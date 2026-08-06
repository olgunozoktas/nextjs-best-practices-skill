# Revalidation — keeping surfaces honest after a write

Sources: [`/getting-started/revalidating`](https://nextjs.org/docs/app/getting-started/revalidating) ·
[`/guides/how-revalidation-works`](https://nextjs.org/docs/app/guides/how-revalidation-works) ·
[`revalidateTag`](https://nextjs.org/docs/app/api-reference/functions/revalidateTag) ·
[`updateTag`](https://nextjs.org/docs/app/api-reference/functions/updateTag) ·
[`refresh`](https://nextjs.org/docs/app/api-reference/functions/refresh) ·
[`/guides/upgrading/version-16`](https://nextjs.org/docs/app/guides/upgrading/version-16)
Next.js `16.3.0`, read `2026-08-06`.

> **This file was rewritten 2026-08-06 after an adversarial review.** The first version
> recommended `revalidateTag(tag)` — a signature **deprecated in Next 16 that produces a
> TypeScript error** — and never mentioned `updateTag`, which is the documented answer to the
> very problem the file opens with. Both errors are recorded here rather than quietly fixed,
> because they are an instance of the exact trap this skill warns about: advice that was correct
> for Next 15, given a 16.3 citation.

## The default failure is a stale screen after a successful write

The mutation worked and the list still shows the old data. Users read that as "it didn't save"
and do it again — which is how a working feature produces duplicates.

**Next 16 has a specific API for this, and it is not `revalidateTag`.**

## Pick the right one of three

| API | Behaviour | Use when |
|---|---|---|
| **`updateTag(tag)`** — ONE argument | **Immediately expires** the cached data | **Read-your-own-writes** — the user must see their own change now. **Server Actions only** |
| **`revalidateTag(tag, profile)`** | **Stale-while-revalidate** | Background refresh, where a slightly stale read is acceptable |
| **`refresh()`** | Refetches the current route's RSC payload **without invalidating cached data** | The view depends on state outside the cache that the action just changed |

From the docs, verbatim:

> `updateTag` immediately expires cached data for read-your-own-writes scenarios — the user sees
> their change right away instead of stale content. Unlike `revalidateTag`, it can only be used
> in Server Actions.

**This is the correction that matters most in this file.** After a user's own mutation, reach for
`updateTag`. Using `revalidateTag` there is stale-while-revalidate, which means the user's next
read can still be the old value — reproducing the exact bug you were trying to fix.

## `revalidateTag` takes two arguments. `updateTag` takes ONE.

```ts
revalidateTag(tag: string, profile: string | { expire?: number }): void
updateTag(tag: string): void          // ONE argument. There is no profile.
```

> **Corrected again 2026-08-06 — this was my error, twice over.** The first version of this file
> recommended the deprecated single-argument `revalidateTag`. The *fix* for that then claimed
> "both tag APIs take TWO arguments" and printed `updateTag(tag, profile)` in the table above.
> **That signature does not exist**: the docs give `updateTag(tag: string): void`, and all 17
> `updateTag(` call sites in the corpus are single-argument. So the correction failed the very
> typecheck it cited as its clincher — `Expected 1 arguments, but got 2. ts(2554)`.
> The profile argument belongs to `revalidateTag` only. `refresh()` is likewise Server-Actions-only,
> which the table's wording implied was unique to `updateTag`.

> The single-argument form `revalidateTag(tag)` is deprecated. It currently works if TypeScript
> errors are suppressed, but this behavior may be removed in a future version. Update to the
> two-argument signature.

The upgrade guide is blunter: it "now requires a second argument specifying a `cacheLife`
profile. The single-argument form is deprecated and will produce a TypeScript error."

So `revalidateTag('posts')` **fails the typecheck this skill tells you to run.** Pass a profile:
a named `cacheLife` profile, or `{ expire: n }`.

## Tag before you can invalidate

A tag must be on the cached data first, by one of two documented routes:

```tsx
// 1. fetch, for external requests
fetch(url, { next: { tags: ['posts'] } })

// 2. cacheTag, inside a 'use cache' scope
async function getData() {
  'use cache'
  cacheTag('posts')
}
```

**This ordering is what people get wrong**: the tag has to exist on the entry before a mutation
can target it. Tag by entity and id (`project:123`), not by page — `project:123` survives a
routing change and `/projects/123` does not.

## `revalidatePath` still exists, and tags still scale better

`revalidatePath('/projects')` is right when you know the URLs and there are few. Once the same
entity appears on more than one route, prefer tags: adding a sixth route that shows a project
otherwise means finding every mutation and adding a sixth `revalidatePath`, and the one you miss
is the stale one.

## Practices

- **Revalidate after the write succeeds**, not before.
- **Do not invalidate everything on every mutation.** That converts your cache into a cold cache
  and the next visitor pays for it.
- **Time-based revalidation bounds staleness for data that changes without your app knowing** (an
  external feed). For your own writes, invalidate explicitly — a 60-second window is 60 seconds
  of a user not seeing their own edit.
- **Revalidation does not reach a browser cache** you granted with a long `max-age`, and it does
  not reach a CDN layer above the framework unless configured to. If a page is still stale after
  a successful call, ask which layer you are looking at.
- **After a mutation that invalidates the caller's own session, redirect — do not revalidate.**
  Revalidating with a dead session renders a signed-out page with no explanation.
