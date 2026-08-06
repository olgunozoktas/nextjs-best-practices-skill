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

Facts trace to `nextjs.org/docs`, cited by URL in the rule file that uses them, from
`nextjs.org/docs/llms-full.txt` — **449 documents (446 stamped `version: 16.3.0`),
`@router: App Router`**.

> **An honest correction to this claim, 2026-08-06.** The first version said facts were "read out
> of the corpus rather than recalled". **That was not uniformly true**, and an adversarial review
> proved it: the security, DAL and Proxy material *was* extracted from the corpus and is
> verbatim-accurate, but the caching, revalidation and asset rules were written from prior
> Next-15-era knowledge and given 16.3 citations afterwards. That produced five real errors — a
> deprecated `revalidateTag` signature, a missing `updateTag`, a deprecated `priority` prop,
> `draftMode()` wrongly listed as forbidden inside `'use cache'`, and Cache-Components-only
> behaviour presented as the App Router default.
>
> All five are fixed and each carries an inline correction note. The lesson is the one this skill
> teaches in `caching.md`: **a citation is not provenance.** Verify load-bearing numbers against
> the live doc for your installed version.

**Version pinning is not a formality for this framework.** Across recent majors, the caching
defaults changed, the dynamic APIs became async, and **Middleware was renamed to Proxy**. Most
confidently-wrong Next.js advice is advice that was correct two versions ago. The frontmatter
records `nextjs_version: 16.3.0`; the upgrade rule tells you to point your agent docs at
version-matched docs, because [the Next.js upgrade guide itself says to](https://nextjs.org/docs/app/guides/upgrading/version-16).

## Why this exists alongside `react-best-practices`

Install [`vercel-labs/agent-skills`](https://github.com/vercel-labs/agent-skills) too. Its
`react-best-practices` is larger and written by the people who build the framework — and it is
about a **different axis**.

> **This section was rewritten 2026-08-06 after an adversarial review refuted its central
> claim.** The original said `react-best-practices` had "no rule about Server Action
> authorization". **That is false**, and it was false in the way that most flattered this skill.
> The original also enumerated seven rule prefixes while omitting `server-*` — the one category
> containing the rule that refutes it. Both are corrected below, and the honest overlap is larger
> than I first wrote.

Measured on the upstream repo at `7c180d9`, `react-best-practices` has **70 rules** (its own
`SKILL.md` says so) across 8 prefixes: `rerender-` 15, `js-` 14, `rendering-` 11, **`server-` 10**,
`bundle-` 6, `async-` 6, `client-` 4, `advanced-` 4.

**It is mostly a performance skill, but not exclusively — and it has two security rules:**

- **`server-auth-actions.md`** — "Authenticate Server Actions Like API Routes",
  `impact: CRITICAL`, `tags: … security, authorization`. It says Server Actions "are exposed as
  public endpoints, just like API routes", tells you to verify auth **inside** each action, warns
  against relying on middleware or layout guards, and ships a zod-validated example. **This is
  nearly identical to what was previously listed here as this skill's #1 differentiator.**
- **`server-no-shared-module-state.md`** — cross-request contamination, "security bugs where one
  user's data appears in another user's response". A tenancy-leak rule, not a performance rule.
- **`server-serialization.md`** — "Only pass fields that the client actually uses", with a
  50-field `user` prop narrowed to `name`. The token "DTO" never appears, and the motive is page
  weight rather than leakage, but the *mechanic* is the same one this skill teaches.

**What genuinely is absent from it:** the Data Access Layer pattern (0 hits), error-boundary
leakage (0 hits), the Middleware→Proxy rename (the string `proxy` appears once, inside an
unrelated filename), `'use cache'` purity, `cacheTag`/`cacheLife`, and `revalidateTag`/`updateTag`
strategy.

**And two caveats on the cache claims**, because `react-best-practices` is not the only official
skill: `vercel-optimize` covers `'use cache'` purity head-on — it ships a **static scanner**
(`lib/scanners/use-cache-date-stamp.mjs`) for `new Date()` / `Date.now()` / `Math.random()` inside
a `'use cache'` file, plus `revalidateTag` gates. So on that topic the official set is ahead of
this skill, with automated detection this skill does not have. The differentiator is the *reason*
(correctness for the reader vs ISR write amplification), not the topic.

| | `react-best-practices` (official) | `nextjs-best-practices` (this) |
|---|---|---|
| Axis | Mostly **is it fast?** | **Is it correct, and can a user reach data they shouldn't?** |
| Size (same basis: SKILL + rules) | **70 rules, ~14.6k words** | 13 files, ~6.8k words — so **~2.1x**, not 4x |
| Covers | Waterfalls, bundle size, re-renders, JS micro-optimisation — **plus** Server Action auth and cross-request state leakage | DAL/DTOs as a security boundary, cache purity, revalidation APIs, proxy limits, error leakage, dynamic-API placement |
| **Real overlap** | **Server Action authorization — a head-on collision, we agree.** Prop narrowing, from different motives. Cross-request state | |

**Use both.** But the honest framing is *"we agree on Server Action authorization and arrive at
prop-narrowing from different motives; we differ on the DAL, cache purity as a correctness
concern, the revalidation APIs, proxy limits, and error leakage"* — not "they collide nowhere".

A **size correction** while I am here: the "~28k words" figure counted the corpus twice, because
`AGENTS.md` is a re-expansion of the same 70 rules. Measured on the same basis as this skill,
the gap is 2.1x. My original "a quarter the size" was over-humble in a way that conveniently
inflated the "it is vast, therefore mine must be a different axis" premise.

### Where this skill is worse

- **Smaller, and deliberately narrower** — about half the size on a like-for-like basis (see the correction above).
- **No performance guidance.** Deferred entirely, with a routing table at the top of `SKILL.md`.
- **A snapshot, not a live feed.** Pinned to 16.3.0; a framework this fast-moving will outrun it.
  The rules carry source URLs so any number can be re-checked.
- **Opinionated where the docs offer options.** It tells you to pick the Data Access Layer for
  new projects and to enforce authorization at the data layer. That is the documented
  recommendation, but it is a stance.

## What is actually distinctive here

Revised down from "the three highest-value things" once two of the three turned out to be covered
elsewhere:

1. **Proxy is not authorization**, and `fetch` cache options have **no effect** in it — both
   verbatim from the docs. The Middleware→Proxy rename appears nowhere in the official set.
   *This one is genuinely unique.*
2. **The Data Access Layer as a security boundary** — authorize next to the query, return DTOs,
   `cache()` the session read for the documented security reason. 0 hits for "data access" in the
   official set.
3. **The revalidation APIs, correctly** — `updateTag` for read-your-own-writes, the two-argument
   `revalidateTag`, `refresh()`. Not in `react-best-practices`; `vercel-optimize` gates
   `revalidateTag` from a cost angle but does not teach the choice.
4. **What an error boundary leaks**, and `global-error` as a hard requirement. 0 hits for "error
   boundary" officially.

**Not distinctive, contrary to what this README first claimed:** Server Action authorization
(`server-auth-actions.md`, CRITICAL) and `'use cache'` purity (`vercel-optimize` has a scanner).

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
