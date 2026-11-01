# Technical Privacy Review

Initial review: 27 October 2026. Current demo-state summary: 28 October 2026. Public privacy-page wording was generalized on 1 November 2026.

This is a technical review of the implemented application and documented provider behavior. It is not legal advice or confirmation that every legal, contractual, and transfer requirement has been resolved.

## Current demo state

- **Delivery:** both email flows are restricted by the server-side `DEMO_ALLOWED_EMAILS` allowlist. It requires exactly two distinct valid addresses. Their values are not published and are never sent to the browser, analytics, or application diagnostics.
- **Rejected requests:** an entered address and related request data still pass through hosting and the application server. Input validation and Siteverify occur before the recipient check. A non-approved request is not sent to Resend and the PDF is not read.
- **Operational confirmation:** the owner confirmed delivery of both the HTML report and PDF guide to approved recipients and rejection of a non-approved address. No new agent-performed production integration test is claimed.
- **Purpose:** ETF Kompas is a public, personal, non-commercial demonstration created as a practical developer recruitment assignment and maintained as an independent portfolio project. No advertising campaign or marketing-contact collection is running.
- **Public wording:** the application refers to pre-approved test recipients without publishing their number or values. The original company name is not needed to describe current operation.

Technically verified delivery restrictions do not establish legal readiness. Calling the application a demo and publishing open questions do not resolve those questions.

## Controller and public contact

The controller is **Michal Růžička**, a natural person operating ETF Kompas as a personal non-commercial demo. The original assignment's commissioning party is not presented as the operator or controller of visitor data.

The owner designated `privacy@michalwebdeveloper.cz` as the public contact and reported successful two-way testing. The application receives the contact through the separate server variable `PRIVACY_CONTACT_EMAIL`; it never falls back to Resend sender credentials. Invalid or missing configuration displays an unresolved notice without changing either delivery flow.

The privacy page is prerendered. Changing the public contact or related public configuration requires a new build/deployment.

## Verified processing map

| Area | Implemented flow |
| --- | --- |
| Forms | Email address, material type, honeypot, Turnstile token; the report also submits selected ETFs and model inputs. No backend contact registration. |
| Report | Validates recipient and scenario, recalculates on the server, and passes the recipient plus HTML/text report to Resend. |
| Guide | Passes the approved recipient, HTML/text message, and a general PDF loaded from a fixed path to Resend. It does not send calculator inputs. |
| Turnstile | The browser communicates with Cloudflare while loading and verifying the widget. Server Siteverify receives the secret and response token, not the email or investment scenario. Cloudflare may still process the browser IP and technical signals. |
| Request identity | A random UUID represents one logical request in client memory. The server derives a material-specific HMAC idempotency key. It is not a visitor ID and is not sent to Umami or application diagnostics. |
| Storage | Runtime application code has no form database and does not write submitted contacts to files. The PDF is read only. Providers may retain their own data. |
| Diagnostics | Application logs contain only allowlisted delivery categories and optional provider status, never email addresses, tokens, UUIDs, content, or raw exceptions. Infrastructure logging was not fully audited. |
| Analytics/storage | Contact data is not stored in cookies, `localStorage`, or `sessionStorage`. Consent preferences use limited browser storage. Umami loads only after explicit opt-in. |
| Fonts | `next/font/google` prepares fonts at build time and the application serves them locally; the visitor page does not load a Google Fonts stylesheet. |
| Hosting | Vercel serves the page and server endpoints, so form and network data pass through its infrastructure. |

## Current open questions

The documented assessment of legal bases for one-time delivery, recipient checking, form protection, and operational diagnostics is not complete. The following also remain to be established for the actual accounts and current demo:

- applicable Vercel Hobby data-processing terms, processing regions, logs, and contractual role;
- applicable Resend terms, retention, open/click tracking, processing locations, and transfer safeguards;
- Cloudflare Turnstile account settings, pre-clearance, signal retention, and the division between processor and independent-controller activities;
- concrete retention periods or criteria for the allowlist, provider records, technical signals, and correspondence concerning data-subject requests;
- subprocessors, countries of processing, transfer mechanisms outside the EEA, and any required supplementary safeguards;
- complete production browser verification of network requests, cookies, storage, hosting additions, and provider tracking.

The public [Vercel DPA](https://vercel.com/legal/dpa) describes scope for Pro and Enterprise; applicability to the Hobby project has not been established. [Resend's DPA](https://resend.com/legal/dpa) states primary processing in the United States and addresses relevant Standard Contractual Clauses, but linking to a DPA is not an audit of the actual account or transfer. Resend's public documentation describes standard 30-day retention; the account-specific state and tracking configuration remain unverified.

Before ordinary public delivery, marketing campaigns, or contact acquisition, resolve the present open issues and reassess the broader processing, notices, abuse protection, and interpretation of conversions.

## Legal basis and consent checkbox

The code demonstrates a one-time request for a material but cannot itself determine every condition of the applicable legal basis.

If performance of a contract under Article 6(1)(b) GDPR is considered, the operator must establish the actual relationship and necessity of processing. If legitimate interests under Article 6(1)(f) are considered, document purpose, necessity, and balancing. If consent under Article 6(1)(a) is selected, it must be specific, informed, demonstrable, and withdrawable.

The application deliberately does not add a generic “I agree to the privacy policy” checkbox. Acknowledging a notice is not an independent legal basis, and a one-time request is not a marketing subscription. The privacy page and links do not create a recorded consent mechanism for delivery.

Analytics uses a separate explicit opt-in. It is not combined with material delivery, a newsletter, or an “ETF Watcher”. Refusing or withdrawing analytics does not limit calculator or form functionality.

## Cookies, storage, and analytics

Application code does not use contact data in cookies or browser storage. Analytics denial is stored for at most 180 days; permission remains only in current-document memory and must be granted again after reload. This conservative design prevents a historical stored grant from silently restoring measurement.

Before consent, no Umami tracker or history queue is created. Withdrawal closes the synchronous gate and discards pending analytics. Same-origin tabs receive only a constant denial signal where storage events or BroadcastChannel are available. No grant, campaign data, personal data, or visitor identifier is synchronized.

Umami receives only allowlisted events and campaign properties, never email addresses, investment values, UUIDs, or Turnstile tokens. The tracker and network still expose technical metadata such as IP address and User-Agent, and Umami derives identifiers. The application therefore does not claim complete anonymity.

The owner reported an EU Umami region and Hobby plan. Public information describes six months of production-data retention plus separate backup cycles. EU storage does not guarantee that CDN, support, security, and operational logs remain exclusively inside the EEA. The applicable agreement, subprocessors, transfers, and safeguards remain the controller's responsibility.

See [analytics.md](analytics.md) for the exact technical configuration and testing boundaries.

## Providers

### Resend

For an approved request, Resend receives the recipient, subject, HTML and text content, and, for the guide, a PDF attachment. The report contains the scenario and modeled results. Rejected demo recipients are not passed to Resend.

The operator must verify the account's applicable terms, DPA, retention, open/click tracking, subprocessors, and transfer mechanism. Application-level absence of a contact database does not mean that Resend retains no recipient or content data.

### Cloudflare Turnstile

Turnstile processes technical browser and network signals for bot protection. The server submits the response token for verification but does not attach the email or investment scenario. The operator must verify widget mode, pre-clearance, account terms, retention, and Cloudflare's stated purposes.

### Vercel

Vercel hosts both the public pages and server endpoints. It therefore processes requests containing form data and operational metadata. The applicable Hobby-plan contractual coverage, regions, logs, retention, and provider role remain open questions.

### Umami

Umami is loaded only after opt-in and receives a strictly filtered event payload. It still processes network metadata and derived identifiers. Consent for analytics does not repair missing contractual or transfer safeguards for Umami or any other provider.

## Retention and rights

The application creates no database of form contacts. The recipient allowlist remains in private server configuration. A logical request snapshot is held only in the open form's memory and is lost on reload or closure. Application diagnostics exclude request contents and identifiers, but provider and infrastructure retention remains separate.

The operator must define real retention periods or criteria and deletion procedures for provider data, operational logs, the test-recipient list, and correspondence concerning rights. The public privacy page explains rights of access, correction, deletion, restriction, objection, portability where applicable, withdrawal where consent applies, and complaint to the Czech Office for Personal Data Protection.

## Required owner follow-up

1. Keep `PRIVACY_CONTACT_EMAIL` configured in production and maintain a real procedure for handling requests.
2. Determine and document the legal bases for delivery, recipient restriction, and form protection.
3. Verify the actual Resend terms, DPA, retention, tracking, subprocessors, and transfers.
4. Verify Cloudflare Turnstile settings, pre-clearance, retention, and contractual roles.
5. Confirm the Vercel project plan, applicable agreement, regions, logs, and retention; choose a suitable plan or provider if Hobby coverage is insufficient.
6. Identify relevant subprocessors, processing countries, and safeguards for transfers outside the EEA.
7. Define retention and deletion criteria for all provider and operational data.
8. Audit production cookies, storage, network requests, widgets, hosting additions, and email tracking after material changes.
9. Update the public privacy-page date whenever the processing description changes materially.

Until these points are resolved, the project should remain a restricted non-commercial demo rather than be represented as legally production-ready for public financial or marketing use.

## Primary sources

Reviewed on 27 October 2026:

- [EDPB: lawful processing](https://www.edpb.europa.eu/sme/be-compliant/process-personal-data-lawfully_en) and [individual rights](https://www.edpb.europa.eu/sme/be-compliant/respect-individuals-rights_en)
- [Czech DPA: cookies and legal-basis distinction](https://uoou.gov.cz/verejnost/qa-otazky-a-odpovedi/cookies)
- [Resend DPA](https://resend.com/legal/dpa) and [retention documentation](https://resend.com/docs/knowledge-base/account-quotas-and-limits#data-retention)
- [Cloudflare Turnstile Privacy Addendum](https://www.cloudflare.com/turnstile-privacy-policy/) and [pre-clearance documentation](https://developers.cloudflare.com/turnstile/additional-configuration/hostname-management/pre-clearance/)
- [Vercel DPA](https://vercel.com/legal/dpa) and [Privacy Notice](https://vercel.com/legal/privacy-notice)

Public provider documents are references, not confirmation of the agreements or settings attached to a specific account.
