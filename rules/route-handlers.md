# Route Handlers — and when they beat a Server Action

Sources: [`/getting-started/route-handlers`](https://nextjs.org/docs/app/getting-started/route-handlers) ·
[`/guides/backend-for-frontend`](https://nextjs.org/docs/app/guides/backend-for-frontend)
Next.js `16.3.0`, read `2026-08-06`.

## Choose by who is calling

| Caller | Use |
|---|---|
| **Your own UI**, in the app | **Server Action** (mutations) or a Server Component read |
| **Anything that is not your UI** — webhooks, third-party API clients, mobile apps, CLIs, cron | **Route Handler** |

That single question resolves most of the ambiguity. A Server Action is an internal RPC with a
framework-specific wire format; a Route Handler is an HTTP endpoint with a contract you can
document and version.

**Do not have your own UI fetch your own Route Handler for data a Server Component could read
directly.** It adds a network hop, loses the request's own context, and duplicates the
authorization. The exception worth making deliberately: a genuinely client-driven surface where
you *want* to exercise the same public API your customers integrate against — and if you do it
for that reason, write the reason down.

## Practices

- **Validate the request** — body, query, headers — with a schema. Everything arriving is
  untrusted, and TypeScript is erased.
- **Authorize explicitly.** A Route Handler has no ambient session guarantee; resolve identity
  and check it. A handler that works because it happens to be behind a proxy redirect is not
  authorized.
- **Return a documented error shape.** Pick one (RFC 9457 problem+json is a good default) and
  keep it, so a client can branch on `kind` rather than parsing prose.
- **Do not enumerate.** A 404 for "not yours" and a 404 for "does not exist" should be
  indistinguishable, or the endpoint is an existence oracle.
- **Verify webhook signatures, then re-fetch the object from the source.** A payload can be
  replayed, delayed, or reordered; the signature proves origin, not currency.
- **Make webhook handling idempotent** by recording the provider's event id and skipping what
  you have seen. Providers retry for days — duplicate delivery is documented behaviour, not an
  edge case.
- **`GET` caching has defaults that changed across versions.** Check the behaviour for your
  installed version rather than assuming, and be explicit when it matters.
