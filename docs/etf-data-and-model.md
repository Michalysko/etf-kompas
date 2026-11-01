# ETF Kompas: Data and Calculation Model

## Verified dataset

Manual verification date: **26 October 2026**, Europe/Prague. This is a maintained snapshot, not a live market-data feed. The data is stored in `src/data/etfs.ts`. Expense ratios are expressed as percentages, so `0.03` means **0.03% per year**. All four funds currently track the S&P 500 Index and trade on NYSE Arca. The verification date is not necessarily the effective date of the fee.

| Ticker | Full name | Annual expense ratio | Primary source | Source date |
| --- | --- | --- | --- | --- |
| SPY | State Street® SPDR® S&P 500® ETF Trust | 0.0945% (Gross Expense Ratio) | [State Street](https://www.ssga.com/us/en/individual/etfs/state-street-spdr-sp-500-etf-trust-spy) | Fund Information, 26 October 2026 |
| SPYM | State Street® SPDR® Portfolio S&P 500® ETF | 0.02% (Gross Expense Ratio) | [State Street](https://www.ssga.com/us/en/individual/etfs/state-street-spdr-portfolio-sp-500-etf-spym) | Fund Information, 26 October 2026 |
| VOO | Vanguard S&P 500 ETF | 0.03% (Expense ratio) | [Vanguard factsheet](https://workplace.vanguard.com/assets/corp/fund_communications/pdf_publish/us-products/fact-sheet/F0968.pdf) | 30 September 2026 |
| IVV | iShares Core S&P 500 ETF | 0.03% (Expense Ratio) | [iShares](https://www.ishares.com/us/products/239726/ishares-core-sp-500-etf-ivv) | Current prospectus; the product page does not state the exact effective date |

Management roles are stored explicitly because the legal structures differ. State Street Global Advisors Trust Company is listed as trustee for SPY, while SSGA Funds Management, Inc. is the Investment Manager for SPYM. Vanguard Capital Management is listed in the Management section of the [Vanguard product page](https://advisors.vanguard.com/investments/products/voo/vanguard-sp-500-etf?fromSearch=true&source=autosuggest). BlackRock Fund Advisors is the investment adviser to IVV according to the [BlackRock prospectus](https://www.blackrock.com/us/individual/literature/prospectus/p-ishares-trust-us-sp-cap-non-sector-3-31.pdf) dated 31 July 2026. Issuer branding is stored separately.

SPYM previously traded as SPLG. An official [State Street notice dated 22 October 2025](https://www.ssga.com/library-content/products/fund-docs/etfs/us/information-schedules/ap-notice/notice-to-aps-ticker-fund-name-and-benchmark-index-name-changes-10-31-25.pdf) states that the change took effect on 31 October 2025. The notice was visible in indexed issuer-domain search output, although the PDF returned 404 during the review. The live SPYM product page confirms the current ticker; the historical document may have moved.

`sourceAsOf: null` for IVV means that the exact fee effective date is unknown while the verification date is known. If a future review cannot verify a fee, set `verificationStatus: "unverified"` and `annualExpenseRatioPercent: null`. Never substitute zero or use an unverified fee in calculations. Automated tests do not verify whether an issuer website is still current; the dataset requires manual review before publication of an update.

## Recurring-investment model

`modelInvestment` is a pure synchronous TypeScript function with no React, I/O, or ETF-catalog dependency. It starts with a zero balance. During each month, modeled growth is applied first and the contribution is added **at the end of the month**. All amounts use one user-selected monetary unit; the function performs no currency conversion.

For monthly contribution `P`, horizon `Y` years, gross annual return `G` percent, and annual expense ratio `F` percent:

```text
g = G / 100; f = F / 100
q = ((1 + g) * (1 - f))^(1/12)
V[0] = 0
V[m] = V[m-1] * q + P, m = 1 ... 12Y
total contributions = P * 12Y
```

The implementation uses the numerically equivalent expression `exp((log1p(g) + log1p(-f)) / 12)`. The fee is modeled as a proportional reduction of the growth factor, not as simple subtraction `g - f`. This approximates the continuous effect of fund costs on NAV; it is not a simulation of fund accounting or an invoiced fee. Historical net ETF returns are deliberately not used, which avoids counting expenses twice.

The result contains `totalContributions`, `finalValue`, and `annualProgress`, including year 0 and the end of every subsequent year. Each annual point contains `year`, `totalContributions`, and `value`. Internal values are not rounded and use IEEE 754 `number`; rounding belongs to presentation code. This is not accounting software with sub-cent precision.

`compareInvestments(sharedInputs, [feeA, feeB])` applies one shared gross return, contribution, and horizon. It returns `first`, `second`, and `finalValueDifference = second.finalValue - first.finalValue`. The difference isolates the modeled compound effect of expenses. It is neither the sum of fees actually charged nor a prediction of the performance gap between specific funds.

### Validation and limits

- Inputs must be finite numbers; strings are not coerced. Invalid values raise `RangeError`.
- Monthly contribution: 0 to 1,000,000,000. Horizon: integer from 0 to 100 years. A zero horizon returns only the initial point.
- Gross annual return: greater than -100% and no more than 100%. Expense ratio: from 0% to less than 100%. Negative returns are supported. These bounds protect the application; they are not market forecasts.
- Infinite results or values above `Number.MAX_SAFE_INTEGER` are rejected. The maximum calculation contains 1,200 monthly iterations.
- The model assumes a constant return and fee, full reinvestment, fractional shares, and zero initial investment. It does not model volatility, actual return sequences, tracking difference, changing expenses, or different cash-management policies.
- **Returns are not guaranteed and losses are possible.** Taxes, inflation, currency movements, broker commissions, spreads, and other transaction costs are excluded. The result is not investment advice.

## Availability to Czech retail investors: PRIIPs/KID

Listing on NYSE Arca does not automatically make these US ETFs available to Czech retail investors. PRIIPs rules generally require a Key Information Document for retail distribution. Many US ETFs do not provide one, so European brokers commonly block purchases. A US factsheet or prospectus does not replace a KID.

This is not a blanket prohibition on holding US ETFs. Availability depends on the documentation, the broker, and the client's classification. ETF Kompas does not perform a broker-by-broker eligibility or KID check for these funds.

Sources reviewed on 26 October 2026: [European Commission — PRIIPs/KID](https://finance.ec.europa.eu/consumer-finance-and-payments/retail-financial-services/key-information-documents-packaged-retail-and-insurance-based-investment-products-priips_en), [ESMA consolidated Q&A, Part I](https://www.esma.europa.eu/sites/default/files/2023-05/JC_2023_22_-_Consolidated_JC_PRIIPs_Q_As.pdf), and [LYNX — KID and US ETF restrictions](https://www.lynxbroker.com/service/products-and-regulations/regulations/key-information-document/). Investors must confirm current conditions with their broker before a transaction.

## Verification commands

```sh
npm test
npm run lint
npm run build
```

Tests use `node:test` and `node:assert/strict`. Direct TypeScript execution requires Node.js 24.12 or newer; the verified environment used Node.js 24.22.0. `allowImportingTsExtensions` permits type-checking explicit `.ts` imports while the existing `noEmit` setting remains enabled. Node does not type-check the source by itself; the production build performs TypeScript validation. See the [Node.js TypeScript documentation](https://nodejs.org/api/typescript.html).

Coverage includes zero returns and fees, identical funds, zero contribution and horizon, 1/5/30/100-year horizons, negative returns, annual progress, percentage units, invalid inputs, numerical bounds, and an independent closed-form annuity calculation including a known 1% monthly-growth case.
