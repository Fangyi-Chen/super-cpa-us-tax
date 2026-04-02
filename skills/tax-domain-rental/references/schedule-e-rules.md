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
