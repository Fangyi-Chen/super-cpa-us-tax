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
