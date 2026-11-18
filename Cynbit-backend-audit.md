# Cynbit Technologies · Passive Backend Audit

**Re-verified 18 November 2026 · Scope: unauthenticated public browsing on `www.cynbit.in`**

Prioritize removing raw blog search terms from third-party telemetry. A normal search for `cloud` sent the term to Google Analytics in a dedicated search event and in the page location carried by both the search event and a page-view event. Two homepage captures each showed 13 distinct same-origin route-data URLs carrying `_rsc`. Measure their transfer and cache cost before tuning prefetch. For Contact, verify the server-side intake contract and align the form’s browser constraints with it.

## 1. Backend Technical Audit Report

### Scope and evidence

Reopened the public site in a fresh in-app browser tab and compared the new observations with the first pass. Used the browser resource inventory, rendered markup, and ordinary Blog navigation. The Blog search used two harmless editorial terms through the site’s search box. No data-writing forms were submitted. No sign-ins, request modifications, endpoint guesses, scanners, or load tests were performed. Inventory counts reflect distinct observed URLs and vary with viewport, time on page, browser state, and cache. Only the audit-generated search term and public location are reproduced from analytics queries. Client and session field values are omitted.

| Evidence | Public journey | Observed interface and data |
|---|---|---|
| E1 | [Homepage](https://www.cynbit.in/) | Both snapshots showed 13 distinct first-party URLs carrying `_rsc` and 24 script URLs. Full route list below. A Google Analytics `page_view` request appeared in the recheck before any input or click by the auditor. `stats.g.doubleclick.net/g/collect` appeared in the first capture only. Google Tag Manager and Google Fonts also appeared in the resource inventory. |
| E2 | [Blog](https://www.cynbit.in/blog) | The search input has `name="q"` and no HTML `maxlength`. Entering `automation` in the search box navigated to `/blog?q=automation` and rendered a changed set of article cards. `/blog?category=Education` rendered a different set. The base page and `/blog?page=1` showed the same sampled article headings, while `/blog?page=2` showed different headings. A `page=0` link was present in the base page markup. Numbering semantics, complete response equivalence, and cache effects remain unverified. |
| E3 | [Contact](https://www.cynbit.in/contact) | The rendered form has `fullName` (`text`, required), `email` (`email`, required), `company` (`text`, optional), `industry` (`text`, optional), and `message` (`textarea`, required). The form declared neither an HTML `action` nor `method`, and the five fields had no HTML `maxlength` attribute. These are DOM attributes. The field names do not establish the wire schema. |
| E4 | [Privacy Policy](https://www.cynbit.in/privacy) and [Cookie Policy](https://www.cynbit.in/cookies) | The privacy page describes collecting contact details through forms. The cookie page describes Google Analytics as anonymous, lists interaction metrics, and directs visitors to browser cookie settings. It lists session tokens and CSRF protection as examples of essential cookies. These are policy statements. Actual cookie issuance, consent state, and retention remain unverified. |
| E5 | [Blog search results](https://www.cynbit.in/blog?q=cloud) | Entering `cloud` through the public search box produced two observed requests to `analytics.google.com/g/collect`. `view_search_results` carried the term in `ep.search_term` and in the `dl` page-location field. `page_view` carried it in `dl`. Both requests included `cid` and `sid` fields. Their values and persistence were outside the review. |
| E6 | [Homepage](https://www.cynbit.in/) and [Blog](https://www.cynbit.in/blog) markup | Found `/_next/static/` script references and inline scripts containing `self.__next_f.push`. Measured 8,873 characters of matching inline script text on the homepage and 33,494 on the base Blog page. Those are rendered DOM string lengths. They provide a reproducible indicator of inline framework data volume, separate from compressed wire size and standalone `_rsc` bodies. |

### Reproducible request and payload excerpts

The E1 route list was `/story`, `/team`, `/careers`, `/impact`, `/services`, `/products`, `/case-studies`, `/bitelevate`, `/bitelevate/paths`, `/diaries`, `/blog`, `/contact`, and `/blog/ai-agents-business-operations`. The inventory showed `_rsc` on each. Reproduce by loading the homepage normally, waiting for the resource list to populate, and filtering the generated requests by `_rsc`.

The E5 excerpt shows decoded request query parameters. Other fields and all client and session values were removed. Reproduce with an innocuous search term and inspect the browser’s resulting collection requests.

```text
Destination: https://analytics.google.com/g/collect

en=view_search_results
ep.search_term=cloud
dl=https://www.cynbit.in/blog?q=cloud
cid=[omitted]  sid=[omitted]

en=page_view
dl=https://www.cynbit.in/blog?q=cloud
cid=[omitted]  sid=[omitted]
```

### Supplemental Chrome Network evidence

Two user-supplied screenshots show the Network panel during public navigation on the Blog and Contact pages. **Keep log** was enabled, **Disable cache** was unchecked, and the browser showed **No throttling**. The same two fetch rows appear in both screenshots and are counted once.

| Visible request row | Status | Type | DevTools Size | DevTools Time |
|---|---:|---|---:|---:|
| `best-automation-tools-2026?_rsc=…` | 200 | fetch | 8.2 kB | 1.39 s |
| `blog?_rsc=…` | 200 | fetch | 4.8 kB | 648 ms |

The screenshots also show several `collect` rows with HTTP `204`. These rows corroborate successful collection requests in this browser session. Their request details are collapsed, so the screenshots do not tie those statuses to the specific `cloud` events in E5. The two route times are browser-observed request durations from one log. The TTFB breakdown, origin time, full requested URLs, route response bodies, and response headers are not visible. Because the log spans navigation, its footer totals cannot be attributed to the currently displayed page alone.

### E7 · Blog route headers, response, and detailed timing

A subsequent user-supplied capture shows an HTTPS `GET` to `www.cynbit.in/blog?_rsc=…`, with `Rsc: 1` and a `Next-Router-State-Tree` request header. The response was `200 OK`, served by Vercel, with `X-Matched-Path: /blog.rsc`, `Content-Type: text/x-component`, and Brotli (`br`) encoding. This confirms delivery of a React Server Component (RSC) navigation response. The browser sent Google Analytics cookies on this same-origin request. Their values are intentionally omitted. The request view does not establish how those cookies were set or which attributes they have.

| Observed response detail | Value |
|---|---|
| `Cache-Control` | `private, no-cache, no-store, max-age=0, must-revalidate` |
| `Age` / `X-Vercel-Cache` | `0` / `MISS` |
| `Vary` | `rsc, next-router-state-tree, next-router-prefetch, next-router-segment-prefetch` |
| `Strict-Transport-Security` | `max-age=63072000` |

The supplied response is a React Flight stream with component references and route-tree data. Record `0` identifies the Blog route, record `6` contains page metadata, and record `2` serializes the page component tree. The tree includes the `q` search input, category and pagination links, article card content, image references, and client chunk references. For example, it contains `href: /blog?category=Education` and `href: /blog/best-automation-tools-2026`. This is the browser-facing rendering payload. The underlying content store and service interfaces remain unverified.

The Timing tab shows **1.28 s total**, including **385.54 ms waiting for the server response** and **890.39 ms content download**. No server-side timing entries were displayed. The download phase can include time between streamed chunks, so this sample cannot attribute the delay to network throughput, origin rendering, or a downstream service. The earlier 648 ms row is retained as a separate displayed measurement. The captures do not provide a controlled comparison or a latency distribution.

### E8 · Contact HTML document measurement

A user-supplied Chrome Network screenshot with the **Doc** filter selected shows the `contact` document request returning HTTP `200`. DevTools displays **11.4 kB transferred**, **64.0 kB resource size**, and **109 ms elapsed** for the request. The page markers show `DOMContentLoaded` at **597 ms** and `Load` at **787 ms**. These are browser observations from one navigation. The detailed Timing breakdown was not captured.

A follow-up capture of the same document row verifies `GET https://www.cynbit.in/contact` and the response headers below. Analytics cookies are visible in the request. Their values are omitted, and the capture does not establish their setting attributes or any authentication state.

| Observed response detail | Value |
|---|---|
| `Content-Type` / `Content-Encoding` | `text/html; charset=utf-8` / `br` |
| `Server` / `X-Powered-By` | `Vercel` / `Next.js` |
| `X-Matched-Path` / `X-Nextjs-Prerender` | `/contact` / `1` |
| `Cache-Control` | `public, max-age=0, must-revalidate` |
| `Age` / `X-Vercel-Cache` | `56170` / `HIT` |
| `ETag` | Weak validator present |
| `Vary` | `rsc, next-router-state-tree, next-router-prefetch, next-router-segment-prefetch` |
| `Strict-Transport-Security` | `max-age=63072000` |

The Vercel cache `HIT` and Next.js prerender marker show that this Contact HTML request was served from cached, prerendered output. The response sends a zero freshness lifetime via `max-age=0`; the headers alone do not establish Vercel's internal TTL or invalidation policy. The Blog RSC response in E7 is a different route and representation. Its `MISS` and timing cannot be compared with this Contact cache `HIT` as a controlled cache experiment.

The captured Contact response header list shows HSTS. It contains no `Content-Security-Policy`, `X-Frame-Options`, or `X-Content-Type-Options` entries. This observation applies to the sampled response. Effective page framing and CSP behavior, including any policy delivered in markup where applicable, were not tested.

### Observable architecture

```text
Unauthenticated browser
  ├── www.cynbit.in public pages and static resources
  │     └── Route-data URLs carrying _rsc (observed)
  ├── Google collection endpoints (observed during homepage load)
  └── Contact form field schema (observed in DOM)
        └── Submission endpoint and downstream handling (unverified)
```

Next.js static asset paths, inline Flight chunks, and the verified Blog RSC response support Next.js App Router delivery on Vercel. The HTML document exposes inline framework data alongside rendered content. Blog filters and pagination operate through a public page read interface that can deliver HTML or framework navigation data. The observable telemetry interface accepts URL-encoded event parameters. Public route delivery could use prerendered content, edge caching, request-time rendering, or a combination. The sample cannot locate the origin, database, or content store, or identify a separately exposed first-party JSON API for these journeys. [Next.js prefetching guide](https://nextjs.org/docs/app/guides/prefetching)

E7 verifies the method, response headers, body format, and detailed browser timing for one Blog navigation response. E8 verifies the Contact HTML document headers and a cached prerendered document response. The homepage and Blog HTML document headers, cookie-setting attributes, and Contact submission transport remain unverified. The sampled pages were accessible without sign-in. Internal authentication, authorization, CSRF enforcement, abuse controls, and downstream handling require evidence from Cynbit’s team. No security rating is assigned to unverified controls.

## 2. Security & Performance Findings

| ID | Category | Priority | Evidence and impact |
|---|---|---|---|
| F1 | Privacy and data minimization | P1 · Confirmed outbound data | The browser included the raw search term in requests to Google Analytics through two fields across two event types (E5). Someone could enter private or identifying information in the free-text box. The tested term was harmless. Receiver acceptance, prior visitors’ values, consent compliance, advertising use, and retention remain unverified. The homepage Google Analytics request recurred. DoubleClick appeared once (E1). |
| F2 | Performance and origin efficiency | P2 · Confirmed route traffic and one uncached response | Both homepage captures recorded 13 `_rsc` route URLs (E1). One Blog route response had `no-store` and a Vercel cache `MISS`. Its timing breakdown showed 1.28 s total (E7). Inspect cache eligibility and navigation reuse before changing policy. Typical latency, cause of delay, and origin request volume remain unverified. |
| F3 | Input handling and error contracts | P2 · Confirmed browser constraint gap, backend review needed | Five Contact fields lack browser-side length limits (E3). Inspect the server’s request-body bound, validation, rate limits, error responses, and data retention before assuming any enforcement gap. Align browser constraints with the server rules. |
| F4 | Read interface and cache efficiency | P3 · Candidate improvement | The Blog index and `page=1` showed the same sampled headings, `page=2` changed them, and base markup exposed `page=0` (E2). Define one canonical representation of the first page if the complete results and semantics match. Equivalent URLs can create duplicate cache entries or confusing pagination if each representation is keyed independently. Neither effect was measured. |
| F5 | Browser security headers | P3 · Hardening opportunity | HSTS was present on the Contact document. Its captured response did not expose CSP, X-Frame-Options, or X-Content-Type-Options (E8). Review script and framing policy for this public form. The capture establishes the header state for one response. It does not establish an exploitable issue or sitewide configuration. |

Priority reflects investigation order and plausible impact. The audit established observable behaviors and implementation opportunities. Exploitability, security control failures, and performance regressions remain unverified.

## 3. Recommended Improvements

### F1 · Keep raw search terms out of third-party telemetry

Disable automatic Site Search measurement if its raw terms are unnecessary. Strip `q` from analytics page location and referrer fields, and configure GA4 URL-parameter redaction as defense in depth. Turning off the dedicated search event alone still leaves the page-location field in `page_view`. Preserve search analytics through a consent-gated event that counts a results view without including user input. Route all page views through the same sanitization path and configure tags to avoid duplicate automatic events.

In the owner’s environment, capture homepage requests before a privacy choice, after rejection, and after acceptance. Check whether the observed DoubleClick destination is expected. If Cynbit's policy permits optional analytics only after opt-in, gate tag loading and honour revocation. Document the actual categories and purposes in the cookie notice.

```ts
// Apply before sending manually configured page views or search counts.
function analyticsLocation(rawUrl: string): string {
  const url = new URL(rawUrl)
  return `${url.origin}${url.pathname}`
}

const fields = { page_location: analyticsLocation(window.location.href) }
// Send fields through the consent-gated analytics wrapper.
// Apply the same sanitizer to page_referrer if that field is collected.
```

**Acceptance check:** With analytics enabled, an ordinary search yields no raw term in the search event, page location, referrer, or any other outbound field. Under an opt-in-only configuration, fresh and rejected sessions generate zero optional analytics events. Revocation prevents subsequent optional events. [GA4 Enhanced Measurement](https://support.google.com/analytics/answer/9216061) · [GA4 Data redaction](https://support.google.com/analytics/answer/13544947) · [Google tag consent implementation](https://developers.google.com/tag-platform/security/guides/consent)

### F2 · Tune route prefetching using navigation and cache evidence

Capture an initial homepage visit and a typical click-through in an owner-approved environment. Group `_rsc` requests by route and measure transferred bytes, cache hits, and subsequent navigations. Correlate with origin request volume and route latency. If low-use links account for a material share, disable automatic prefetch on those links first. Review why this Blog response is private and non-storable before any cache change. If editorial data is independent of cookies and user identity, consider caching the public data at the application layer with deliberate freshness and invalidation rules. Any shared response caching also requires preserving the framework's separation of HTML and RSC variants. [Next.js CDN caching guide](https://nextjs.org/docs/app/guides/cdn-caching)

```tsx
import Link from 'next/link'

export function PublicLinks() {
  return (
    <nav>
      <Link href="/contact">Contact</Link>
      <Link href="/team" prefetch={false}>Our Team</Link>
    </nav>
  )
}
```

**Acceptance check:** Fewer speculative requests and bytes on the baseline homepage journey, with no material loss in navigation speed on high-use routes. Confirm the cache serves the correct representation after a publish or content update. [Next.js prefetching guide](https://nextjs.org/docs/app/guides/prefetching)

### F3 · Validate intake at the server boundary and return predictable errors

Verify the existing submission route with the owner first. For a JSON HTTP handler, the following TypeScript sketch proposes a contract based on the observed form field names. It is illustrative and has no dependency on an existing Cynbit endpoint. Apply the same validation policy at the existing entry point if the site uses a server action or form-encoded request.

```ts
import { z } from 'zod'

const inquirySchema = z.object({
  fullName: z.string().trim().min(1).max(120),
  email: z.string().trim().max(254).email(),
  company: z.string().trim().max(160).optional(),
  industry: z.string().trim().max(120).optional(),
  message: z.string().trim().min(1).max(4000),
}).strict()

// Run after a bounded body reader and an abuse-control check.
function validateInquiry(body: unknown) {
  const result = inquirySchema.safeParse(body)
  if (result.success) return { data: result.data }

  return {
    response: Response.json({
      type: 'about:blank',
      title: 'Unprocessable Content',
      status: 422,
      invalidFields: [...new Set(
        result.error.issues.flatMap(issue =>
          typeof issue.path[0] === 'string' ? [issue.path[0]] : []
        )
      )],
    }, {
      status: 422,
      headers: {
        'Content-Type': 'application/problem+json',
        'Cache-Control': 'no-store',
      },
    }),
  }
}
```

Set a total request-body limit before parsing and enforce per-client abuse controls using trusted edge information. Keep free-text messages and email addresses out of application logs and analytics. Record status, latency, and an opaque request ID. If delivery depends on an email or CRM provider, acknowledge queued work only after durable acceptance. Deduplicate retries using a scoped idempotency key and store the associated outcome atomically. Define who can access submitted data and when it expires.

**Acceptance check in staging:** A valid submission creates one durable inquiry. A retry after an ambiguous response returns the recorded outcome without duplicate delivery. Malformed JSON returns `400`, oversized content returns `413`, invalid fields return `422`, and rate-limited requests return `429` with retry guidance. Error bodies avoid raw inputs and internal exception details. No production writes were needed for this audit. [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)

### F4 · Give the Blog read interface a stable URL contract

Confirm the intended indexing before changing redirects. If the base URL and `page=1` are equivalent, generate one canonical URL for page one and contain legacy aliases at the routing layer. Preserve supported filters during normalization. Publish one convention for out-of-range pages and apply server-side bounds to search input and page numbers. Let caches share eligible public listing responses where freshness permits. Review free-text search caching and logging separately because query cardinality is high and the term may contain private information.

**Acceptance check:** Page one has one canonical link, each next or previous link changes the article list as intended, category and search filters persist across pagination, and search terms stay out of shared logs and telemetry where they are unnecessary.

### F5 · Verify and harden the Contact document policy

Review the allowed script, connection, and framing requirements in staging. Start with a report-only CSP to inventory violations, then enforce a policy that permits the site's actual Next.js and third-party dependencies. Set `frame-ancestors 'none'` in the enforced response header if the form has no legitimate embed use. Evaluate `X-Content-Type-Options: nosniff` with the deployed asset types. Preserve prerendering and cache behavior when choosing how to accommodate inline scripts.

**Acceptance check:** A normal Contact document request exposes the intended headers, the page renders and functions in staging, and only approved origins can frame it if framing is required. [Next.js CSP guide](https://nextjs.org/docs/app/guides/content-security-policy)

## Requirement check

| Deliverable | Verified coverage | Evidence boundary |
|---|---|---|
| Backend Technical Audit Report | Browser-to-route architecture, Blog query behavior, Contact input schema, redacted analytics request fields, a verified Blog RSC response, and a Contact HTML header and timing sample | Homepage and Blog HTML headers, remaining route variants, and Contact submission transport and downstream architecture remain unverified. |
| Security & Performance Findings | Data minimization, route prefetch, input handling and errors, Blog routing/cache efficiency, and Contact response header hardening | Authentication is scoped to public access. Session, authorization, and CSRF behaviour require owner-side evidence. Typical performance impact remains unmeasured. |
| Recommended Improvements | Analytics sanitization, selective prefetch, intake validation and error handling, canonical pagination, Contact document policy, and verification criteria | Code snippets are illustrative. Implementation and staging verification remain owner-side work. |
