# nextjs-best-practices

A Claude Code / [Agent Skills](https://agentskills.io/) skill for **Next.js App Router
correctness, security and architecture** — the server/client boundary, Server Action security,
the Data Access Layer pattern, Cache Components, revalidation, `proxy.ts`, error boundaries,
async dynamic APIs, forms, metadata, and upgrading.

```sh
git clone https://github.com/olgunozoktas/nextjs-best-practices-skill.git \
  ~/Herd/nextjs-best-practices-skill
ln -s ~/Herd/nextjs-best-practices-skill ~/.claude/skills/nextjs-best-practices
```

Refresh with `git pull`. Nothing to build, nothing to run, works offline.

## Provenance and version pinning

Facts trace to `nextjs.org/docs`, cited by URL in the rule file that uses them. They were read
out of the official documentation corpus — `nextjs.org/docs/llms-full.txt`, **449 documents,
`version: 16.3.0`, `@router: App Router`** — rather than recalled.

**Version pinning is not a formality for this framework.** Across recent majors, the caching
defaults changed, the dynamic APIs became async, and **Middleware was renamed to Proxy**. Most
confidently-wrong Next.js advice is advice that was correct two versions ago. The frontmatter
records `nextjs_version: 16.3.0`; the upgrade rule tells you to point your agent docs at
version-matched docs, because [the Next.js upgrade guide itself says to](https://nextjs.org/docs/app/guides/upgrading/version-16).

## Why this exists alongside `react-best-practices`

Install [`vercel-labs/agent-skills`](https://github.com/vercel-labs/agent-skills) too. Its
`react-best-practices` is larger and written by the people who build the framework — and it is
about a **different axis**.

Measured on the upstream repo at `7c180d9`, `react-best-practices` is 76 files / ~28k words, and
its rule filenames are: `js-*` (micro-optimisation — `js-set-map-lookups`, `js-hoist-regexp`,
`js-early-exit`), `bundle-*`, `rerender-*`, `rendering-*`, `client-*`, `async-*`, `advanced-*`.

It is a **performance** skill. There is no rule about Server Action authorization, the Data
Access Layer, DTOs across the boundary, `'use cache'` purity, `revalidateTag` strategy, the
Middleware→Proxy rename, or what an error boundary leaks — because those are not performance
questions.

| | `react-best-practices` (official) | `nextjs-best-practices` (this) |
|---|---|---|
| Axis | **Is it fast?** | **Is it correct, and can a user reach data they shouldn't?** |
| Size | 76 files, ~28k words | 13 files, ~6.8k words |
| Covers | Waterfalls, bundle size, re-renders, hydration perf, JS micro-optimisation | Boundary as a trust boundary, Server Action security, DAL/DTOs, cache purity, revalidation, proxy limits, error leakage, dynamic-API placement |
| Overlap | Hydration and Suspense, from opposite directions | |

**Use both.** They collide almost nowhere, and the overlap that exists is complementary: their
hydration rules are about flicker and cost; ours is about the server and client computing the
same string.

### Where this skill is worse

- **Smaller, and deliberately narrower.** A quarter the size.
- **No performance guidance.** Deferred entirely, with a routing table at the top of `SKILL.md`.
- **A snapshot, not a live feed.** Pinned to 16.3.0; a framework this fast-moving will outrun it.
  The rules carry source URLs so any number can be re-checked.
- **Opinionated where the docs offer options.** It tells you to pick the Data Access Layer for
  new projects and to enforce authorization at the data layer. That is the documented
  recommendation, but it is a stance.

## The three highest-value things in it

1. **Every export of a `'use server'` module is a public HTTP endpoint.** Authorize inside the
   action, from the session — never from an argument, and never because the UI hid the button.
2. **A cached value must be a function of its cache key** — not the clock, not the request, not
   the reader. Nothing errors when you get this wrong; a stale-but-plausible value is served for
   the whole window.
3. **Proxy is not authorization.** The docs say it "should not be used as a full session
   management or authorization solution", and that `fetch` cache options have no effect there.

## Companion skills

| Question | Skill |
|---|---|
| Is this fast? | `react-best-practices` (official) |
| Is this correct/secure? | **this skill** |
| Platform config, functions, CDN, firewall, spend | [`vercel-platform-practices`](https://github.com/olgunozoktas/vercel-platform-practices-skill) |
| Deploy it | `deploy-to-vercel` (official) |
| Cost/latency audit of a deployment | `vercel-optimize` (official) |
| UI/accessibility review | `web-design-guidelines` (official) |

## Licence

MIT. The documentation it cites is Vercel's/Next.js'; the rules, structure and judgements are the
author's.
