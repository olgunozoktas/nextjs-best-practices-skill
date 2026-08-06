# Metadata, images, fonts, scripts

Sources: [`/getting-started/metadata-and-og-images`](https://nextjs.org/docs/app/getting-started/metadata-and-og-images) ·
[`/getting-started/images`](https://nextjs.org/docs/app/getting-started/images) ·
[`/getting-started/fonts`](https://nextjs.org/docs/app/getting-started/fonts) ·
[`/guides/scripts`](https://nextjs.org/docs/app/guides/scripts) ·
[`/guides/json-ld`](https://nextjs.org/docs/app/guides/json-ld) ·
[`/guides/third-party-libraries`](https://nextjs.org/docs/app/guides/third-party-libraries)
Next.js `16.3.0`, read `2026-08-06`.

## Metadata

- **Export `metadata` (static) or `generateMetadata` (dynamic)** rather than writing `<head>`
  tags by hand — the framework dedupes and merges across the layout tree.
- **`generateMetadata` does NOT block UI rendering on dynamically rendered pages** — corrected
  2026-08-06, the first version said it delayed the whole document. The docs: *"For dynamically
  rendered pages, Next.js streams metadata separately, injecting it into the HTML once
  `generateMetadata` resolves, **without blocking UI rendering**."* It *does* gate prerendered
  pages and `htmlLimitedBots`. Either way, reuse the page's data function — a `cache()`-wrapped
  read is fetched once for both.
- **Every page needs a distinct title and description.** Inherited-everywhere metadata is a
  duplicate-content problem and reads as unfinished when shared.
- **Set a canonical URL** for anything reachable at more than one path.
- **OG images: generate them, do not hand-maintain them.** The file convention produces one per
  route with the real title. A single static share image for every page is a missed and cheap
  win.
- **JSON-LD** has its own guide; use it for structured data rather than inventing microdata.

## Images

- **`next/image` over `<img>`** — you get sizing, format negotiation and lazy loading.
- **Always set `width`/`height` or `fill`.** Omitting them is the layout-shift bug.
- **`priority` is DEPRECATED in Next 16** — corrected 2026-08-06. *"Starting with Next.js 16,
  the `priority` property has been deprecated in favor of the `preload` property in order to make
  the behavior clear."* And even `preload` is second choice: *"In most cases, you should use
  `loading="eager"` or `fetchPriority="high"` instead of `preload`."*
  So: reach for `loading="eager"` / `fetchPriority="high"` on the LCP image. Do not apply it
  broadly — but note the docs also list *"multiple images that could be considered the LCP element
  depending on the viewport"* under when NOT to use it, so a flat "only one image, ever" is too
  strong for responsive art direction.
- **Set `sizes` when the image is responsive**, or the browser downloads a wrong-sized asset.
- **Image optimisation costs money on most hosts.** Optimising a decorative 20px icon at request
  time is waste; ship it as a correctly sized static asset.

## Fonts

- **`next/font` self-hosts and removes the external round trip** — no layout shift from a late
  webfont, and no third-party request on the critical path.
- **Subset to the characters you need**, and load only the weights you actually use. Four weights
  "for flexibility" is three unnecessary downloads.
- **Never a `<link>` to a font CDN** when `next/font` can host it. That is a render-blocking
  request to somebody else's infrastructure.

## Scripts

- **`next/script` with a deliberate strategy.** `afterInteractive` for most, `lazyOnload` for
  anything non-essential, `beforeInteractive` almost never — it blocks.
- **Third-party scripts are the usual reason a fast app feels slow.** Each one is somebody
  else's code on your critical path with your users' data. Audit the list periodically and ask
  what each is for; the tag manager that was added for one experiment is still there.
- **Load analytics after interactive.** No measurement is worth delaying the thing being
  measured.
