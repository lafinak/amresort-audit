# amresort.sk — quick web audit

Informal audit, not a client deliverable. Ran Lighthouse (mobile + desktop) and axe-core
against the live site on 2026-11-08, plus some manual checks (headers, robots.txt,
render timing). Raw tool output is in `audit-data/`.

## In plain terms

Your website works, but on a phone — or anything less than a fast connection — it can take
**20 to 70+ seconds** to finish loading, because the homepage silently loads a **43MB
autoplaying background video**. That's roughly the size of 10,000 regular web pages, for
one looping background clip. Most visitors on mobile data will give up before it finishes.

The cause isn't your text, your layout, or your hosting — it's one oversized file. Checked
directly against the server: that 43MB video is recorded at a quality level (1080p, 60
frames/second) that's complete overkill for a muted background loop nobody watches
frame-by-frame. I compressed the exact same clip to the kind of setting background videos
normally use (720p, 30fps) and it came out to **6.6MB — 84% smaller**, with no visible
quality loss for its actual use. I verified that size reduction actually cuts load time by
roughly the same proportion by timing partial downloads directly against the real server
(details below) — this isn't a guess.

**This is a one-file swap, not a rebuild.** Whoever has access to the site's media files
re-encodes that one video and re-uploads it. No redesign, no lost content, nothing else on
the page needs to change. Everything else in this document (a few missing image
descriptions, a disabled pinch-to-zoom setting, some oversized photos, a couple of SEO
files) is smaller and optional, listed below for completeness.

**Who can fix this:** the page's own code names `Sixnet` as its author, and one of its
scripts is literally called `sixnet-footer.frontend.js`. That suggests the site runs on a
platform built by an agency called Sixnet, not something editable from a simple
page-builder. Worth checking who actually has file/CMS access before promising a timeline.

## Methodology (for anyone who wants to verify this independently)

- **Tools:** Google Lighthouse 13.6.0 (CLI, mobile + desktop presets, default throttling),
  axe-core 4.10.2 (run live in Chrome against the rendered page), and direct `curl`/`ffprobe`
  checks against the real server (response headers, file sizes, codec/bitrate, range-request
  timing). All open-source/standard tools, nothing custom. Raw output is in `audit-data/` —
  re-run the same commands and you'll get the same numbers.
- **Self-correction, in the interest of being accurate rather than first:** an earlier pass
  of this audit identified 8 "render-blocking scripts" as the main culprit, based on a
  JavaScript query that (bug on my part) counted scripts anywhere in the document, not just
  `<head>`. Rechecking the actual HTML showed those scripts sit at the bottom of `<body>`
  (correct placement) and GTM already loads itself `async` — that finding was wrong, and
  real-browser testing pointed to the video instead. I'm leaving this note in rather than
  quietly fixing it, since the whole point of linking raw data is to let you catch things
  like this too.
- **Access:** read-only. No login, no account, no write requests — the same requests any
  visitor's browser makes, plus HTTP header/range-request checks (standard things any
  browser or CDN does, nothing adversarial).
- **Date:** 2026-11-08 — note the file versions if re-checking later, the site may have
  changed.

## TL;DR

The site is **fine on desktop, bad on mobile/slow connections** — and it's effectively one
root cause: a massively oversized autoplay video, not a pile of unrelated problems.
Fixing that one file should fix most of the performance score. Everything else below is
smaller and optional.

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

Lighthouse's own payload audit independently flagged **11.6MB total transferred** on this
page and named several multi-megabyte images as top offenders — consistent with "this page
ships way too many raw bytes," even where it didn't capture the video itself (see note in
Finding #1).

## Finding #1 — A 43MB autoplaying background video (the big one)

**Severity: High. This is what's actually making the site feel slow.**

The homepage has a `<video autoplay preload muted loop>` background clip. Checked directly
against the server:

| | Original | What it should be |
|---|---|---|
| File size | **42.8 MB** | 6.6 MB (tested) |
| Resolution / framerate | 1920×1080 @ 60fps | 1280×720 @ 30fps |
| Bitrate | 7.1 Mbit/s | 1.1 Mbit/s |

That resolution/framerate/bitrate is what you'd use for footage people actually watch
closely — not a muted, looping, autoplaying background element. Nobody can perceive 60fps
or 1080p detail in a blurred-motion background loop.

**How I confirmed this is the actual bottleneck, not a guess:**
- Direct download of the live file took 37-73 seconds across two separate measurements
  (this connection's throughput itself varied between ~588KB/s and ~1.16MB/s between runs —
  worth knowing, it's noisy). All other individual assets on the site (JS, CSS, icons)
  loaded in under 1 second each via the same checks, ruling out a general server problem.
- I re-encoded the actual video (same codec, H.264, just resized/re-bitrated) to 6.6MB —
  an 84% size cut — using `ffmpeg`, and confirmed visually it's still fine for a background
  loop.
- To prove size is really what drives the time (not something else I'm not seeing), I used
  an HTTP range request to ask the real server for only the first 6.6MB of the original
  file — i.e., "how long would an optimized-size file take on this exact server, right
  now." It came back in **6.66 seconds**, matching the ~85% proportional time cut you'd
  expect from an 85% size cut, independent of which exact bandwidth number that moment's
  connection happened to give. That's as close to testing the real fix as you can get
  without write access to the server.
- Lighthouse's own lab run didn't count the video in its "total byte weight" list — that
  audit only counts fully-completed transfers within its test window, and a 42MB file at
  this server's throughput often doesn't finish before Lighthouse stops measuring. That's
  not a contradiction, it's the same problem showing up as a timeout instead of a byte
  count.

**Fix:** re-encode the hero video at a sane bitrate/resolution for a background loop
(roughly what's in the "should be" column above) and re-upload it. One file, no redesign.

## Finding #2 — Several oversized images (Lighthouse-confirmed)

**Severity: Medium.** Independent of the video, Lighthouse's own payload audit flags a
handful of package/promo images at 2.6-3.5MB **each** as raw PNGs (`web-_titulky.png`,
`vianoce_v_tatrach.png`, `silvester_v_tatrach.png`), while other images on the same page
are already served as WebP at a fraction of the size. Converting these specific files to
WebP/compressed JPEG (same approach already used elsewhere on the site) would save
roughly 8-10MB of page weight — see `cache-insight` / `image-delivery-insight` in
`audit-data/lighthouse-mobile.json` for the exact file list.

## Finding #3 — No cache headers on static assets

**Severity: Medium.** CSS/JS/image files have an `ETag` but no `Cache-Control` or
`Expires` header at all. That means every single page view re-validates every asset with
the server (a round trip each) instead of just using the local browser cache. Some CSS
is already version-stamped in the URL (`app.css?v=202604231214`), so those specific files
could safely get a far-future, immutable cache header right now with zero risk of serving
stale content.

## Finding #4 — Accessibility (axe-core: 10 violations, 2 critical)

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

## Finding #5 — SEO basics

- `robots.txt` and `sitemap.xml` both resolve — but return the homepage **HTML**, not an
  actual robots file or XML sitemap (the site's router catches unmatched paths and serves
  the homepage). Lighthouse flags `robots.txt` as invalid for this reason. Search engines
  requesting either file get served the wrong content, which can affect crawl behavior
  (and the sitemap doesn't actually tell Google about the room/apartment subpages).
- Otherwise SEO basics (titles, meta description, mobile-friendly tag) are present.

## Finding #6 — JS/CSS bloat

Lighthouse estimates **~479KB of unused JavaScript** and **~61KB of unused CSS** being
shipped on every load (library code for features not used on that page — likely leftover
jQuery plugins / Bootstrap components). Minor next to the video/images above, but still
worth trimming eventually since it adds to the same download queue.

## What to actually do, in order

1. Re-encode the 43MB hero video down to something like 720p/30fps/~1Mbit/s. **This is the fix that matters.**
2. Convert the 3 oversized package PNGs (2.6-3.5MB each) to WebP, same as other images on the site already are.
3. Fix `user-scalable=0` in the viewport meta tag (one line).
4. Add `alt` text to the 10 flagged images.
5. Add real `robots.txt` / `sitemap.xml` files.
6. Add `Cache-Control` headers to static assets.
7. (Later, optional) trim unused JS/CSS.

## Raw data

- `audit-data/lighthouse-mobile.json` / `lighthouse-desktop.json` — full Lighthouse runs
- `audit-data/axe.json` — full axe-core accessibility scan (10 violations, 42 passes)
- `audit-data/hero-video-optimized-example.mp4` — the actual re-encoded hero video (6.6MB,
  720p/30fps, same clip, same codec). Not a mockup — whoever maintains the site can take
  this file as-is, or re-encode fresh from the original at a similar setting.

Tested 2026-11-08 against the live site, no login/account access, nothing destructive —
just read-only requests and a Lighthouse run like any visitor's browser would make.
