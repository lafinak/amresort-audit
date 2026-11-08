# amresort.sk — quick web audit

Informal audit, not a client deliverable. Ran Lighthouse (mobile + desktop) and axe-core
against the live site on 2026-11-08, plus some manual checks (headers, robots.txt,
render timing). Raw tool output is in `audit-data/`.

## TL;DR

The site is **fine on desktop, bad on mobile/slow connections** — and it's one root cause,
not ten unrelated problems: too many third-party tracking scripts loaded the wrong way in
`<head>`. Fixing that one thing should fix most of the performance score. Everything else
below is smaller and optional.

| Area | Desktop | Mobile |
|---|---|---|
| Performance | 79/100 | **50/100** |
| Accessibility | 77/100 | 73/100 |
| Best Practices | 96/100 | 100/100 |
| SEO | 83/100 | 83/100 |

| Metric | Desktop | Mobile |
|---|---|---|
| First Contentful Paint | 1.4s | 2.3s |
| Largest Contentful Paint | 1.8s | **39.1s** |
| Time to Interactive | 2.5s | **39.2s** |
| Total Blocking Time | 10ms | 660ms |

## Finding #1 — Render-blocking third-party scripts (the big one)

**Severity: High. This is almost certainly what you felt as "slow."**

There are **8 blocking `<script>` tags in `<head>`** (no `async`/`defer`), pulling from
6 different third-party domains: Google Tag Manager, Google Analytics, Facebook Pixel,
Microsoft Clarity (session recording), CookieYes (cookie banner — 9 requests on its own),
and Google Fonts. The browser downloads and runs each one in order before it can keep
parsing the page, so they queue up like a relay race instead of running in parallel.

In a real (non-simulated) browser session, the page took **18 seconds just to reach
DOMContentLoaded** — and the total page weight is only ~800KB, so it's not a "too much
data" problem, it's purely this serial script chain. Under Lighthouse's mobile network
throttle it compounds to a 39-second Time to Interactive.

**Fix (cheap, no redesign needed):**
- Add `defer` to every `<script src="...">` tag in `<head>` that isn't needed before paint.
- Since Google Tag Manager is already installed, fire Facebook Pixel, Clarity, and GA
  *through* GTM as tags instead of hardcoding separate `<script>` includes — GTM loads
  itself async and dispatches the rest without blocking the page.
- Move the CookieYes banner script to load after first paint (most consent-banner vendors
  support a deferred/lazy init mode).

## Finding #2 — No cache headers on static assets

**Severity: Medium.** CSS/JS/image files have an `ETag` but no `Cache-Control` or
`Expires` header at all. That means every single page view re-validates every asset with
the server (a round trip each) instead of just using the local browser cache. Some CSS
is already version-stamped in the URL (`app.css?v=202604231214`), so those specific files
could safely get a far-future, immutable cache header right now with zero risk of serving
stale content.

## Finding #3 — Accessibility (axe-core: 10 violations, 2 critical)

Full detail in `audit-data/axe.json`. Headline items:

- **Critical — 10 images missing alt text** (`image-alt`). Screen reader users get nothing
  for these.
- **Critical — zoom/pinch disabled** (`meta-viewport`): `user-scalable="no"` is set in the
  viewport meta tag. This actively blocks low-vision users from zooming in on mobile, and
  Lighthouse also flags it directly. Easy fix: drop `user-scalable=0` and `maximum-scale=1.0`.
- **Serious — 4 links with no discernible text** (icon-only social links in the footer
  with no `aria-label`).
- **Serious — low color contrast** on 2 elements, including the cookie consent button.
- **Serious — iframe with no accessible title**, and one `<ul>` with non-`<li>` children
  (the main slider).
- **Moderate** — heading levels skip a level in places, no single `<h1>`, and a couple of
  regions aren't contained in a landmark.

## Finding #4 — SEO basics

- `robots.txt` and `sitemap.xml` both resolve — but return the homepage **HTML**, not an
  actual robots file or XML sitemap (the site's router catches unmatched paths and serves
  the homepage). Lighthouse flags `robots.txt` as invalid for this reason. Search engines
  requesting either file get served the wrong content, which can affect crawl behavior
  (and the sitemap doesn't actually tell Google about the room/apartment subpages).
- Otherwise SEO basics (titles, meta description, mobile-friendly tag) are present.

## Finding #5 — JS/CSS bloat

Lighthouse estimates **~479KB of unused JavaScript** and **~61KB of unused CSS** being
shipped on every load (library code for features not used on that page — likely leftover
jQuery plugins / Bootstrap components). Not urgent, but worth trimming once the
render-blocking fix is in, since it adds to the same download queue.

## What to actually do, in order

1. `defer` the 8 blocking scripts / consolidate trackers into GTM. **This is the fix that matters.**
2. Fix `user-scalable=0` in the viewport meta tag (one line).
3. Add `alt` text to the 10 flagged images.
4. Add real `robots.txt` / `sitemap.xml` files.
5. Add `Cache-Control` headers to static assets.
6. (Later, optional) trim unused JS/CSS.

## Raw data

- `audit-data/lighthouse-mobile.json` / `lighthouse-desktop.json` — full Lighthouse runs
- `audit-data/axe.json` — full axe-core accessibility scan (10 violations, 42 passes)

Tested 2026-11-08 against the live site, no login/account access, nothing destructive —
just read-only requests and a Lighthouse run like any visitor's browser would make.
