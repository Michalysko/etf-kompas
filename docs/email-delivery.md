# One-Time Materials via Resend

## Current demo operation

The application is deployed on [Vercel Hobby](https://etf-kompas-zeta.vercel.app/). The calculator is public, while both email materials are restricted by the server-side `DEMO_ALLOWED_EMAILS` allowlist. It must contain exactly two distinct valid addresses separated by a comma, with no wildcards. Their values are never published.

Missing or invalid configuration disables delivery. After input validation and Siteverify, the handler checks the recipient before preparing the material, reading the PDF, or calling Resend. A non-approved address receives HTTP `403` with public code `demo_not_allowed`; it is not passed to Resend and is not written to application logs.

The owner manually confirmed delivery of both the HTML report and PDF guide to approved recipients and rejection of a non-approved address. This is operational confirmation, not an automated production integration test. Open contractual and privacy questions are documented in the [technical privacy review](privacy-review.md).

## API contract

The application exposes only:

- `POST /api/materials/report`
- `POST /api/materials/guide`

Requests must use JSON and may contain at most **8,192 actual bytes**. Both the declared `Content-Length` and streamed byte count are checked. The MIME type must be `application/json`, and responses are not cached.

Shared fields are `requestId` (lowercase UUID v4), `email`, `website` (an empty honeypot), and `turnstileToken`. The report additionally requires `scenario` with `firstTicker`, `secondTicker`, `monthlyContribution`, `years`, `grossAnnualReturnPercent`, and `initialInvestment: 0`.

Only known tickers with verified fees are accepted. Unknown fields, client-calculated results, custom fees, numeric strings, and invalid bounds are rejected. The server validates the scenario and recalculates it with `compareInvestments`; it never trusts results supplied by the client.

The report email contains the scenario, both expense ratios, total contributions, modeled fund values, the absolute final-value difference with context, and annual rows from year 0 through `Y`. It includes methodology, limitations, source links, and verification dates. Values are rounded only for display. A responsive HTML email and plain-text alternative are sent; the report is not a PDF.

The guide is a static seven-page PDF at `output/pdf/etf-od-nuly.pdf`, outside public routes. The route reads it from a fixed path and attaches it as base64 `application/pdf`. The application limit is 2 MiB; the committed file is 128,205 bytes. Its header and size are checked. `outputFileTracingIncludes` ensures that Next.js includes it in the server function. The source repository makes the file inspectable, but the deployed application exposes no anonymous download URL.

### Response status

| Status | Meaning |
| --- | --- |
| `202` | Resend returned a valid acceptance containing an ID |
| `400` | Invalid request or input |
| `403` | Origin, Turnstile, or demo-recipient rejection |
| `405` | Unsupported method |
| `413` | Request too large |
| `415` | Unsupported MIME type |
| `429` | Provider throttling |
| `503` | Missing configuration or unconfirmed delivery request |

The client receives generic messages only, never internal exceptions, provider responses, or key identifiers. The API does not accept attachment URLs, subjects, HTML, or sender addresses from the client.

## Protection without a contact database

The approved protection model is **Turnstile + honeypot + Resend idempotency**, without a distributed per-IP or per-email limiter.

- The request origin and URL must match `APP_ORIGIN`. This is not authentication; Turnstile remains mandatory.
- Turnstile is always verified server-side. `success`, the exact hostname, and action `report` or `guide` must match. Tokens are single-use and expire after five minutes. Verification failure fails closed.
- The client resets the widget after every attempt and requires a fresh token. The honeypot is excluded from normal keyboard and assistive-technology interaction.
- Each intentional submission creates a random UUID for one logical request. A material-specific HMAC-SHA256 Resend idempotency key is derived server-side. Report and guide namespaces cannot collide.
- A retry of the same uncertain request preserves the UUID and original recipient/scenario. A new intentional request receives a new UUID. Resend deduplicates keys for 24 hours.
- The application has no contact database, cron job, newsletter, or “ETF Watcher”. It does not collect IP addresses for rate limiting or log email addresses, request bodies, tokens, UUIDs, idempotency keys, or raw provider errors.

Turnstile reduces automated abuse but does not provide a hard sending budget. Resend quotas and throttling still apply. A larger public service would need separately approved distributed rate protection. Protection failure must never enable an unprotected fallback.

## Request lifecycle and retries

The client stores the current request identity and a snapshot of its recipient/scenario only in component memory. It does not use a database, `localStorage`, or `sessionStorage`.

After an uncertain outcome, the address is locked and the UI offers “Retry the same request”. Calculator changes do not rewrite the snapshot. Every retry requires a new Turnstile token and passes all server checks again. HTTP `202` with `status=accepted` closes the request; the next deliberate submission creates a new UUID.

Reloading, closing, unmounting, or opening another tab loses the in-memory identity. The UI therefore warns against reloading after an uncertain outcome. Without durable shared storage, deduplication cannot be guaranteed across tabs or after reload. Client retries are blocked after 23 hours as a safety margin against Resend's 24-hour idempotency window.

The public code `input_rejected` is returned only for the application's own pre-provider input rejection. It permits correction only if no earlier uncertain result exists. Uncertainty is monotonic: a later validation response cannot erase an earlier timeout or network ambiguity.

Recognized Resend conflicts are mapped conservatively:

- `concurrent_idempotent_requests` → public `request_pending`;
- `invalid_idempotent_request` → public `request_conflict`, which blocks further retries of that identity.

Unknown conflicts remain generic HTTP `503`. Provider response reading distinguishes timeout, abort, stream failure, invalid JSON, and unexpected response structure without exposing provider payloads.

## Safe diagnostics

Server diagnostics use a fixed `email_delivery_failure` marker and allowlisted fields only: `material`, `phase`, `code`, and optionally numeric provider status. Phases include configuration, validation, Turnstile, content, PDF, and Resend. No personal data, tokens, UUIDs, idempotency keys, message content, or raw exceptions are logged by application code. Infrastructure providers may keep their own logs; their retention must be assessed separately.

## Configuration and operation

Delivery requires:

```text
RESEND_API_KEY
RESEND_FROM_EMAIL
TURNSTILE_SECRET_KEY
NEXT_PUBLIC_TURNSTILE_SITE_KEY
APP_ORIGIN
DEMO_ALLOWED_EMAILS
```

`PRIVACY_CONTACT_EMAIL` is a separate public contact and is never used as recipient or sender configuration. Never create `NEXT_PUBLIC_RESEND_API_KEY`; only the Turnstile site key is public. The page receives only an enabled flag and the public site key.

The sender validation currently requires the verified domain `etf.michalwebdeveloper.cz`. Local `APP_ORIGIN` must exactly match the browser origin; `localhost` and `127.0.0.1` are not interchangeable. Production requires an HTTPS origin. Environment changes require a new build or restart because page availability and the public site key are not a live configuration service.

Timeouts are 8 seconds for Siteverify, 12 seconds for Resend, and 30 seconds for the client. Route Handlers declare a 30-second `maxDuration` where supported. There is no automatic Resend retry. Success means provider acceptance, not mailbox delivery; a later bounce remains possible and webhooks are not implemented.

Submitting an address requests one material only. It does not grant marketing consent or create a reusable contact. The public privacy page describes Resend and Turnstile processing. The legal bases, retention, provider agreements, and transfer safeguards for parts of the current demo remain open; transparency does not resolve them.

## Tests and their limits

`npm test` uses fake configuration and mocked provider functions. It does not read `.env.local`, use real keys, or send email. Coverage includes validation, independent server recalculation, report content, annual rows, PDF attachment checks, stream limits, origin checks, honeypot behavior, token verification contracts, safe errors, retries, and idempotency.

The tests do not verify a live Resend account, real delivery, rendering in every email client, or the deployed Turnstile account. Any additional real email test requires explicit authorization and an approved recipient. Never send a real POST merely because credentials are available.

## Conditions for broader use

Before public delivery or a real advertising campaign, the operator must resolve:

- the legal bases for one-time delivery, recipient checks, and form protection;
- applicable provider terms and data-processing agreements for Resend, Cloudflare, Vercel, and consented Umami analytics;
- retention periods, deletion procedures, subprocessors, processing locations, and safeguards for transfers outside the EEA;
- a real procedure for data-subject requests;
- production browser verification of storage, Turnstile settings, hosting additions, and email tracking;
- stronger abuse controls if the service is opened beyond a small allowlisted demo.

The controller is Michal Růžička. The public privacy contact is configured separately through `PRIVACY_CONTACT_EMAIL`. The page is prerendered, so changing that contact requires a new build/deployment.

## Primary technical sources

Reviewed on 26 October 2026:

- [Resend Send Email](https://resend.com/docs/api-reference/emails/send-email)
- [Resend idempotency](https://resend.com/docs/dashboard/emails/idempotency-keys)
- [Turnstile server-side validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)
- [Turnstile explicit rendering](https://developers.cloudflare.com/turnstile/get-started/client-side-rendering/)
- [Turnstile testing](https://developers.cloudflare.com/turnstile/troubleshooting/testing/)
- Privacy information: [Resend](https://resend.com/legal/privacy-policy), [Cloudflare](https://www.cloudflare.com/privacypolicy/)
