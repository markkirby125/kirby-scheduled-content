# Scheduled Content Publishing — Edge-Gated Design (Portable Pattern) - Dispatcher

**Pattern ID:** SCHED-EDGE-001
**Status:** Approved portable design.
**Scope of this doc:** project-agnostic design, reusable on any static-site + edge-worker setup.
**Related:** Shared nav/header/footer drift on flat HTML is `kirby-chrome-drift-prevention`. CMS staging, publishing velocity, and satellite deploys are `kirby-seo-deployment`.

---

## 1. Problem Statement

Content teams want to write and deploy articles ahead of time, but keep each article invisible to the public, to search crawlers, and to LLM retrieval until a specific date and time — with **no human action required at the moment of publishing**.

Traditional solutions each fall short on a pure static-hosting setup:

| Approach | Why it fails here |
| --- | --- |
| WordPress-style post status | No application server; content ships as static files. |
| Deploy at publish time (cron + redeploy) | Requires a build machine awake at the publish minute; ties publishing to deploy infrastructure. |
| Client-side JS date check | Content is fully present in the HTML source; crawlers and LLMs read it anyway. Useless for hiding. |
| robots.txt / noindex only | Hides from indexing, not from humans; also hides it *forever*, not "until a date". |
| Cloudflare Access / WAF rules | Requires manual rule toggling per post; not content-driven. |

## 2. Core Insight

If the edge worker runs **before** static assets are served (`run_worker_first: true` on Workers Assets, or an equivalent "worker intercepts all requests" mode on any platform), then the worker is a natural, stateless gatekeeper. A publish schedule is just data the worker can consult on every request:

> **Gate rule:** serve the asset if and only if `now >= publishAt`. Otherwise behave exactly as if the file did not exist.

No KV, no Durable Objects, no Cron Triggers are needed. The worker already executes per request; `Date.now()` at the edge is the scheduler.

## 3. Architecture

```
                 ┌──────────────────────────────┐
 request ───────►│        Edge Worker            │
 /blog/<slug>/    │                              │
                 │  1. parse slug from URL       │
                 │  2. look up slug in           │
                 │     PUBLISH SCHEDULE          │
                 │  3a. not scheduled            │──► serve ASSETS as today
                 │  3b. scheduled, now < publish │──► 404 + X-Robots-Tag: noindex
                 │      (uncached response)      │      (indistinguishable from
                 │  3c. scheduled, now >= publish│      a post that never existed)
                 │      → fall through to ASSETS │
                 │                              │
                 │  4. /sitemap.xml → generate   │──► live-filtered sitemap
                 │     dynamically, excluding    │
                 │     unpublished slugs         │
                 └──────────────────────────────┘
```

### 3.1 The publish schedule (single source of truth)

A plain **JSON data file** in the deployable source tree, e.g. `src/blog/schedule.json` (JSON, not TypeScript, so non-developers can schedule posts by hand-editing data without touching code):

```json
{
  "posts": [
    { "slug": "my-new-post", "publishAt": "2026-09-15T08:00:00Z" }
  ]
}
```

Consumed through a thin typed wrapper module (e.g. `src/blog/schedule.ts`) that loads the JSON, validates it against `ScheduledPost { slug: string; publishAt: string }`, and exposes the pure helpers. TypeScript cannot type-check raw JSON contents at authoring time, so the wrapper performs explicit runtime validation on load: unknown fields rejected (entry-level and top-level), slugs lower-kebab, `publishAt` a full zoned ISO 8601 timestamp (naive datetimes and date-only forms rejected — a bare date would publish at a surprise midnight), duplicate slugs rejected. A malformed entry fails loudly (thrown typed error) rather than silently unpublishing or publishing a post.

Rules:
- **UTC everywhere internally.** Content team schedules in local time; a helper converts to ISO on entry. Never compare local-time strings.
- The same file is read by the worker (runtime gate) and by any build-time tooling (index/llms.txt generation, sitemap baseline), so there is exactly one place a date can be wrong.
- Timezone policy must be explicit: edge workers run on UTC; document that `publishAt` is UTC or store an explicit offset in the ISO string. A scheduling helper (or the validation warning) should flag naive datetimes without a zone designator.

### 3.2 Request gating (the runtime gate)

In the worker's `fetch` handler, before the asset fallthrough:

1. Normalise the URL path; match `/blog/<slug>` and `/blog/<slug>/` (and any extensionless variants the host serves).
2. Look up `slug` in the schedule. **Not present → normal behaviour.**
3. Present and `Date.now() < Date.parse(publishAt)` → return:

   ```
   HTTP/1.1 404 Not Found
   X-Robots-Tag: noindex
   Cache-Control: no-store, max-age=0
   ```

   Rationale for each decision:
   - **404, not 302/403** — a 404 is indistinguishable from a typo'd URL; it leaks the least information and produces no soft-404 index signals for a URL that will exist later. A 403 or redirect invites curiosity and crawling.
   - **`X-Robots-Tag: noindex`** — belt-and-braces; 404s are already dropped by crawlers, but explicit noindex guards against a cached/edge-cached variant leaking.
   - **`Cache-Control: no-store`** — critical: the 404 must never be cached at the edge or by any intermediary, or the post stays invisible *after* its publish time until cache expiry. The *content* response, once served, can use the site's normal caching policy.
   - The gate must apply to **every URL variant** the asset server would answer for the post (with/without trailing slash, `.html` variants) — enumerate them in tests.

4. Once `now >= publishAt`, the handler simply does not fire the 404 and the static asset is served as normal. Publishing is automatic to the minute, with zero moving parts.

### 3.3 Discovery surfaces (where hidden content leaks)

An unpublished post must be absent from **every surface that enumerates content**, not just gated at its URL:

| Surface | Leak risk | Handling |
| --- | --- | --- |
| `sitemap.xml` | **High — worst leak.** A listed URL advertises the post even if the page 404s. | Serve sitemap dynamically from the worker, filtered by `now >= publishAt` at request time. (Worker-first config means the worker intercepts the static file with no config change.) |
| Blog index / category listings | Medium — titles/links visible on the listing page. | Simplest: exclude scheduled posts from the static index at authoring time; they appear on the next deploy after publish. Fully automatic alternative: serve the listing from the worker with the same filter (more moving parts — adopt only if listing freshness to the minute is required). |
| `llms.txt` / content manifests | Medium — machine-readable leak. | Same filter as sitemap; regenerate at build or serve dynamically. |
| JSON-LD `BlogPosting`/`itemListElement` on other pages | Low unless posts are cross-referenced. | Never reference a scheduled slug from live content before publish. |
| Internal links from other posts | Medium. | Same rule: no links to scheduled slugs until the post is live. |
| RSS/Atom feeds (if any) | High where present. | Filter identically to sitemap. |
| Git history | Not a leak (repo is private), but note it: the content *is* in the deployed asset bundle — the gate is the only thing hiding it. |

### 3.4 Fallback and failure modes

- **Worker unavailable / bypassed** (e.g. misconfig removes `run_worker_first`, or a future platform migration serves assets directly): the gate silently disappears and all scheduled content is live. Mitigations: a config test asserting worker-first is enabled; a pre-deploy check that the schedule file has no entries older than today (stale schedules mean the worker *and only the worker* was ever hiding content — keep the list pruned).
- **Rescheduling or unscheduling a page that already has impressions**: re-gating returns 404 + `noindex` for a URL that search engines have already seen, which can take it out of results once it is recrawled and can lose accumulated signals (directional, unverified). Once a slug has impressions (check Search Console before editing), do not move its `publishAt` into the future or add it back to the schedule; unpublish only deliberately, with a redirect or 410 decided per URL and implemented in host or CDN rules (outside this pattern, which only serves or 404s), not via the gate. If a rescheduled slug is unavoidable, note that the 404 only applies to requests that reach the worker: copies already cached by browsers, intermediaries or crawlers may keep serving the old 200, and this pattern has no purge step, so plan any cache purge separately. Record the reason in the commit message; test item 10 in §3.5 covers it.
- **Clock skew**: edge workers are UTC-accurate to seconds; not a concern. Do not build minute-level "grace period" logic — the requirement is "hidden until X", and serving at exactly X is correct.
- **Slug typos in the schedule**: a scheduled slug with no matching deployed page just means the gate never fires — harmless. A deployed page missing from the schedule publishes immediately — the failure mode to test for. Test: every slug in the deployed blog tree is either in the schedule or intentionally public (a build-time drift check).

### 3.4a Cadence guard (warn on bursty schedules)

The validator in §3.1 checks entry shape and timestamp format but enforces no distribution or spacing, so a schedule can put 50 posts live in one minute. Bursts of near-identical publication times, and perfectly regular spacing, are reported to read as automated mass publishing (directional, unverified; not a confirmed Google rule). Add a non-blocking warning to the schedule validation, with thresholds the owner can change:

- **Same-hour burst:** warn if more than `N_hour` entries (suggested start: 2) share one UTC hour.
- **Same-day burst:** warn if more than `N_day` entries (suggested start: 5) share one UTC day.
- **Mechanical spacing:** warn if `N_regular` or more consecutive entries (suggested start: 4) are spaced by an interval identical down to the minute and shorter than `N_interval` (suggested start: 24 hours). A real weekly or daily calendar is not flagged. Flag the batch once, not each post.
- Thresholds are local editorial policy, not Google numbers. The warning explains the count and the affected slugs; it never blocks a deploy and never rewrites `publishAt` values on its own. It considers future, unpublished entries only. The owner decides whether a burst is legitimate (a launch, an embargo, a series) and records the reason in the commit message or review note that edits the schedule (the §3.1 schema stays `{ slug, publishAt }`).
- Do not "fix" a warning by adding random jitter to timestamps: that is cosmetic cadence-masking, not editorial planning. Space posts because the content calendar is real. For the wider velocity guidance, see `../../kirby-seo-deployment/references/module_7_deployment_staging_publishing_velocity_network_isolation.md` §7.2.

### 3.5 Test matrix (minimum)

1. Unscheduled post URL → 200, normal headers.
2. Scheduled post, before `publishAt` → 404 + `X-Robots-Tag: noindex` + `Cache-Control: no-store`, for **every** URL variant of the slug.
3. Scheduled post, after `publishAt` → 200 with content.
4. Scheduled post, exactly at `publishAt` → 200 (boundary inclusive policy: `>=`).
5. Dynamic sitemap excludes pre-publish slugs and includes post-publish slugs.
6. Blog index/manifest excludes scheduled slugs (per chosen strategy).
7. Schedule-module drift check: every deployed blog slug accounted for.
8. Time-injection: gate logic takes `now` as a parameter (pure function) so tests don't fake the clock.
9. Cadence guard (§3.4a): one fixture per rule, each built to trip only that rule and stay inside the other thresholds (more than `N_hour` entries in one UTC hour; more than `N_day` in one UTC day across distinct hours; `N_regular` identically spaced entries under `N_interval`), each produces a warning naming those slugs. A fixture within all thresholds, including a weekly calendar, produces none, and no fixture changes the exit code.
10. Rescheduled slug: 404 before its new `publishAt`, 200 and present in the sitemap at and after it.

### 3.6 Porting checklist (for other projects)

To reuse on another site, replace the platform-specific bits:

1. Confirm the edge layer can run before assets (Workers `run_worker_first`, Vercel middleware, Netlify edge functions, CloudFront Functions, etc.). If it cannot, this pattern needs the "deploy at publish time" fallback from §1 instead.
2. Identify the content surface tree to gate (here `/blog/<slug>/`; could be `/guides/`, `/news/`, any enumerable folder).
3. Create the schedule file + typed wrapper (§3.1) in the source tree.
4. Implement the gate (§3.2) and the discovery filters (§3.3) for that project's actual enumeration surfaces.
5. Adopt the test matrix (§3.5); make `now` injectable first. Add the cadence guard (§3.4a) to the schedule validation.
6. Add the drift check (§3.4) so "deployed but unscheduled" can never happen silently.

### 3.7 Anti-goals

- No client-side hiding (trivially bypassed, crawlers ignore it).
- No CMS/auth layer — this is a low-friction pattern for a small static content operation, not a staging environment. Draft *quality* control stays in review, not at the edge.
- No per-post Cloudflare dashboard rules — everything must be data-driven from the committed schedule.
