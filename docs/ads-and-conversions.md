# A/B Advertising Variants and Conversion Evaluation

Prepared on 28 October 2026 for a developer recruitment assignment and later retained as portfolio documentation. Audit baseline: commit `653798ec439a4323217943a47aafa8676bfbff20`.

No advertisements have been launched and this document contains no campaign results. Both variants lead to the same landing page; the proposed experiment tests advertising language, not two different web applications. Both email offers are visible to everyone, but the current demo delivers only to pre-approved recipients. The calculator remains public.

The recipient allowlist prevents the current demo from measuring ordinary public conversions. The 5% target below is a future working hypothesis for a legally and operationally prepared public delivery flow, not a result or benchmark. Current privacy questions are described in the [technical privacy review](privacy-review.md).

## 1. Approved attribution contract

The approved links use only the existing input parameter `utm_content`. `campaignContext` in `src/lib/analytics.ts` reads its allowlisted value and adds `ad_variant` to named analytics events. `utm_content` is the URL input; `ad_variant` is a derived event property, not a supported URL parameter.

| New-entry URL | Analytics property | Status |
| --- | --- | --- |
| [Variant A](https://etf-kompas-zeta.vercel.app/?utm_content=ad_a) | `ad_variant=ad_a` | Supported |
| [Variant B](https://etf-kompas-zeta.vercel.app/?utm_content=ad_b) | `ad_variant=ad_b` | Supported |
| `/` without a recognized variant | `ad_variant=unknown` | Supported |

This behavior was verified against the actual controller with a mocked transport. No production request was sent. The URL parameter `ad_variant` is ignored and must not be used in campaign links.

## 2. Proposed advertising copy

The application targets Czech-speaking users, so the production ad copy remains in Czech.

### Variant A — Expense-ratio impact

- **Headline:** Stejný index. Různé poplatky. Kolik vás stojí rozdíl?
- **Text:** Dva ETF fondy mohou sledovat stejný index, ale mít rozdílné náklady. Spočítejte si, jak se mohou projevit za 20 let.
- **CTA:** Spočítat rozdíl zdarma
- **Rationale:** emphasizes the modeled financial difference caused by ETF expenses. It must not be presented as guaranteed savings or return.
- **Proposed audience:** Czech-speaking adult beginner retail investors who already recognize the term ETF and want to compare ongoing costs. This is an audience proposal, not proof of configured platform targeting.
- **URL:** `https://etf-kompas-zeta.vercel.app/?utm_content=ad_a`
- **Directional hypothesis:** Variant A may increase interest in the personalized report.

### Variant B — Beginner education

- **Headline:** Chcete investovat do ETF, ale nevíte, který vybrat?
- **Text:** Porovnejte čtyři známé ETF, pochopte rozdíly a získejte zdarma průvodce pro začátečníky.
- **CTA:** Porovnat ETF zdarma
- **Rationale:** emphasizes understandable comparison and education; the basic result requires no email.
- **Proposed audience:** Czech-speaking adult beginners who want to understand ETFs. A clean A/B comparison must use the same audience as Variant A.
- **URL:** `https://etf-kompas-zeta.vercel.app/?utm_content=ad_b`
- **Directional hypothesis:** Variant B may increase interest in the beginner guide.

Both links contain only `utm_content`; they do not identify a platform, medium, or campaign name. Do not infer Meta or paid traffic from these URLs. Other UTM values remain technical possibilities in the library, not part of the approved links.

## 3. Event coverage

All eight named events receive `ad_variant=ad_a/ad_b/unknown`. Nothing is sent without consent or outside the allowed production environment.

| Event | Meaning | Important limit |
| --- | --- | --- |
| `landing_view` | First measured entry to `/` | Occurs only after consent; not every arrival |
| `calculator_change` | Valid parameter change completed on blur | Sends the field name, never the value |
| `etf_change` | Real ETF selection change | Sends only allowlisted ticker and position |
| `form_view` | At least 10% of a form visible | Once per material and measured entry |
| `form_start` | First focus/pointer interaction | Interaction, not proof that an email was typed |
| `form_submit` | Valid POST attempt after client checks | `attempt=first/repeat` distinguishes attempts |
| `material_accepted` | HTTP 202 with `status=accepted` | Provider acceptance, not mailbox delivery or unique email |
| `form_error` | General validation or attempt error | Demo-recipient rejection is intentionally excluded |

`demo_not_allowed` creates neither `form_error` nor `material_accepted`; after consent, the real attempt remains `form_submit`.

Only form events contain `material`. A global `material=report/guide` filter must not remove shared introductory funnel steps. Payload allowlists exclude email addresses and hashes, investment values, portfolios, request UUIDs, idempotency keys, Turnstile tokens, and arbitrary query values. Umami still processes network metadata and derived identifiers; the data is not absolutely anonymous.

## 4. Funnel definition

The full behavior path is:

1. `landing_view`;
2. optional `calculator_change` or `etf_change`;
3. `form_view` for a material;
4. `form_start` for the same material;
5. `form_submit`;
6. `material_accepted`.

The **primary conversion funnel** is:

```text
landing_view → form_view → form_start → form_submit → material_accepted
```

Evaluate report and guide separately where per-step filtering is supported. Calculator interaction is optional because a visitor may use defaults or request the guide directly. A diagnostic funnel may insert `calculator_change`, but it must never be presented as all conversions.

The primary A/B metric is the proportion of measured report participants who progress from `landing_view` to at least one `material_accepted`, using the same period, attribution, window, and participant unit for numerator and denominator:

```text
CR_v = 100 × U_v(landing_view → material_accepted) / U_v(landing_view)
```

Here `v` is `ad_a` or `ad_b`, and `U` is the consistent unit supplied by the verified Umami report. Do not add report and guide percentages: one participant may request both. A raw event-count ratio is not a visitor conversion rate and may exceed 100% because of retries or repeat entries.

## 5. Dashboard verification limits

Umami documentation describes sequential funnels, UTM reports, and event properties. The Funnel Window limits time between adjacent steps, not necessarily the whole journey. A consistent 60-minute adjacent-step window is proposed.

The owner confirmed a Hobby account with Events, Funnels, and UTM sections, but this review did not access the authenticated account. Still unverified:

- filtering a funnel by `ad_variant`;
- applying `material` only to relevant steps;
- the deduplication unit;
- behavior across repeated entries.

If these filters are unavailable, present Events/Properties and UTM Views only as limited count breakdowns. Do not label separate tables as a verified sequential funnel or derive visitor conversion rates from incompatible totals.

## 6. Working conversion hypothesis

The proposed target is **5% of measured conversion-report participants reaching at least one `material_accepted`**. It is not an achieved result, external benchmark, forecast, or promise.

At 200 comparable measured participants per variant, 5% would equal 10 converting participants. This is arithmetic illustration only; it does not guarantee statistical power. A small or zero traffic sample must be reported as a limitation, never replaced with invented results.

Expected direction: A may produce a higher report conversion rate and B a higher guide conversion rate. No overall winner is predicted. Keep audience, placement, period, and budget equal, and randomize allocation if supported. Define duration and sample requirements before launch; do not stop after the first few conversions. Report numerator, denominator, percentage-point difference, and uncertainty. Without advertising cost data, cost per conversion and an economic winner cannot be determined.

## 7. Prioritized future hypotheses

These are separate future experiments and must not run simultaneously with the advertising-message test.

| Priority | Test | Primary metric | Hypothesis |
| --- | --- | --- | --- |
| P1 | Current headline vs. a shorter explanation of the same index and modeled expense difference | Overall `landing_view → material_accepted` rate | Faster mobile comprehension may improve conversion |
| P2 | Current report paragraph vs. a three-point list: annual breakdown, expenses, assumptions | Report `form_view → material_accepted` rate | A more concrete offer may improve report conversion |
| P3 | Current guide introduction vs. a readable preview of real guide content | Guide `form_view → material_accepted` rate | A preview may reduce uncertainty |

Monitor overall conversion, general errors, speed, and accessibility as guardrails. Improvement to one material is not automatically improvement to the whole project.

## 8. Consent and interpretation boundaries

Measurement includes only consenting, unblocked, technically recorded activity. Denial does not limit the application, and pre-consent actions are not backfilled. Permission lasts only in the current document; reload requires a new opt-in. Denial is remembered for at most 180 days.

Visit, unique visitor, pageview, custom `landing_view`, submission attempt, provider acceptance, and delivered email are different metrics. Umami uses derived identifiers, so one person may split into multiple records or multiple people may be merged. Consent and blocking rates may differ between variants. No measured values are reported in this document.

## Primary sources reviewed on 28 October 2026

- [Umami Funnel](https://docs.umami.is/docs/funnel)
- [Umami Event data](https://docs.umami.is/docs/event-data)
- [Umami UTM](https://docs.umami.is/docs/utm)
- [Umami metric definitions](https://docs.umami.is/docs/metric-definitions)

The unavailable `/docs/filtering` page was not used as evidence for any account-specific capability.
