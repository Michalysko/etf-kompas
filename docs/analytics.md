# Conversion Analytics — Umami Cloud

Analytics is implemented in the deployed demo. No advertising campaign has been run, and this documentation contains no measured campaign results. Dashboard filtering and deduplication capabilities must be verified in the actual Umami account before any conversion conclusions are reported.

The demo sends materials only to pre-approved recipients. A `demo_not_allowed` response creates neither `form_error` nor `material_accepted`; after consent, the valid attempt remains recorded as `form_submit`. Public-visitor conversion rates therefore cannot be evaluated from the current demo.

## Configuration and boundaries

- Tracker: `https://cloud.umami.is/script.js`; current collection endpoint: `https://gateway.umami.is/api/send`.
- Public Website ID: `55615934-0f26-4e71-b7b2-d93427020fa6`. It is not an API key; no Umami API key is required.
- The script renders only after consent and only when both `NODE_ENV=production` and `VERCEL_ENV=production` are set and the browser hostname is exactly `etf-kompas-zeta.vercel.app`.
- Localhost, development, and preview deployments do not send analytics. These guards must not be weakened for local testing.
- The owner reported an EU-region Hobby account and access to Events, Funnels, and UTM views. This is not an authenticated account audit, and specific funnel filters were not verified.
- The tracker is loaded through `next/script` with `afterInteractive`, `data-auto-track=false`, `data-auto-pageview=false`, and `data-performance=false`. Application code enables no automatic click tracking, history patching, identify calls, performance collection, replay, or heatmaps.

## Consent model

The consent panel is part of the normal document flow, not a modal overlay. Both choices have equal visual weight and at least a 44 px target. Analytics settings remain available below the footer. The calculator and both forms work regardless of the choice.

Consent is the selected legal basis for analytics. The localStorage record `etf-kompas.analytics-consent.v1` stores only version 1, a denied state, and denial time. A denial is respected for at most 180 days. Invalid or expired records are removed where possible and are never treated as permission.

**Permission exists only in memory for the current document.** Reload, full navigation, a different tab, or BFCache restoration requires a new explicit opt-in. Historical persistent `granted` records are never activated and are removed where possible. Permission in one long-lived document also expires after 180 days and is checked during every event and in the before-send filter.

Withdrawal immediately closes the gate, discards queues, invalidates attempts, and disconnects transport. If denial can be persisted, the page reloads to remove vendor runtime. If storage fails, the document remains blocked and the UI explains that the preference could not be saved. Previously sent requests cannot be withdrawn.

Only denial is synchronized between same-origin tabs through a storage event and BroadcastChannel `etf-kompas.analytics-consent.v1` with constant message `denied`. Permission, campaign data, and identifiers are never broadcast. If all cross-tab channels and storage are blocked, already permitted isolated tabs cannot be notified reliably and must be closed or revoked individually.

Before consent, no analytics history queue or analytics IntersectionObserver exists. After consent, the in-memory transport queue is limited to 32 records, plus at most 16 pending form events per material while visibility is resolved. Overflow or service failure loses analytics but never blocks the application. There is no offline replay storage or automatic retry.

## Events

| Event | Trigger and allowed properties |
| --- | --- |
| `landing_view` | Once per measured entry to `/`; `ad_variant` |
| `calculator_change` | Blur after a real valid change; `field=monthly_contribution/years/gross_return`, `ad_variant`; never the value |
| `etf_change` | Real select change; `position=first/second`, allowlisted `ticker`, `ad_variant` |
| `form_view` | At least 10% of the form visible; once per material and measured entry; `material`, `ad_variant` |
| `form_start` | First focus/pointer interaction, released only after `form_view`; `material`, `ad_variant` |
| `form_submit` | After client validation and immediately before POST; `material`, `attempt=first/repeat`, `ad_variant` |
| `material_accepted` | Only HTTP 202 with JSON `status=accepted`; `material`, `attempt`, `ad_variant` |
| `form_error` | General category `validation/verification/busy/server/network`; `material`, optional `attempt`, `ad_variant` |

`attempt` distinguishes first and repeated attempts only within the current document. Consent epoch and internal counters are never sent. A pending completion is discarded after leaving the landing page or changing to a new explicit campaign, so it cannot be attributed to a different entry. Native form invalidity may produce `form_error` without `form_submit`.

No values, keystrokes, email data, form contents, DOM snapshots, request UUIDs, or tokens are recorded. Pre-consent activity is never backfilled. Focus may precede the visibility observer; only the fact of interaction is buffered and events are released in order after real visibility. Mounting alone is not a view.

## Payload sanitization

A typed wrapper and a second runtime validator enforce the event schema. The synchronous global `data-before-send` function is defined before tracker loading. It accepts only event payloads, rejects identify/performance and unknown names or properties, and rebuilds the payload from an allowlist rather than forwarding vendor metadata.

The payload contains the fixed Website ID, hostname, ETF Kompas title, `/` with sanitized UTM values, a restricted referrer, and allowlisted event data. Referrers are reduced to the origin of known Facebook, Instagram, Google, or LinkedIn hosts; all others are blank. Paths, credentials, query strings, and fragments are removed. Screen, language, vendor ID/tag, and default title metadata are not sent.

The page sets `referrer=no-referrer`, and the script uses `referrerPolicy=no-referrer`. This limits the HTTP Referer but cannot hide IP address or User-Agent. The tracker itself may read browser metadata, location, referrer, and `umami.disabled`; the filter prevents forwarding unwanted fields but not local reads. Umami derives identifiers from network metadata. ETF Kompas adds no custom visitor ID and makes no claim of complete anonymity.

## Campaign attribution

Allowlisted values:

- source: `meta`, `facebook`, `instagram`, `google`, `linkedin`, `demo`;
- medium: `paid_social`, `cpc`, `social`, `referral`, `demo`;
- campaign: `etf_kompas_demo`, `etf_kompas`;
- content: `ad_a`, `ad_b`.

Unknown or duplicate parameters are discarded. Campaign context is evaluated only after consent and remains in document memory. An internal SPA path from an attributed landing page to privacy and back to bare `/` preserves the campaign. A new explicit valid campaign replaces it; invalid explicit values produce `unknown`. Reload, a new document, BFCache restoration, or withdrawal clears context.

The approved variant links are:

- A: `https://etf-kompas-zeta.vercel.app/?utm_content=ad_a`
- B: `https://etf-kompas-zeta.vercel.app/?utm_content=ad_b`

`utm_content` is the input URL parameter. The controller derives `ad_variant=ad_a/ad_b/unknown` for named events. `ad_variant` is not a supported input parameter.

## Funnel and interpretation

One manually emitted pageview and one named `landing_view` are recorded per measured entry. They are not two visits and must not be added together.

The intended material funnel is:

```text
landing_view → form_view → form_start → form_submit → material_accepted
```

Build separate report and guide views. `material` exists only on form events, so a global material filter may incorrectly remove `landing_view`; apply it only where supported per step. Calculator interaction is optional and must not be a mandatory conversion step.

Provider acceptance is not mailbox delivery, a unique person, or necessarily a newly created message; an idempotent retry may return the original acceptance. Visits and unique visitors are derived Umami metrics, not counts of physical people. Analytics covers only consenting, unblocked, technically recorded activity and cannot be generalized to every visitor.

Umami's Funnel Window is the maximum interval between adjacent steps, not automatically the entire journey. A consistent 60-minute window between steps is the proposed configuration. Filters for `ad_variant`, per-step `material`, and the report's deduplication unit must be verified in the account. If unavailable, report only limited Events/Properties and UTM counts and do not present them as a verified sequential conversion funnel.

## Testing without real email or analytics

1. Run `npm run dev -- --webpack`. On localhost, Umami must remain unloaded even after permission because the production guard is closed.
2. Run `npm test`. Tests use fake React/DOM/fetch and send no email or analytics. The real-tracker test is skipped unless a fixture is supplied.
3. For isolated tracker verification, download the public script to a temporary file, set `UMAMI_TRACKER_TEST_FILE` for the test process, and run `npm test`. The VM uses mock fetch and sends nothing to Umami.
4. Run `npm run lint` and `$env:NODE_OPTIONS='--use-system-ca'; npm run build -- --webpack` in PowerShell.
5. Any production browser inspection or real email request requires separate approval. Inspect Network before consent, after denial, after permission, and after withdrawal; never invent dashboard results.

## Primary sources

- [Tracker configuration](https://docs.umami.is/docs/tracker-configuration)
- [Tracker functions](https://docs.umami.is/docs/tracker-functions)
- [Metric definitions](https://docs.umami.is/docs/metric-definitions)
- [Cloud regions and retention](https://docs.umami.is/docs/cloud/faq)
- [Funnels](https://docs.umami.is/docs/funnel) and [UTM reports](https://docs.umami.is/docs/utm)
- [Umami DPA](https://umami.is/dpa) and [subprocessors](https://umami.is/subprocessors)
- [Next.js Script guide](https://nextjs.org/docs/app/guides/scripts)
- [Czech DPA guidance on cookies](https://uoou.gov.cz/verejnost/qa-otazky-a-odpovedi/cookies)

This document is not legal advice and does not resolve the separate open questions concerning hosting, email delivery, and provider agreements.
