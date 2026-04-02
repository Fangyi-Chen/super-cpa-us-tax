# Plan 2: Domain Skills Implementation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create 4 domain-specific reference skills that extend the core tax pipeline with specialized knowledge for RSU, rental income, tax-loss harvesting, and cryptocurrency.

**Architecture:** Each domain skill provides trigger conditions, interview questions, data schema, calculation rules, and reference material. Core skills (tax-start, tax-import, tax-calculate) invoke domain skills when they detect relevant income types.

**Tech Stack:** Claude Code skills (Markdown only — no code)

**Spec:** `docs/superpowers/specs/2026-04-02-tax-agent-plugin-design.md`

---

## Task 1: `tax-domain-rsu` — RSU / ESPP Income

- [ ] Create `skills/tax-domain-rsu/SKILL.md` with content:

```markdown
---
name: tax-domain-rsu
description: RSU and ESPP income tax rules. Invoked by core skills when W2 box 14 contains RSU codes, 1099-B contains ESPP/RSU lots, or user mentions stock compensation during interview.
allowed-tools: [Read, Write, Edit, AskUserQuestion]
---

## Trigger Conditions

Core skills invoke this domain skill when:
- `tax-start`: User mentions RSU vesting, ESPP, stock compensation, equity awards, or "company stock"
- `tax-import`: W2 box 14 contains codes such as RSU, RSUS, ISO, NQ, NQSO, ESPP, or dollar amounts labeled as stock income; or 1099-B lots include "RSU", "ESPP", or "supplemental" in the description field
- `tax-calculate`: `domains.rsu` in tax-profile.json is non-null

## Interview Questions

Ask these questions when triggered. Ask 2-3 at a time; do not overwhelm.

1. "Did you have RSU shares vest in 2025? If so, roughly how many shares, and what was the company's stock price on each vesting date?"
2. "Do you participate in an Employee Stock Purchase Plan (ESPP)? If yes, did you sell any ESPP shares in 2025?"
3. "For any RSU shares you sold in 2025: did you sell them on the same day they vested (same-day sale), use shares to cover taxes (sell-to-cover), or hold them after vesting?"
4. "Check your W2 box 14 — do you see any amounts labeled RSU, ESPP, ISO, NQ, or similar? If yes, what is the label and dollar amount?"
5. "Did your company withhold taxes on RSU vesting using the 22% federal supplemental rate (or 37% if total supplemental income exceeded $1 million)?"
6. "For ESPP: what was the offering date price and the purchase date price? Was the discount 15% or another percentage?"
7. "Have you received a 1099-B for any stock sales? If yes, does it show the cost basis in Box 1e, or is it marked 'cost basis not reported to IRS'?"
8. "Did you exercise any stock options (ISO or NQ/NSO) in 2025?"

## Data Schema

Add under `domains.rsu` in `data/tax-profile.json`:

```json
{
  "domains": {
    "rsu": {
      "hasRSU": true,
      "hasESPP": false,
      "rsuLots": [
        {
          "vestDate": "2025-03-15",
          "sharesVested": 100,
          "fmvAtVest": 42.50,
          "sharesWithheldForTax": 30,
          "saleType": "sell-to-cover",
          "saleDate": "2025-03-15",
          "salePricePerShare": 42.48,
          "w2IncomeReported": 4250.00,
          "federalWithheld": 935.00,
          "supplementalRateUsed": 0.22
        }
      ],
      "esppLots": [
        {
          "offeringDate": "2025-01-01",
          "purchaseDate": "2025-06-30",
          "offeringDateFMV": 38.00,
          "purchaseDateFMV": 44.00,
          "purchasePriceActual": 32.30,
          "sharesPurchased": 50,
          "sharesSold": 50,
          "saleDate": "2025-09-10",
          "salePricePerShare": 46.00,
          "dispositionType": "disqualifying",
          "w2IncomeReported": 580.00
        }
      ],
      "isoExercises": [],
      "nsoExercises": []
    }
  }
}
```

Field notes:
- `saleType`: one of `same-day`, `sell-to-cover`, `hold`, `no-sale`
- `dispositionType` for ESPP: `qualifying` (held >2yr from offering + >1yr from purchase) or `disqualifying`
- `w2IncomeReported`: the dollar amount already included in W2 box 1 for this lot (prevents double-counting)

## Calculation Rules

### RSU Cost Basis
- **Rule:** Cost basis for RSU shares = FMV at vest date × shares received (after withholding).
- **W2 income already included:** The FMV at vest is already in W2 box 1. Do NOT add it again to income.
- **1099-B adjustment:** Brokers often report $0 or incorrect cost basis on 1099-B for RSU. Correct basis = FMV at vest per lot.
- **Form 8949:** Enter corrected basis; use code B (short-term, basis not reported) or E (long-term, basis not reported). Add "Adjusted basis per IRC §1012" in column f.

### RSU Holding Period
- Short-term: sold within 1 year of vest date → Schedule D / Form 8949, Box B or E rates apply
- Long-term: held >1 year from vest date → 0%, 15%, or 20% preferential rates

### ESPP — Qualifying Disposition
Requirements: held >2 years from offering date AND >1 year from purchase date.
- Ordinary income = lesser of: (a) actual gain, or (b) discount at offering date FMV × shares
- Formula: min(sale_price - purchase_price, offering_date_fmv × discount_rate) × shares
- Remaining gain (if any) = long-term capital gain
- Report ordinary income on Schedule 1 line 8 (not in W2 box 1 — employer may not have known)

### ESPP — Disqualifying Disposition
Requirements: sold before meeting qualifying disposition holding periods.
- Ordinary income = (FMV at purchase date - actual purchase price) × shares sold
- This amount MUST appear in W2 box 1 (employer is required to include it)
- Remaining gain/loss = short- or long-term capital gain on Form 8949

### Supplemental Withholding
- Flat 22% federal rate applies to supplemental wages up to $1,000,000
- Above $1,000,000 cumulative supplemental wages in the year: flat 37%
- Supplemental rate withholding often underpays for high earners in high tax brackets — flag underpayment risk

### ISO Exercise
- No regular income tax at exercise (for regular tax purposes)
- AMT preference item = (FMV at exercise - exercise price) × shares (note: AMT calculation out of scope for v1 — flag to user)
- At sale: if qualifying disposition, entire gain is long-term capital gain

### NSO/NQ Exercise
- Ordinary income at exercise = (FMV at exercise - exercise price) × shares exercised
- Must be included in W2 box 1
- Cost basis for future sale = FMV at exercise date

## References

Read `skills/tax-domain-rsu/references/rsu-tax-rules.md` for detailed IRS citations, Publication 525 rules, and Form 8949 instructions.
```

- [ ] Create `skills/tax-domain-rsu/references/rsu-tax-rules.md` with content:

```markdown
# RSU and ESPP Tax Rules — IRS Reference

## Governing Law and Publications

- **IRC §83** — Taxation of property transferred in connection with services. FMV at vesting is ordinary income.
- **IRC §421–424** — Statutory stock options (ISO and ESPP).
- **IRS Publication 525** — Taxable and Nontaxable Income (stock options and ESPPs).
- **IRS Publication 550** — Investment Income and Expenses (holding period, capital gains).
- **Form 8949 Instructions** — Reporting sales and dispositions of capital assets.
- **Rev. Rul. 2005-48** — FMV of publicly traded stock determined by average of high and low trading prices on vesting date.

## RSU Taxation at Vesting

### Ordinary Income Recognition (IRC §83(a))
When RSU shares vest, the employee recognizes ordinary income equal to:

  **Income = FMV at vest date × number of shares vested**

- This income is a "wage" and must appear in W2 Box 1 (wages), Box 3 (social security wages), and Box 5 (Medicare wages).
- Box 14 is used by employers to separately disclose the RSU income amount, though this is informational — the amount is already in Box 1.
- Federal income tax withholding applies at the 22% supplemental rate (or 37% above $1M).
- FICA (6.2% SS + 1.45% Medicare) also withheld on vest income.

### Sell-to-Cover Transactions
When shares are withheld to cover taxes:
- Shares withheld are treated as if sold at FMV on vest date.
- The sale typically results in a negligible gain/loss (sale price ≈ FMV used for withholding).
- 1099-B will be issued for withheld shares. Enter on Form 8949 with corrected cost basis = FMV at vest.

### Cost Basis After Vest
  **Basis per share = FMV at vest date**

Broker may report $0 or "N/A" for RSU basis on 1099-B (especially for shares vested before 2014 basis reporting rules). Taxpayer must supply correct basis.

### Holding Period
- Starts the day after vesting date.
- Short-term if sold within 12 months of vest. Long-term if held more than 12 months.

### Form 8949 Coding for RSU Sales
| Situation | Box | Column (f) Adjustment Code |
|-----------|-----|--------------------------|
| Short-term, basis NOT reported to IRS | B | None needed if basis corrected in col (e) |
| Long-term, basis NOT reported to IRS | E | None needed if basis corrected in col (e) |
| Wash sale adjustment | A or D | W |
| Basis was incorrectly reported | A, B, D, or E | B (incorrect basis reported to IRS) |

## ESPP Taxation

### Plan Requirements for Section 423 Treatment (IRC §423)
- Offered to all employees (with limited exclusions).
- Purchase price ≥ 85% of FMV (at offering date or purchase date, whichever is lower).
- Maximum $25,000 FMV of stock purchasable per year.
- Offering period ≤ 27 months.

### Qualifying Disposition
Holding requirements:
1. Shares held more than **2 years** from the offering date, AND
2. Shares held more than **1 year** from the purchase date.

Tax treatment:
- **Ordinary income** = lesser of:
  - (a) Sale price − Purchase price (actual gain), OR
  - (b) Offering date FMV × plan discount rate × shares
- **Capital gain** = any remaining gain beyond ordinary income amount.
- Capital gain is long-term (holding period satisfied by definition).
- Employer is NOT required to include qualifying disposition income in W2 — taxpayer reports on Schedule 1, Line 8.

### Disqualifying Disposition
Any sale that does not meet qualifying disposition holding periods.

Tax treatment:
- **Ordinary income** = (FMV on purchase date − actual purchase price) × shares sold
- **Capital gain/loss** = (Sale price − FMV on purchase date) × shares sold
- Short- or long-term depending on whether held >1 year from purchase date.
- Employer MUST include disqualifying disposition income in W2 Box 1 for the year of sale.

### Double-Count Risk
If employer includes ESPP income in W2 AND taxpayer also enters the full purchase price as basis on 1099-B, the income is counted once. If taxpayer enters $0 basis (as sometimes reported), they must adjust upward to actual purchase price to avoid double-counting the ordinary income portion.

## ISO Taxation (IRC §422)

### At Exercise (Regular Tax)
- No regular income tax recognized at exercise.
- For AMT purposes: preference item = (FMV at exercise − exercise price) × shares (Form 6251 — AMT calculation is out of scope for v1).

### At Sale — Qualifying Disposition
Requirements: held >2 years from grant date AND >1 year from exercise date.
- Entire gain = long-term capital gain.
- **Gain** = sale price − exercise price.
- No ordinary income; no FICA.

### At Sale — Disqualifying Disposition
- **Ordinary income** = lesser of: (FMV at exercise − exercise price) OR (sale price − exercise price).
- Remaining gain (if any) = short- or long-term capital gain.
- Employer reports in W2 Box 1 for year of sale.

## NSO/NQ Taxation (IRC §83)

### At Exercise
- **Ordinary income** = (FMV at exercise − exercise price) × shares.
- Subject to income tax withholding and FICA.
- Must appear in W2 Box 1.

### At Sale
- **Basis** = FMV at exercise date.
- **Gain** = sale price − FMV at exercise.
- Short- or long-term depending on holding period from exercise date.

## Supplemental Wage Withholding Rates (2025)

| Cumulative Supplemental Wages | Federal Withholding Rate |
|-------------------------------|--------------------------|
| Up to $1,000,000 | 22% flat |
| Above $1,000,000 | 37% flat |

Note: Employees in higher marginal brackets (e.g., 32%, 35%, 37%) may be underwithheld at the 22% rate. Flag this risk during calculation.

## Key IRS Forms

- **W2 Box 1:** Total wages including RSU/ESPP/NSO ordinary income.
- **W2 Box 12, Code V:** NSO income included in box 1 (employer may report here).
- **W2 Box 14:** Informational RSU/ESPP amounts (employer discretion; not a second tax).
- **Form 8949:** Report each lot sale (Part I for short-term, Part II for long-term).
- **Schedule D:** Summarize Form 8949 totals; apply capital gains tax rates.
- **Schedule 1, Line 8:** Qualifying ESPP ordinary income (not on W2).
```

- [ ] Commit:
  ```bash
  git add skills/tax-domain-rsu/
  git commit -m "feat: add tax-domain-rsu skill (RSU/ESPP income handling)"
  ```

---

## Task 2: `tax-domain-rental` — Rental Property / Schedule E

- [ ] Create `skills/tax-domain-rental/SKILL.md` with content:

```markdown
---
name: tax-domain-rental
description: Rental property and Schedule E tax rules. Invoked by core skills when the user reports rental income or ownership of income-producing real property.
allowed-tools: [Read, Write, Edit, AskUserQuestion]
---

## Trigger Conditions

Core skills invoke this domain skill when:
- `tax-start`: User mentions rental income, rental property, landlord, Airbnb, VRBO, tenants, Schedule E, or "I rent out a property/room/unit"
- `tax-import`: 1099-MISC or 1099-K shows rental income (box 1 or box 3)
- `tax-calculate`: `domains.rental` in tax-profile.json is non-null and non-empty array

## Interview Questions

Ask these questions for each rental property. If the user has multiple properties, repeat for each. Ask 2-3 at a time.

1. "What is the address of the rental property?"
2. "How many days was the property rented to tenants at a fair market rate in 2025? How many days did you or your family use it personally (including free rentals to family)?"
3. "What was the total rental income received in 2025 (before any expenses)?"
4. "Do you have a mortgage on this property? If yes, what was the mortgage interest paid in 2025? (Check your Form 1098.)"
5. "What were your operating expenses for this property in 2025? I'll go through each category: property taxes, insurance premiums, repairs and maintenance, HOA fees, utilities you paid, property management fees, advertising, and any other costs."
6. "What did you originally pay for this property (purchase price), and when did you buy it? We'll need this to calculate depreciation."
7. "Approximately what portion of the purchase price was the building itself vs. the land? (Land is not depreciable. If unsure, we can use the property tax assessment ratio.)"
8. "Did you make any capital improvements to the property in 2025 (e.g., new roof, HVAC, additions)? These are depreciated separately from the building."
9. "Did you actively participate in managing this rental (e.g., approving tenants, setting rents, authorizing repairs)? Or is it managed entirely by a professional management company?"
10. "Is your modified AGI above $100,000? (This affects the $25,000 passive loss allowance.)"

## Data Schema

Add under `domains.rental` in `data/tax-profile.json` (array to support multiple properties):

```json
{
  "domains": {
    "rental": [
      {
        "propertyId": "rental-1",
        "address": "123 Main St, Oakland CA 94601",
        "daysRented": 280,
        "daysPersonalUse": 0,
        "rentalIncome": 24000,
        "expenses": {
          "mortgageInterest": 8400,
          "propertyTaxes": 3200,
          "insurance": 1200,
          "repairs": 850,
          "hoaFees": 0,
          "utilitiesPaid": 0,
          "propertyManagement": 2400,
          "advertising": 150,
          "other": 200
        },
        "depreciation": {
          "purchaseDate": "2018-06-15",
          "totalCostBasis": 480000,
          "landValue": 96000,
          "buildingBasis": 384000,
          "priorDepreciationTaken": 48000,
          "currentYearDepreciation": null,
          "capitalImprovements2025": 0
        },
        "activeParticipation": true,
        "isMixedUse": false,
        "scheduleE": {
          "netIncomeLoss": null,
          "passiveLossAllowed": null,
          "passiveLossCarryforward": 0
        }
      }
    ]
  }
}
```

## Calculation Rules

### Step 1: Determine Property Classification
- **Pure rental** (personal use days ≤ 14 days AND ≤ 10% of rental days): Full Schedule E rules apply.
- **Mixed-use / vacation home** (personal use > 14 days AND > 10% of rental days): Allocate expenses by rental-day fraction; cannot deduct losses beyond rental income.
- **Primarily personal** (rented < 15 days): All rental income excluded from income (IRC §280A(g)); no deductions.

### Step 2: Calculate Annual Depreciation
Residential rental property uses straight-line depreciation over 27.5 years (IRC §168).

  **Annual depreciation = Building basis ÷ 27.5**

For the first and last year, use mid-month convention:
- Months in service in year 1 = 12 − month placed in service + 0.5
- First-year depreciation = (Building basis ÷ 27.5) × (months in service ÷ 12)

Capital improvements are depreciated separately over 27.5 years (residential) starting mid-month of the improvement month.

### Step 3: Compute Schedule E Net Income / Loss
```
Gross rental income
- Mortgage interest
- Property taxes
- Insurance
- Repairs and maintenance
- HOA fees
- Utilities (if paid by landlord)
- Property management fees
- Advertising
- Depreciation
- Other ordinary and necessary expenses
= Net income (or loss) — Schedule E, Line 26
```

Enter on Schedule E Part I. Net income flows to Form 1040 Schedule 1, Line 5.

### Step 4: Passive Activity Loss Rules (IRC §469)

Rental activities are **passive** by default. Passive losses can only offset passive income unless an exception applies.

**$25,000 Allowance Exception (Active Participation):**
- If taxpayer actively participated (not just invested — must make management decisions), up to $25,000 of rental loss can offset non-passive income.
- Phase-out: Allowance reduces by $0.50 for every $1 of MAGI above $100,000.
  - Formula: `max(0, $25,000 − 0.50 × max(0, MAGI − $100,000))`
  - Fully phased out at MAGI = $150,000.

**Real Estate Professional Exception (IRC §469(c)(7)):**
- Taxpayer qualifies if: (1) more than 50% of personal services performed in real property trades/businesses, AND (2) more than 750 hours per year in those activities.
- If qualified, rental losses are not passive and fully deductible — flag this to user if they mention significant real estate activity.

**Passive Loss Carryforward:**
- Any disallowed loss carries forward to future years.
- Track `passiveLossCarryforward` in the data schema.
- Carryforward losses are released when property is sold.

### Step 5: Safe Harbor for Small Landlords (Rev. Proc. 2019-38)
Rental activity qualifies for safe harbor treatment (as a trade or business for QBI deduction purposes) if:
- 250+ hours of rental services performed per year (or all rental activity is one property).
- Contemporaneous records maintained.
- Not a triple-net lease.
If safe harbor met, rental income may be eligible for the 20% QBI deduction (out of scope for v1 — flag to user).

## References

Read `skills/tax-domain-rental/references/schedule-e-rules.md` for detailed depreciation tables, Schedule E line-by-line instructions, and passive activity loss examples.
```

- [ ] Create `skills/tax-domain-rental/references/schedule-e-rules.md` with content:

```markdown
# Schedule E and Rental Property Tax Rules — IRS Reference

## Governing Law and Publications

- **IRC §61** — Gross income includes rents received.
- **IRC §168** — Accelerated cost recovery system (ACRS/MACRS); residential rental = 27.5 years straight-line.
- **IRC §280A** — Disallowance of certain expenses for vacation homes / mixed-use properties.
- **IRC §469** — Passive activity loss rules.
- **IRS Publication 527** — Residential Rental Property (the primary reference for landlords).
- **IRS Publication 946** — How to Depreciate Property (MACRS tables, mid-month convention).
- **Schedule E (Form 1040)** — Supplemental Income and Loss.

## Schedule E, Part I — Line-by-Line Reference

| Line | Description | Notes |
|------|-------------|-------|
| 1 | Property address and type | A=single family, B=multi-family, C=vacation/short-term, D=commercial, E=land, F=self-rental, G=QJV, H=other |
| 2 | Days rented at fair rental price | |
| 3 | Days of personal use | |
| 4 | Rents received | Gross amount |
| 5 | Advertising | |
| 6 | Auto and travel | Actual expense or IRS mileage rate |
| 7 | Cleaning and maintenance | |
| 8 | Commissions | |
| 9 | Insurance | |
| 10 | Legal and other professional fees | |
| 11 | Management fees | |
| 12 | Mortgage interest paid to banks (1098) | |
| 13 | Other interest | |
| 14 | Repairs | (Distinguish from improvements — improvements capitalized) |
| 15 | Supplies | |
| 16 | Taxes | Property taxes |
| 17 | Utilities | |
| 18 | Depreciation (from Form 4562) | First year or new property requires Form 4562 |
| 19 | Other | |
| 20 | Total expenses | Sum of lines 5-19 |
| 21 | Subtract line 20 from line 4 — income (or loss) | |
| 22 | Deductible rental loss (limited by passive activity rules) | |
| 23-26 | Passive activity loss limitations | |

Net income (or allowed loss) from all Schedule E properties flows to Schedule 1, Line 5.

## MACRS Depreciation — Residential Rental Property

- **Asset class:** Residential rental real property (27.5-year GDS life).
- **Method:** Straight-line.
- **Convention:** Mid-month (placed in service in the middle of the month, regardless of actual date).
- **Depreciation basis:** Total cost (purchase price + closing costs) MINUS land value.

### Mid-Month Convention Table — First Year Percentage (27.5 years)
Month placed in service and percentage of annual depreciation allowed in year 1:

| Month | Year 1 % of Annual |
|-------|-------------------|
| January | 11.5/12 = 95.83% |
| February | 10.5/12 = 87.50% |
| March | 9.5/12 = 79.17% |
| April | 8.5/12 = 70.83% |
| May | 7.5/12 = 62.50% |
| June | 6.5/12 = 54.17% |
| July | 5.5/12 = 45.83% |
| August | 4.5/12 = 37.50% |
| September | 3.5/12 = 29.17% |
| October | 2.5/12 = 20.83% |
| November | 1.5/12 = 12.50% |
| December | 0.5/12 = 4.17% |

Full years 2 through 27: 100% of annual depreciation (basis ÷ 27.5).
Year 28 (or 29): remaining fractional month.

### Land Value Determination
If the tax assessment separates land and improvement values, use the ratio:
  **Building basis = Total cost × (Assessed improvement value ÷ Total assessed value)**

If no assessment available, a commonly accepted default is 20-25% land for typical residential properties, though this varies greatly by location. Advise user to verify with county assessor.

## Repairs vs. Capital Improvements

### Repairs (Deduct in current year)
- Restore property to working condition without adding value or extending useful life.
- Examples: fixing a broken window, patching a roof, repainting, replacing a faucet.

### Capital Improvements (Capitalize and depreciate)
- Add value, extend useful life, or adapt property to a new use.
- Examples: new roof, new HVAC system, addition of a room, new appliances.
- Capitalize and depreciate over 27.5 years (residential).

### Safe Harbor for Small Taxpayers (Reg. §1.263(a)-3(h))
Taxpayers with unadjusted basis ≤ $1,000,000 per property may expense repairs/improvements if the lesser of $10,000 or 2% of unadjusted basis per year.

## Passive Activity Loss Rules — Examples

### Example 1: Loss within allowance
- Rental loss = $18,000
- MAGI = $85,000 (below $100,000 phase-out threshold)
- Active participation: Yes
- Allowance = $25,000 (full, since MAGI < $100,000)
- Deductible loss = $18,000 (entire loss allowed)

### Example 2: Partial phase-out
- Rental loss = $20,000
- MAGI = $120,000
- Active participation: Yes
- Phase-out reduction = 0.50 × ($120,000 − $100,000) = $10,000
- Allowance = $25,000 − $10,000 = $15,000
- Deductible loss = $15,000
- Carryforward = $5,000

### Example 3: Fully phased out
- Rental loss = $15,000
- MAGI = $155,000 (above $150,000)
- Allowance = $0
- Deductible loss = $0
- Carryforward = $15,000

## Recapture Upon Sale (Depreciation Recapture)

When rental property is sold, all prior depreciation taken (and allowed to be taken) is recaptured:
- **Section 1250 recapture:** Recaptured at a maximum rate of 25% for residential real property (unrecaptured §1250 gain).
- Recapture amount = total depreciation deducted over ownership period.
- This is reported on Schedule D (not as ordinary income on line 1z).
- Remaining gain above recapture = §1231 long-term capital gain.

Note: Depreciation recapture calculation is triggered when the property sale is entered. Track `priorDepreciationTaken` in the data schema for this purpose.

## Vacation Home / Mixed-Use Rules (IRC §280A)

If personal use days exceed the greater of 14 days or 10% of rental days:
- Allocate expenses between rental and personal use based on rental days ÷ total days used.
- Cannot deduct rental losses (net loss deduction disallowed).
- Deductions limited to rental income (no loss allowed below zero).
- Interest and taxes attributable to personal use deductible on Schedule A.

## QBI Deduction Safe Harbor (Rev. Proc. 2019-38)

Rental activity may qualify for 20% qualified business income deduction if:
- ≥250 hours of rental services per year (or single property treated as one enterprise).
- Contemporaneous records of hours kept.
- Not a triple-net lease arrangement.
- Separate from personal residence.
Note: QBI deduction is out of scope for v1 — flag eligibility to user.
```

- [ ] Commit:
  ```bash
  git add skills/tax-domain-rental/
  git commit -m "feat: add tax-domain-rental skill (Schedule E / rental property)"
  ```

---

## Task 3: `tax-domain-harvest` — Tax-Loss Harvesting / Wash Sale Rules

- [ ] Create `skills/tax-domain-harvest/SKILL.md` with content:

```markdown
---
name: tax-domain-harvest
description: Tax-loss harvesting and wash sale tax rules. Invoked by core skills when 1099-B contains wash sale adjustments or when unrealized losses are detected in the portfolio.
allowed-tools: [Read, Write, Edit, AskUserQuestion]
---

## Trigger Conditions

Core skills invoke this domain skill when:
- `tax-start`: User mentions tax-loss harvesting, wash sales, capital loss carryforward, or selling investments at a loss
- `tax-import`: 1099-B has any non-zero amount in Box 1g (wash sale loss disallowed); or 1099-B shows losses with Box 5 checked (non-covered security)
- `tax-calculate`: `domains.harvest` in tax-profile.json is non-null; or total capital losses on Form 8949 exceed $3,000

## Interview Questions

Ask these questions when triggered. Ask 2-3 at a time.

1. "Your 1099-B shows wash sale adjustments. This means some losses were disallowed because you bought the same or substantially identical security within 30 days before or after the sale. Did you intentionally harvest losses this year?"
2. "For any disallowed wash sale losses: did you rebuy the same stock or fund within 30 days after selling at a loss? Or did you buy it within 30 days before the loss sale?"
3. "Did you or your spouse hold the same security in any other account (IRA, 401k, other brokerage) when you sold at a loss? Wash sale rules apply across all accounts."
4. "Do you have any capital loss carryforward from prior years? (Check your 2024 Schedule D, line 14 — or Form 1040, line 6.)"
5. "Did you harvest losses by selling index funds and replacing them with similar-but-not-identical funds? For example, selling S&P 500 ETF (VOO) and buying Total Market ETF (VTI)?"
6. "Are any of your wash sale disallowances in a tax-advantaged account (IRA, Roth IRA)? If so, those losses are permanently lost — they do not get added to the replacement shares' basis."

## Data Schema

Add under `domains.harvest` in `data/tax-profile.json`:

```json
{
  "domains": {
    "harvest": {
      "hasWashSales": true,
      "priorYearCarryforward": {
        "shortTerm": 0,
        "longTerm": 2400
      },
      "washSaleLots": [
        {
          "security": "TSLA",
          "cusip": "88160R101",
          "saleDate": "2025-08-15",
          "saleProceed": 8200,
          "costBasis": 11500,
          "grossLoss": -3300,
          "washSaleDisallowed": 3300,
          "netLossAllowed": 0,
          "replacementSharesBuyDate": "2025-08-20",
          "replacementSharesBuyPrice": 235.40,
          "adjustedBasisOfReplacement": 11500,
          "inTaxAdvantagedAccount": false
        }
      ],
      "summary": {
        "totalShortTermGainLoss": null,
        "totalLongTermGainLoss": null,
        "netCapitalGainLoss": null,
        "allowableDeduction": null,
        "carryforwardToNextYear": {
          "shortTerm": null,
          "longTerm": null
        }
      }
    }
  }
}
```

## Calculation Rules

### Step 1: Identify Wash Sales on 1099-B
- 1099-B Box 1g contains the disallowed wash sale amount.
- Box 5 checked = noncovered security (may have additional unreported wash sales).
- Net loss for each lot = Proceeds − Cost Basis + Wash Sale Disallowance (from Box 1g)
- Enter disallowed amount in Form 8949, Column (g) with adjustment code "W".

### Step 2: Adjust Replacement Share Basis
For each disallowed wash sale loss:
  **Adjusted basis of replacement shares = Purchase price of replacement shares + Disallowed loss amount**

This preserves the economic loss — it is not permanently lost (unless replacement shares are in an IRA/Roth IRA, where the loss IS permanently lost).

### Step 3: Adjusted Holding Period
The holding period of replacement shares includes the holding period of the sold shares when a wash sale occurs.

### Step 4: Net Capital Gains/Losses
After all Form 8949 entries:
```
Total short-term gain/loss (Part I net) → Schedule D Line 7
Total long-term gain/loss (Part II net) → Schedule D Line 15
Prior year ST carryforward → Schedule D Line 6
Prior year LT carryforward → Schedule D Line 14

Net short-term capital gain/loss = Line 7 + Line 6
Net long-term capital gain/loss = Line 15 + Line 14

Combined net = Schedule D Line 16
```

### Step 5: $3,000 Net Capital Loss Deduction Limit
If Schedule D Line 21 shows a net loss:
- Maximum deduction against ordinary income = $3,000 ($1,500 if MFS).
- Enter on Form 1040 Schedule D Line 21, then Form 1040 Line 7.

### Step 6: Capital Loss Carryforward
Excess loss (above $3,000) carries forward to future tax years:
  **ST carryforward = net ST loss beyond $3,000 (excess ST loss)**
  **LT carryforward = net LT loss (after applying to any ST gain)**

Use Schedule D Worksheet in IRS instructions to determine carryforward amounts.
Carryforward retains its short-term or long-term character.

### Capital Gains Tax Rates (2025)
| Rate | Single | MFJ | HoH |
|------|--------|-----|-----|
| 0% | Up to $48,350 | Up to $96,700 | Up to $64,750 |
| 15% | $48,350–$533,400 | $96,700–$600,050 | $64,750–$566,700 |
| 20% | Above $533,400 | Above $600,050 | Above $566,700 |

Note: These thresholds apply to taxable income, not just capital gains. Short-term gains taxed at ordinary income rates.

## References

Read `skills/tax-domain-harvest/references/wash-sale-rules.md` for detailed wash sale examples, substantially identical security analysis, and multi-account wash sale scenarios.
```

- [ ] Create `skills/tax-domain-harvest/references/wash-sale-rules.md` with content:

```markdown
# Tax-Loss Harvesting and Wash Sale Rules — IRS Reference

## Governing Law and Publications

- **IRC §1091** — Wash sale rules: loss disallowed when substantially identical stock or securities purchased within 30-day window.
- **IRC §1212** — Capital loss carryforward rules.
- **IRC §1222** — Definitions of short-term and long-term capital gain/loss.
- **IRS Publication 550** — Investment Income and Expenses (wash sales, capital gains/losses).
- **Schedule D Instructions** — Capital gains and losses, carryforward worksheet.
- **Form 8949 Instructions** — Sales and other dispositions of capital assets.

## Wash Sale Rule — IRC §1091

### The Rule
A wash sale occurs when you sell stock or securities at a loss AND within the **61-day window** (30 days before AND 30 days after the sale date) you buy "substantially identical" stock or securities.

The disallowed loss is NOT permanently lost — it is added to the basis of the replacement shares.

### The 61-Day Window
```
[30 days before sale] ──── [Sale Date] ──── [30 days after sale]
                         ←────61 days────→
```
The sale date itself is included. Count exactly 30 calendar days in each direction.

### Substantially Identical Securities
**Clearly substantially identical (wash sale triggers):**
- Same stock (ticker symbol) in any account
- Same mutual fund (same CUSIP)
- Contracts or options to acquire the same stock
- Bonds that are otherwise identical differing only in interest rates if issued by same company at same time

**Generally NOT substantially identical (safe to swap for tax-loss harvesting):**
- Different company's stock in the same sector (e.g., sell Ford, buy GM)
- Different S&P 500 ETFs from different providers (e.g., VOO → IVV) — NOTE: IRS has not issued definitive guidance; many tax professionals consider these not substantially identical due to different portfolios, but this is not 100% settled
- Total US Market ETF vs. S&P 500 ETF (different index, different holdings)
- Short-term bond fund vs. intermediate-term bond fund
- Stock vs. the same company's bonds

**Gray area (proceed with caution):**
- Two S&P 500 index funds from different providers (VOO and SPY both track S&P 500) — risk of IRS challenge
- Options on the same underlying stock

### Multi-Account Wash Sale Rules
Wash sales apply across **all accounts you own**, including:
- All taxable brokerage accounts
- IRAs (traditional and Roth)
- 401(k) and other retirement plans
- Accounts owned by your spouse

**Critical IRA wash sale trap:** If you sell at a loss in a taxable account and buy the same security in an IRA within the wash sale window, the loss is **permanently disallowed**. It cannot be added to the IRA basis because IRAs do not track individual security basis.

## Form 8949 Reporting

### For Each Wash Sale Lot
Enter on Form 8949 (Part I for short-term, Part II for long-term):
- Column (d): Proceeds
- Column (e): Original cost basis
- Column (f): Adjustment code **W**
- Column (g): Amount of wash sale disallowance (positive number — this REDUCES the loss)
- Column (h): Net gain/loss = Proceeds − Basis + Wash Sale Disallowance

### Example Wash Sale Entry
```
Security: AMZN 100 shares
Sale date: 2025-09-10
Proceeds: $17,500
Basis: $22,000
Gross loss: ($4,500)
Wash sale disallowed (Box 1g): $4,500
Adjusted loss: $0
Replacement shares bought: 2025-09-25 (15 days later)
Adjusted basis of replacement shares: $15,000 (purchase price) + $4,500 = $19,500
```

## Basis Adjustment for Replacement Shares

### Formula
  **Adjusted basis = Actual purchase price + Disallowed wash sale loss**

### Holding Period Tacking
The holding period of the replacement shares INCLUDES the holding period of the sold shares for the portion of shares that triggered the wash sale.

### Example: Partial Wash Sale
Sold 100 shares at a loss. Bought back only 60 shares within 30 days.
- Wash sale applies to 60 out of 100 shares.
- 60 shares: loss disallowed; added to basis of 60 replacement shares.
- 40 shares: loss fully recognized.

## Capital Loss Carryforward

### Annual Deduction Limit
- Maximum net capital loss deductible per year: **$3,000** ($1,500 if Married Filing Separately).
- Apply net capital losses in this order: first offset capital gains, then up to $3,000 against ordinary income.

### Carryforward Character
Excess losses carry forward retaining their short-term or long-term character:
1. If net short-term loss > net long-term gain: carryforward = excess short-term loss (short-term character)
2. If net long-term loss > net short-term gain: carryforward = excess long-term loss (long-term character)
3. Both ST and LT losses: carryforward both amounts separately

### Schedule D Carryforward Worksheet
Use the Capital Loss Carryover Worksheet in the Schedule D instructions:
- Line 6: Prior year short-term carryforward (from 2024 Sch D, line 16)
- Line 14: Prior year long-term carryforward (from 2024 Sch D, line 21)
- After applying carryforwards: compute new carryforward on the worksheet

## Tax-Loss Harvesting Strategy Notes (for Calculation Context)

These notes inform how `tax-calculate` should interpret the harvest domain data:

1. **Offsetting gains:** ST losses first offset ST gains (taxed at ordinary rates); LT losses first offset LT gains. After netting within class, any remaining loss in one class offsets gains in the other class.

2. **Optimal harvesting:** ST losses are more valuable than LT losses because they offset income taxed at higher ordinary rates. Harvesting to offset ordinary income (up to $3,000) is always beneficial if the investor plans to hold the replacement position long-term.

3. **Low-basis position lock-in:** Tax-loss harvesting does not help with low-basis positions that would generate a gain; those require different strategies (donation, step-up at death) which are out of scope for v1.

4. **Qualified dividends interaction:** Net long-term capital losses reduce the amount of qualified dividends eligible for preferential rates. Flag this if the user has significant qualified dividends.
```

- [ ] Commit:
  ```bash
  git add skills/tax-domain-harvest/
  git commit -m "feat: add tax-domain-harvest skill (wash sale / tax-loss harvesting)"
  ```

---

## Task 4: `tax-domain-crypto` — Digital Assets / 1099-DA

- [ ] Create `skills/tax-domain-crypto/SKILL.md` with content:

```markdown
---
name: tax-domain-crypto
description: Digital asset and cryptocurrency tax rules. Invoked by core skills when a 1099-DA is detected during document import or when the user reports cryptocurrency transactions.
allowed-tools: [Read, Write, Edit, AskUserQuestion]
---

## Trigger Conditions

Core skills invoke this domain skill when:
- `tax-start`: User mentions cryptocurrency, Bitcoin, Ethereum, NFT, DeFi, staking rewards, mining income, USDC, stablecoins, or "digital assets"
- `tax-import`: A 1099-DA document is detected; or 1099-MISC box 3 (other income) is present from a known crypto exchange
- `tax-calculate`: `domains.crypto` in tax-profile.json is non-null

## Interview Questions

Ask these questions when triggered. Ask 2-3 at a time.

1. "Did you buy, sell, trade, or dispose of any cryptocurrency or other digital assets in 2025? This includes converting one crypto to another (e.g., ETH → BTC), using crypto to pay for goods/services, and receiving crypto as payment."
2. "Which exchanges or wallets did you use? Did you receive a 1099-DA from any of them? (Starting in 2025, most major U.S. exchanges are required to issue 1099-DA.)"
3. "What cost basis method do you want to use: FIFO (first-in, first-out), HIFO (highest-in, first-out), or Specific Identification? Note: you must elect Specific ID at the time of sale and maintain adequate records."
4. "Did you receive any staking rewards in 2025? If yes, approximately what was the fair market value of the rewards at the time you received them?"
5. "Did you do any cryptocurrency mining in 2025? If yes, what was the approximate fair market value of the mined coins when you received them?"
6. "Did you participate in any DeFi (decentralized finance) activities — such as providing liquidity, yield farming, lending, or borrowing — in 2025?"
7. "Did you receive any crypto as a gift, in an airdrop, or as compensation for services?"
8. "Did you hold any stablecoins (USDC, USDT, DAI, etc.)? Did you redeem, swap, or transfer them in 2025?"
9. "Did you buy, sell, or trade any NFTs (non-fungible tokens) in 2025?"
10. "Did you use any crypto-to-crypto swaps or bridging transactions (e.g., wrapped tokens, Layer 2 bridges)?"

## Data Schema

Add under `domains.crypto` in `data/tax-profile.json`:

```json
{
  "domains": {
    "crypto": {
      "has1099DA": true,
      "exchanges": ["Coinbase", "Kraken"],
      "costBasisMethod": "FIFO",
      "form1040DigitalAssetCheckbox": true,
      "salesAndDisposals": [
        {
          "asset": "BTC",
          "saleDate": "2025-07-14",
          "proceeds": 28500,
          "costBasis": 18200,
          "gainLoss": 10300,
          "holdingPeriod": "long",
          "reportingCategory": "box-H-noncovered",
          "source1099DA": true
        }
      ],
      "ordinaryIncome": {
        "stakingRewards": 840,
        "miningIncome": 0,
        "airdrops": 120,
        "cryptoCompensation": 0,
        "defiIncome": 200
      },
      "stablecoinTransactions": {
        "hasStablecoinActivity": true,
        "aggregatedGainLoss": 0,
        "note": "USDC redeemed at $1.00; basis $1.00 per coin — de minimis gain/loss"
      },
      "nftTransactions": [],
      "summary": {
        "totalShortTermGainLoss": null,
        "totalLongTermGainLoss": null,
        "totalOrdinaryIncome": null
      }
    }
  }
}
```

## Calculation Rules

### Rule 1: Form 1040 Digital Asset Checkbox
Every taxpayer who received, sold, exchanged, or otherwise disposed of digital assets must check "Yes" on Form 1040, page 1, digital asset question. This applies even if no gain or loss resulted. Check "No" only if the taxpayer merely held digital assets without any transactions.

### Rule 2: Capital Gains/Losses — Sales and Exchanges
Every sale, trade, or exchange of cryptocurrency is a taxable event.

For each transaction:
  **Gain/Loss = Proceeds − Adjusted Cost Basis**

- **Short-term** (held ≤ 1 year): taxed at ordinary income rates.
- **Long-term** (held > 1 year): taxed at preferential capital gains rates (0%, 15%, 20%).
- Report on Form 8949; summarize on Schedule D.

### Rule 3: Crypto-to-Crypto Swaps
Converting one cryptocurrency to another (e.g., ETH → SOL) is a taxable disposal of the first asset.
- Proceeds = FMV of asset received at time of swap.
- Basis in new asset = FMV at acquisition date.

### Rule 4: 1099-DA Form 8949 Box
- 1099-DA transactions are generally reported in **Form 8949 Box H** (long-term) or **Box E** (short-term) for "noncovered" securities.
- Box B (short-term covered) or Box E (long-term covered) if basis IS reported to IRS by the broker.
- Verify: if 1099-DA Box 1e (cost basis) is populated, use Box A or D (covered); if $0 or blank, use Box B or E (noncovered).

### Rule 5: Cost Basis Methods
- **FIFO (First-In, First-Out):** Default method if no election made. Oldest lots sold first.
- **HIFO (Highest-In, First-Out):** Sell highest-cost lots first; maximizes loss recognition (or minimizes gains). Requires adequate records.
- **Specific Identification:** Identify exact lot at time of sale. Must be elected contemporaneously with the sale and records maintained per IRS guidelines. Most flexible but most demanding.
- Once a method is chosen for a given asset, it should be applied consistently (no formal lock-in rule, but IRS scrutinizes switches that appear tax-motivated).

### Rule 6: Staking Rewards and Mining Income
Per Rev. Rul. 2023-14, staking rewards are includible in gross income at **FMV when received**.
- Report as ordinary income on Schedule 1, Line 8z (other income).
- Basis in the received tokens = FMV at time of receipt.
- Subsequent sale of staking tokens is a capital gain/loss event.
- Mining income: same treatment — ordinary income at FMV when received.

### Rule 7: Stablecoin Transactions
Stablecoins (USDC, USDT, DAI, etc.) are treated as property, not currency, for U.S. tax purposes.
- Disposal of stablecoins is a taxable event.
- In practice, stablecoins pegged to $1 typically result in $0 gain/loss if held at stable peg.
- If 1099-DA reports stablecoin transactions, report on Form 8949; gains/losses likely de minimis.
- Aggregation rule: For stablecoin-to-stablecoin swaps or redemptions that are entirely at $1.00, enter as a single aggregated line if gain/loss is $0 — see IRS Notice 2014-21 (still applicable to stablecoin context).

### Rule 8: Airdrops and Hard Forks
- Airdrops: ordinary income at FMV when received (regardless of whether actively claimed).
- Hard forks resulting in new tokens: ordinary income at FMV when the taxpayer has dominion and control over the new tokens.
- Report on Schedule 1, Line 8z.

### Rule 9: DeFi — Providing Liquidity
- Depositing tokens into a liquidity pool: potentially a taxable exchange if LP tokens received differ from deposited tokens (FMV determines proceeds).
- Receiving liquidity pool rewards: ordinary income at FMV when received.
- Removing liquidity: taxable disposal at FMV of tokens received.

### Rule 10: NFT Sales
- NFTs are property; sales are capital gain/loss events.
- Short-term or long-term based on holding period.
- Collectible NFTs may be subject to 28% collectibles capital gains rate if they qualify as "collectibles" under IRC §408(m) — IRS guidance pending; flag to user.

## References

Read `skills/tax-domain-crypto/references/digital-asset-rules.md` for IRS notices, Rev. Rulings, Form 8949 examples, and cost basis method comparisons.
```

- [ ] Create `skills/tax-domain-crypto/references/digital-asset-rules.md` with content:

```markdown
# Digital Asset Tax Rules — IRS Reference

## Governing Law and IRS Guidance

- **IRC §61** — Gross income from all sources, including virtual currency.
- **IRC §1001** — Gain/loss on sale or exchange of property.
- **IRC §1221/1222** — Capital asset definitions and holding periods.
- **IRS Notice 2014-21** — Virtual currency is property; general tax principles apply; mining income = ordinary income at FMV when received.
- **Rev. Rul. 2019-24** — Hard forks result in ordinary income when taxpayer has dominion and control over new tokens.
- **Rev. Rul. 2023-14** — Staking rewards are includible in gross income at FMV when received; overrides Jarrett v. United States (M.D. Tenn. 2022) which ruled staking rewards were not immediately taxable.
- **IRS FAQ on Virtual Currency (IRS.gov)** — Updated guidance on airdrops, DeFi, NFTs (check for updates as of filing date).
- **Form 1099-DA (introduced 2025)** — Digital Asset Proceeds from Broker Transactions.
- **Form 8949 Instructions** — Capital asset sales and exchanges.
- **IRS Publication 544** — Sales and Other Dispositions of Assets.

## Form 1040 Digital Asset Question

The digital asset question on Form 1040 page 1 must be answered by all taxpayers:

**"At any time during 2025, did you: (a) receive (as a reward, award, or payment for property or services); or (b) sell, exchange, or otherwise dispose of a digital asset (or a financial interest in a digital asset)?"**

- Answer **Yes** if any of the following occurred: received crypto as payment, staking/mining rewards, airdrops, sold/traded crypto, converted crypto to fiat, used crypto to purchase goods/services, gifted crypto.
- Answer **No** if you only HELD digital assets with zero transactions. Checking "No" falsely when transactions occurred is a compliance risk.

## 1099-DA — New for Tax Year 2025

### Background
The Infrastructure Investment and Jobs Act (2021) expanded broker reporting to include digital asset brokers. The IRS finalized regulations requiring brokers (centralized exchanges, hosted wallet providers, and certain DeFi protocols) to issue Form 1099-DA starting for transactions in calendar year 2025.

### Key Fields
| Box | Description |
|-----|-------------|
| 1a | Description of property (asset name/ticker) |
| 1b | Date acquired |
| 1c | Date sold or disposed |
| 1d | Proceeds |
| 1e | Cost or other basis |
| 1f | Accrued market discount |
| 1g | Wash sale loss disallowed |
| 2 | Short-term (Box 2a) or long-term (Box 2b) gain/loss indicator |
| 3 | If checked: basis NOT reported to IRS (noncovered) |
| 4 | Federal income tax withheld |
| 5 | Type of gain or loss |

### Reporting on Form 8949

| 1099-DA Situation | Form 8949 Box |
|-------------------|---------------|
| Short-term, basis reported to IRS | Box A |
| Short-term, basis NOT reported | Box B |
| Long-term, basis reported to IRS | Box D |
| Long-term, basis NOT reported | Box E |

For "basis not reported" entries: enter proceeds in column (d), YOUR calculated basis in column (e), and no adjustment needed in column (g) unless there is a wash sale or other adjustment.

## Cost Basis Methods — Comparison

### FIFO (First-In, First-Out)
- Default if no other election made.
- Oldest lots consumed first on any sale.
- Generally produces the largest gain (or smallest loss) because early crypto purchases often have low basis.
- Simple and defensible.

### HIFO (Highest-In, First-Out)
- Highest-cost lots consumed first.
- Minimizes current-year gain (or maximizes loss recognition).
- Requires detailed records of all purchases with dates and prices.
- No IRS rule explicitly authorizes HIFO by name, but Specific Identification (which allows HIFO-equivalent lot selection) is permitted.

### Specific Identification
- Taxpayer designates exact lots at time of sale.
- Requires contemporaneous documentation: date, amount, price of the specific lot being sold.
- Most tax-efficient but most record-keeping intensive.
- For centralized exchanges: many provide lot selection tools; export trade history as documentation.
- IRS Rev. Proc. 2024-28 provides a safe harbor for taxpayers who have been using a universal method and wish to transition to Specific ID, allowing them to allocate remaining basis to specific lots using a reasonable method.

### Consistency Requirement
Once a method is chosen, the IRS expects consistent application within an asset class. Switching methods to maximize current-year tax benefit (especially if the switch is retroactive) is not permitted.

## Staking and Mining Income — Rev. Rul. 2023-14

### Staking
- When a taxpayer receives staking rewards, they have ordinary income equal to the FMV of the tokens on the date of receipt.
- FMV = price of the token on a reputable exchange at the date/time of receipt (spot price).
- Basis in received tokens = FMV at receipt.
- Reporting: Schedule 1, Line 8z ("Other income" — describe as "Staking rewards").

### Mining
- Same treatment as staking: ordinary income at FMV when coins are received.
- If mining rises to the level of a trade or business (Schedule C), self-employment tax also applies. For occasional/hobby mining, report on Schedule 1.
- Mining equipment may be depreciable (out of scope for v1).

## DeFi Transaction Tax Analysis

### Liquidity Providing
1. **Deposit:** Depositing Token A and Token B into a pool and receiving LP tokens is likely a taxable exchange. Proceeds = FMV of LP tokens received. Gain/loss on disposed Token A and B.
2. **Fees/Rewards:** Periodic fees or reward tokens are ordinary income at FMV when received.
3. **Withdrawal:** Redeeming LP tokens for Token A and Token B is a taxable event. Proceeds = FMV of tokens received. Gain/loss on LP tokens.

Note: IRS has not issued specific DeFi guidance as of 2025; the above analysis follows Notice 2014-21 general property principles.

### Wrapped Tokens
Wrapping ETH → WETH is likely a taxable exchange per property rules (two different tokens). However, many practitioners treat wrapping/unwrapping as non-taxable on the theory that the assets are economically equivalent. Use caution and document the position taken.

### Bridges and Layer 2 Transfers
Moving native ETH to a Layer 2 (e.g., Arbitrum, Optimism) via a bridge where you receive a "bridged" version may be treated as an exchange. The IRS has not issued specific guidance. Document transactions and basis at time of bridge.

## NFT Tax Treatment

### Sales of NFTs
- NFTs are property; gain/loss = sale proceeds − basis.
- Basis = price paid to acquire (including gas fees, which are capitalized into basis).
- Holding period determines short-term or long-term.

### Collectibles Rate
- Under IRC §1(h)(4), "collectibles" gains are taxed at a maximum 28% rate (not the standard 20% long-term rate).
- Collectibles include: works of art, rugs, antiques, metals, gems, stamps, coins, and "any other tangible personal property that the IRS determines is a collectible."
- Whether NFTs constitute collectibles is unresolved. IRS Notice 2023-27 requested public comments. Until further guidance, many practitioners default to treating NFTs as capital assets subject to standard rates unless the underlying asset (e.g., digital art) is clearly a collectible. Flag this uncertainty to user.

## Stablecoin Tax Treatment

- Stablecoins are property under Notice 2014-21.
- Each transfer, swap, or redemption is a taxable event.
- In practice, USDC redeemed at $1.00 with a $1.00 basis yields $0 gain/loss.
- If a stablecoin depegs (e.g., LUNA/UST collapse), losses may be recognizable — either as capital loss on disposal, or potentially as worthless security (IRC §165) if the token becomes worthless.
- Stablecoin-to-stablecoin swaps (e.g., USDC → USDT) are taxable exchanges. If both are at $1.00 parity, the gain/loss is de minimis.

## Gifts of Cryptocurrency

### Recipient's Basis
- Carryover basis: recipient takes donor's basis.
- If FMV at date of gift < donor's basis: basis for loss = FMV at date of gift; basis for gain = donor's original basis.
- Recipient's holding period includes donor's holding period.

### Gift Tax (Donor)
- Gifts of crypto are subject to gift tax rules (annual exclusion $19,000 per recipient in 2025).
- Donor does NOT recognize gain at time of gift.

### Charitable Donation of Crypto
- Deduction = FMV at date of donation (for long-term capital gain property donated to a public charity, within AGI limits).
- No capital gain recognized by donor.
- File Form 8283 if donation > $500.
```

- [ ] Commit:
  ```bash
  git add skills/tax-domain-crypto/
  git commit -m "feat: add tax-domain-crypto skill (digital assets / 1099-DA)"
  ```

---

## Task 5: Symlinks and Final Commit

- [ ] Create symlinks in `.claude/skills/` so Claude Code can discover each domain skill:
  ```bash
  cd /path/to/taxAgent
  ln -s ../../skills/tax-domain-rsu .claude/skills/tax-domain-rsu
  ln -s ../../skills/tax-domain-rental .claude/skills/tax-domain-rental
  ln -s ../../skills/tax-domain-harvest .claude/skills/tax-domain-harvest
  ln -s ../../skills/tax-domain-crypto .claude/skills/tax-domain-crypto
  ```

- [ ] Verify all 4 symlinks resolve correctly:
  ```bash
  ls -la .claude/skills/ | grep tax-domain
  ```

- [ ] Run a quick smoke test — invoke each skill by name and confirm the SKILL.md loads:
  ```bash
  # Verify each SKILL.md exists and has the correct frontmatter name
  grep "^name:" skills/tax-domain-rsu/SKILL.md
  grep "^name:" skills/tax-domain-rental/SKILL.md
  grep "^name:" skills/tax-domain-harvest/SKILL.md
  grep "^name:" skills/tax-domain-crypto/SKILL.md
  ```

- [ ] Stage and commit all remaining files:
  ```bash
  git add .claude/skills/tax-domain-rsu \
          .claude/skills/tax-domain-rental \
          .claude/skills/tax-domain-harvest \
          .claude/skills/tax-domain-crypto
  git commit -m "chore: add .claude/skills symlinks for domain skills"
  ```

---

## Summary

| Task | Skill | Key Reference File |
|------|-------|--------------------|
| 1 | `tax-domain-rsu` | `references/rsu-tax-rules.md` |
| 2 | `tax-domain-rental` | `references/schedule-e-rules.md` |
| 3 | `tax-domain-harvest` | `references/wash-sale-rules.md` |
| 4 | `tax-domain-crypto` | `references/digital-asset-rules.md` |
| 5 | Symlinks | `.claude/skills/tax-domain-*` |

After all tasks complete, the domain skills are discoverable by Claude Code and will be invoked automatically by `tax-start`, `tax-import`, and `tax-calculate` when the corresponding income types are detected.
