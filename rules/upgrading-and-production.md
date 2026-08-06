# Upgrading, renames, and the pre-launch pass

Sources: [`/guides/upgrading/version-16`](https://nextjs.org/docs/app/guides/upgrading/version-16) ·
[`/guides/upgrading/codemods`](https://nextjs.org/docs/app/guides/upgrading/codemods) ·
[`/guides/production-checklist`](https://nextjs.org/docs/app/guides/production-checklist) ·
[`/guides/ai-agents`](https://nextjs.org/docs/app/guides/ai-agents) ·
[`/guides/self-hosting`](https://nextjs.org/docs/app/guides/self-hosting)
Next.js `16.3.0`, read `2026-08-06`.

## Point your agent docs at version-matched docs — the docs say so

The Next.js 16 upgrade guide's own recommended prompt begins by telling the agent to make sure
`AGENTS.md` points at **version-matched** Next.js docs, and to fix it if missing or outdated.

That is the highest-leverage thing in this file. **Most bad Next.js advice is correct advice for
a different version.** The App Router's caching defaults, the dynamic APIs' sync/async shape, and
Middleware's very name have all changed across recent majors — so an answer without a version is
not an answer.

- Pin the docs your agent reads to the version in `package.json`.
- When you state a fact, state the version with it.
- Prefer the **codemods** to hand-editing for mechanical migrations; they are maintained and
  they do not miss files.

## Renames to be aware of (16)

| Was | Now |
|---|---|
| `middleware.ts` | **`proxy.ts`** — see [`proxy.md`](proxy.md) |

Functionality unchanged; the rename signals intent. Older tutorials and answers will use the old
name indefinitely.

## Upgrading, in an order that keeps you sane

1. **Read the upgrade guide as the source of truth** — not a blog post, not memory.
2. **Run the codemod** for the mechanical parts.
3. **Inspect the diff.** Codemods are good, not omniscient.
4. **Typecheck, then build.** A typecheck-clean app can still fail to build — prerendering
   errors only exist at build time, and that is exactly the class a major bump introduces.
5. **Exercise the interactive states**, not just the routes. Check the dev indicator plus browser
   and server logs.

**The build is a gate, not an afterthought.** A change can be type-correct and still break static
generation; if your CI gate omits `next build`, a whole defect class ships green.

## Pre-launch

Work Next's own production checklist rather than a remembered list — it is maintained. The pass
worth doing on top of it:

- **Which routes are static and which are dynamic, and did you choose that?** Compare the build
  output against intent. An accidentally dynamic route is the most common and most expensive
  surprise.
- **Does every cacheable route actually cache?** Check the behaviour, not the intent.
- **Is authorization enforced at the data layer** on every path, not in a layout? See
  [`data-security.md`](data-security.md).
- **Does every `'use server'` export authorize from the session?** See
  [`server-actions.md`](server-actions.md).
- **Is there a `global-error`?** A throwing root layout otherwise renders nothing.
- **CSP in report-only first**, then enforce. Added late, a CSP breaks a live page.
- **Self-hosting?** Read the self-hosting guide for the behaviours a platform normally provides —
  image optimisation, ISR storage and caching are yours to run.

For platform-side configuration — function limits, CDN, firewall, spend limits — use the
`vercel-platform-practices` skill. For component performance, `react-best-practices`.
