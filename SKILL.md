---
name: nextjs-best-practices
description: "Apply this skill whenever writing, reviewing, or refactoring Next.js App Router code. Triggers on Server and Client Component boundaries, `'use client'` placement, Server Actions and their security, the Data Access Layer pattern and DTOs, Cache Components (`'use cache'`, `cacheLife`, `cacheTag`), revalidation (`revalidatePath`/`revalidateTag`), Route Handlers, `proxy.ts` (Middleware's Next 16 name), error boundaries and what they leak, async dynamic APIs (`cookies`, `headers`, `params`, `searchParams`), forms and `useActionState`, metadata and OG images, and Next 16 upgrade breakage. Also use when diagnosing a route that went dynamic unexpectedly, a Server Action reachable without authorization, a stale page after a mutation, or a hydration mismatch."
license: MIT
metadata:
  author: olgunozoktas
  version: "1.0.0"
  nextjs_version: "16.3.0"
  docs_snapshot: "2026-08-06"
---

# Next.js Best Practices (App Router)

Correctness, security and architecture practices for the Next.js **App Router**, organised as
an index of rule files. Each rule teaches what to do and why.

**Grounded and version-pinned.** Every factual claim traces to `nextjs.org/docs`, cited by URL
in the rule file. The facts were read out of the official documentation corpus
(`nextjs.org/docs/llms-full.txt` — 449 documents, `version: 16.3.0`, `@router: App Router`)
rather than recalled. **Next.js moves fast and renames things** — Middleware became Proxy in
16 — so re-verify anything load-bearing against the live doc for *your* installed version.

## Read this first: this is not a performance skill

For component-level performance there is a larger, better, official skill and you should use
it instead. Route by question:

| Question | Use |
|---|---|
| Is this render/fetch/bundle fast? Waterfalls, memoisation, re-renders, dynamic imports, JS micro-optimisation | **`react-best-practices`** (official, Vercel Engineering, ~28k words, ~70 rules) |
| **Is this correct, secure, and architecturally right for the App Router?** | **this skill** |
| How should this be configured/cached/secured on the Vercel *platform*? | **`vercel-platform-practices`** |
| Deploy it | **`deploy-to-vercel`** (official) |
| Cost/latency audit of a deployed project | **`vercel-optimize`** (official) |

The split in one line: `react-best-practices` asks *"is this fast?"*; this skill asks *"is this
right, and can a user reach data they shouldn't?"* They overlap almost nowhere.

## Consistency First

Before applying any rule, check what the application already does. The App Router supports
several valid architectures — HTTP APIs, a Data Access Layer, component-level access — and the
docs are explicit that you should **choose one and not mix them**, because mixing makes the
codebase unauditable for both developers and security reviewers.

So: find the established pattern and follow it. These rules are defaults for where no pattern
exists, not licence to introduce a second one. If the project reads data through a DAL, a new
component that queries directly is a defect even if it works.

## How to Apply

1. **Establish which side of the server/client boundary you are on.** Most App Router bugs are
   boundary bugs, and the answer changes every other decision.
2. **Map the change to the rule index** and read the mapped files before editing. Cross-cutting
   changes need more than one.
3. **Check the project's existing architecture** — is there a DAL? Are mutations Server Actions
   or Route Handlers? Follow it.
4. **For anything version-sensitive, verify against the installed version's docs.** This skill
   is pinned to 16.3.0. Point your agent docs at version-matched Next.js docs (the upgrade
   guide describes an `AGENTS.md` setup for exactly this).
5. **Ask the security question explicitly on every data path**: who is allowed to see this, and
   where is that enforced? "The UI doesn't render the button" is not an answer.
6. **Re-read the diff against every mapped rule**, and run typecheck — the props a Client
   Component receives are the cheapest guard the boundary has.

## Rule Index

| Concern | Read |
| --- | --- |
| The server/client boundary, `'use client'`, serialisation, composition | [`rules/server-and-client-components.md`](rules/server-and-client-components.md) |
| Who can read what, Data Access Layer, DTOs, authorization placement | [`rules/data-security.md`](rules/data-security.md) |
| Server Actions: every export is a public endpoint | [`rules/server-actions.md`](rules/server-actions.md) |
| `'use cache'`, `cacheLife`, `cacheTag`, what may not go in a cached scope | [`rules/caching.md`](rules/caching.md) |
| `revalidatePath`, `revalidateTag`, ISR, staleness after a mutation | [`rules/revalidation.md`](rules/revalidation.md) |
| Route Handlers, and when they beat a Server Action | [`rules/route-handlers.md`](rules/route-handlers.md) |
| `proxy.ts` (Middleware renamed in 16) — and what it must not do | [`rules/proxy.md`](rules/proxy.md) |
| `error.tsx`, `global-error`, `not-found`, and what an error leaks | [`rules/error-handling.md`](rules/error-handling.md) |
| Async `cookies`/`headers`/`params`/`searchParams`, static vs dynamic | [`rules/dynamic-apis.md`](rules/dynamic-apis.md) |
| Forms, `useActionState`, progressive enhancement, validation | [`rules/forms.md`](rules/forms.md) |
| Metadata, OG images, images, fonts, scripts | [`rules/metadata-and-assets.md`](rules/metadata-and-assets.md) |
| Upgrading, renames, agent docs pinning, pre-launch checks | [`rules/upgrading-and-production.md`](rules/upgrading-and-production.md) |

## Decision Rules

- **The boundary is a trust boundary, not just a bundling one.** Anything passed from a Server
  Component to a Client Component is shipped to the browser and readable. Pass DTOs, never
  whole rows.
- **Every export of a `'use server'` module is a public HTTP endpoint.** Authorize inside the
  action, from the session — never from an argument, and never on the strength of the UI having
  hidden the control.
- **A cached value must be a function of its cache key.** Not the clock, not the request, not
  the reader. See [`rules/caching.md`](rules/caching.md).
- **Choose one data-fetching architecture and keep it.** The docs say so explicitly, and the
  reason is auditability rather than taste.
- **Proxy is not authorization.** The docs say it "should not be used as a full session
  management or authorization solution". Optimistic redirect there; enforce in the DAL.
- **Prefer the framework mechanism over a hand-rolled one** — `redirects` in config over a
  proxy redirect, `revalidateTag` over a cache-busting query param, `useActionState` over a
  bespoke fetch.
- **State the version you relied on.** Next.js renames and re-defaults across majors; "as of
  16.3" is checkable, "Next.js does X" is not.
