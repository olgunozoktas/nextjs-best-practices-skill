---
name: nextjs-best-practices
description: "Apply this skill whenever writing, reviewing, or refactoring Next.js App Router code. Triggers on Server and Client Component boundaries, `'use client'` placement, Server Actions and their security, the Data Access Layer pattern and DTOs, Cache Components (`'use cache'`, `cacheLife`, `cacheTag`), revalidation (`updateTag` vs `revalidateTag` vs `refresh`), Route Handlers, `proxy.ts` (Middleware's Next 16 name), error boundaries and what they leak, async dynamic APIs (`cookies`, `headers`, `params`, `searchParams`), forms and `useActionState`, metadata and OG images, and Next 16 upgrade breakage. Also use when diagnosing a route that went dynamic unexpectedly, a Server Action reachable without authorization, a stale page after a mutation, or a hydration mismatch."
license: MIT
metadata:
  author: olgunozoktas
  version: "1.0.0"
  nextjs_version: "16.3.0"
  docs_snapshot: "2026-08-06"
---

# Next.js Best Practices (App Router)

> ## ⚠️ Verification status — read before trusting an API signature
>
> This skill has been through three adversarial review rounds. **Each round found errors, and two
> of the rounds found errors inside the previous round's fixes** — including a fabricated
> `updateTag(tag, profile)` signature that does not compile, and `reset()` recommended where 16.3
> says `retry()`.
>
> The pattern is consistent enough to state plainly: **this skill is reliable where it quotes and
> unreliable where it summarises.** Its prose is more confident than its verification.
>
> | Trust | Areas |
> |---|---|
> | **Verified, quote-backed, genuinely unmatched officially** | `proxy.md` (the Middleware→Proxy rename, `fetch`-cache-inert, not-for-auth) · `data-security.md` (three architectures, DAL, DTOs, `cache()` for security) · the cache-key-invariance rule · Consistency First |
> | **Verify against the live doc before acting** | **any API signature**, any version-sensitive default, `revalidation.md`'s table, error-boundary props, Route Handler `GET` caching, `params`-as-dynamic |
>
> Two method caveats that produced the original errors and are not fully fixed:
> **(a)** the corpus is `nextjs.org/docs/llms-full.txt`, of which **~36% is Pages Router** docs
> (159 of 440 unique URLs) — a grep hit does not tell you which router it came from;
> **(b)** several load-bearing quotes came from API-reference pages the rule files do not cite.
>
> **For Server Action authorization and `'use cache'` purity, prefer the official skills** —
> `react-best-practices`' `server-auth-actions.md` (`impact: CRITICAL`) and `vercel-optimize`'s
> static scanner. See the routing table below.

Correctness, security and architecture practices for the Next.js **App Router**, organised as
an index of rule files. Each rule teaches what to do and why.

**Version-pinned, and see the status block above on provenance.** Claims trace to
`nextjs.org/docs` URLs cited in each rule file, from `nextjs.org/docs/llms-full.txt` — 449 title
blocks, 440 unique URLs, 446 stamped `version: 16.3.0`. The corpus header says
`@router: App Router` but qualifies it, and **159 of those URLs are `/docs/pages/…`** — so the
corpus is not App-Router-only and a grep hit does not identify its router. **Next.js moves fast and renames things** — Middleware became Proxy in
16 — so re-verify anything load-bearing against the live doc for *your* installed version.

## Read this first: this is not a performance skill

For component-level performance there is a larger, better, official skill and you should use
it instead. Route by question:

| Question | Use |
|---|---|
| Is this render/fetch/bundle fast? Waterfalls, memoisation, re-renders, dynamic imports, JS micro-optimisation | **`react-best-practices`** (official, Vercel Engineering, **70 rules / ~14.6k words**) |
| **Server Action authorization** | **`react-best-practices`'s `server-auth-actions.md`** (official, `impact: CRITICAL`) *and* this skill's [`rules/server-actions.md`](rules/server-actions.md) — **they agree; read the official one too** |
| **`'use cache'` purity (clock/random inside a cached scope)** | **`vercel-optimize`** has a *static scanner* for this (`lib/scanners/use-cache-date-stamp.mjs`) — **better than this skill, which only teaches it.** [`rules/caching.md`](rules/caching.md) for the reasoning |
| Authorize-next-to-the-query | Official `vercel-optimize`'s `auth-preserving-parallelization.md` covers the core move; this skill's [`rules/data-security.md`](rules/data-security.md) covers the fuller DAL/DTO pattern |
| **Is this correct, secure, and architecturally right for the App Router?** | **this skill** |
| How should this be configured/cached/secured on the Vercel *platform*? | **`vercel-platform-practices`** |
| Deploy it | **`deploy-to-vercel`** (official) |
| Cost/latency audit of a deployed project | **`vercel-optimize`** (official) |

The split in one line: `react-best-practices` is mostly *"is this fast?"*; this skill is *"is this
right, and can a user reach data they shouldn't?"*

**They do NOT overlap nowhere — corrected 2026-08-06.** An adversarial review found three places
they meet, and in two of them the official set is equal or better:

- **Server Action authorization** — `server-auth-actions.md` is `impact: CRITICAL` and says the
  same thing this skill says. A head-on agreement, not a gap.
- **`'use cache'` purity** — `vercel-optimize` ships a static scanner; this skill only explains
  the rule. **Use theirs to detect, this one to understand.**
- **Prop narrowing** — `server-serialization.md` teaches "only pass fields the client actually
  uses" from a page-weight motive; this skill teaches the same mechanic as a leakage boundary.

What is genuinely absent from the official set: the Middleware→**Proxy** rename, error-boundary
leakage, the revalidation API choice (`updateTag` vs `revalidateTag`), and the full DAL/DTO
pattern. Those are this skill's real contribution.

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
