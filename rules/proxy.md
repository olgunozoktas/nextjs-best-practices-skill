# `proxy.ts` — Middleware's new name, and what it must not do

Sources: [`/getting-started/proxy`](https://nextjs.org/docs/app/getting-started/proxy) ·
[`/guides/authentication`](https://nextjs.org/docs/app/guides/authentication) ·
[`/api-reference/config/next-config-js/redirects`](https://nextjs.org/docs/app/api-reference/config/next-config-js/redirects)
Next.js `16.3.0`, read `2026-08-06`.

## It was renamed in 16

> "Starting with Next.js 16, Middleware is now called Proxy to better reflect its purpose. The
> functionality remains the same."

**Convention:** a single `proxy.ts` (or `.js`) at the project root — or inside `src` if you use
it — at the same level as `app`/`pages`. Only one file is supported per project; split the logic
into modules and import them into it.

If you are reading older material, mentally substitute. The rename is a signal about intent:
it is a *proxy*, not a general-purpose interception layer.

## The three things the docs explicitly rule out

**1. It is not authorization.** Verbatim: *"Proxy is not intended for slow data fetching. While
Proxy can be helpful for optimistic checks such as permission-based redirects, it should not be
used as a full session management or authorization solution."*

So: an **optimistic** redirect for an obviously-unauthenticated visitor is fine and good UX.
The enforcement belongs at the data layer, because every path reaches the data layer and not
every path reaches the proxy. See [`data-security.md`](data-security.md).

**2. It is not for data fetching.** It sits in front of every matched request, so latency there
is latency everywhere.

**3. `fetch` caching does not work there.** Verbatim: *"Using fetch with `options.cache`,
`options.next.revalidate`, or `options.next.tags`, has no effect in Proxy."* A fetch you believe
is cached is running on every request. This is the kind of fact that is invisible until you look
at the invocation count.

## Prefer config to proxy for simple cases

The docs say it directly: for simple redirects, use the `redirects` config in `next.config.ts`
first; reach for proxy when you need request data or more complex logic.

Config redirects are declarative, reviewable, and free of per-request execution. A proxy that
exists only to redirect `/old` to `/new` is a cost with no benefit.

## Practices

- **Match narrowly.** A matcher covering everything runs on static assets, favicons and health
  checks. Write the narrowest matcher that works and re-check it when you add routes.
- **Keep it synchronous-ish and cheap.** No database calls, no chatty upstreams.
- **Beware suppressing the cache.** Varying a response on a cookie or header in the proxy can
  make otherwise-cacheable routes uncacheable. Check the cache behaviour *after* adding proxy
  logic, not before.
- **One file, modules inside.** The docs bless splitting logic into modules and aggregating in
  `proxy.ts` — do that rather than growing one function.
