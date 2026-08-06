# Dynamic APIs — async, and each one makes a route dynamic

Sources: [`/api-reference/functions/cookies`](https://nextjs.org/docs/app/api-reference/functions/cookies) ·
[`/api-reference/functions/headers`](https://nextjs.org/docs/app/api-reference/functions/headers) ·
[`/api-reference/file-conventions/page`](https://nextjs.org/docs/app/api-reference/file-conventions/page) ·
[`/guides/upgrading/version-15`](https://nextjs.org/docs/app/guides/upgrading/version-15)
Next.js `16.3.0`, read `2026-08-06`.

## They are async — await them

`cookies()`, `headers()`, `draftMode()`, and a page's `params` and `searchParams` are
Promise-based (since 15). Await them:

```tsx
export default async function Page({ params, searchParams }: {
  params: Promise<{ id: string }>
  searchParams: Promise<{ q?: string }>
}) {
  const { id } = await params
  const { q } = await searchParams
}
```

Forgetting the `await` produces a Promise where you expected a value — which stringifies to
`[object Promise]` rather than throwing, so it shows up as odd output rather than an error.
There are codemods for the migration; use them rather than hand-editing.

## Reading one makes the route dynamic — and that is the real subject

**With `cacheComponents: true`**, *where* you read matters as much as whether:

- **At the top of a page** → the whole route is dynamic.
- **Inside a `<Suspense>` boundary** → the shell stays static and only the boundary is dynamic.

**Without Cache Components — the default — any of these reads opts the WHOLE route into dynamic
rendering**, wherever you put it: *"Without Cache Components, reading `cookies()`, `headers()`, or
`searchParams` opts the whole route into dynamic rendering."*

**Corrected 2026-08-06**: the first version stated the boundary fix unconditionally and called it
"the answer to most 'why is my page dynamic?' questions". It is the answer only on Cache
Components. Check `next.config.ts` first. See [`caching.md`](caching.md).

Watch for the cheap-but-dynamic read: a locale, a theme cookie, the current year. Individually
trivial, and each one at the top of a layout makes every page under it dynamic.

## Practices

- **Read once, pass down.** Several components each calling `cookies()` is fine for correctness
  but obscures the boundary. A `cache()`-wrapped accessor gives one read and one place to change
  it — and the docs recommend that shape for the session specifically, on security grounds.
- **Validate `searchParams`.** They are user input: arbitrary strings, arrays where you expected
  one value, absent where you expected presence. Parse, do not cast.
- **`searchParams` is the right home for state you want shareable** — filters, tabs, pagination.
  Putting it in client state instead makes the view unlinkable and unrenderable on the server.
- **Do not use a dynamic read as a cache-buster.** If you need per-request output, say so with
  the boundary; making a route accidentally dynamic to get freshness is a decision nobody can
  find later.
