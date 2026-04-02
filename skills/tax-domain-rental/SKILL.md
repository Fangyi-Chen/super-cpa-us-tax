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
