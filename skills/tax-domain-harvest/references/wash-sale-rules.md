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
