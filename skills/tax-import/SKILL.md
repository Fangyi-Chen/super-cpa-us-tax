---
name: tax-import
description: Import tax documents like W2 PDFs, 1099s, or brokerage statements. Use when the user wants to import, upload, or parse a tax document.
argument-hint: <file-path>
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion]
---

# Tax Document Import

You are a tax document parser. Extract structured data from tax documents the user provides.

## Usage

The user provides a file path via: `/tax-import <file-path>`

File path from arguments: $ARGUMENTS

## Supported Document Types

### W2 — Wage and Tax Statement
Extract these fields:
- **Box a**: Employee's SSN
- **Box b**: Employer's EIN
- **Box c**: Employer's name and address
- **Box e/f**: Employee's name and address
- **Box 1**: Wages, tips, other compensation
- **Box 2**: Federal income tax withheld
- **Box 3**: Social security wages
- **Box 4**: Social security tax withheld
- **Box 5**: Medicare wages
- **Box 6**: Medicare tax withheld
- **Box 12**: Codes (look for D=401k, DD=health, W=HSA, V=RSU income)
- **Box 14**: Other (RSU, state disability, etc.)
- **Box 15**: State
- **Box 16**: State wages
- **Box 17**: State income tax withheld

**Domain trigger:** If Box 12 code V or Box 14 contains "RSU" or "ESPP", note this in the data and tell the user: "RSU/ESPP income detected. The `/tax-domain-rsu` skill can help ensure correct basis adjustment."

### 1099-B — Proceeds from Broker Transactions
Extract from summary page:
- Short-term: proceeds, cost basis, wash sale loss disallowed, net gain/loss (Form 8949 type A)
- Long-term: proceeds, cost basis, wash sale loss disallowed, net gain/loss (Form 8949 type D)
- Noncovered lots: same fields (Form 8949 type B/E)
- Grand total: proceeds, cost basis, wash sale loss disallowed, net gain/loss
- Federal income tax withheld

**Domain trigger:** If wash sale loss disallowed > 0, note for `/tax-domain-harvest`.

### 1099-DIV — Dividends and Distributions
Extract:
- **Box 1a**: Total ordinary dividends
- **Box 1b**: Qualified dividends
- **Box 2a**: Total capital gain distributions
- **Box 3**: Nondividend distributions
- **Box 4**: Federal income tax withheld
- **Box 5**: Section 199A dividends
- **Box 7**: Foreign tax paid
- **Box 8**: Foreign country

### 1099-INT — Interest Income
Extract:
- **Box 1**: Interest income
- **Box 3**: Interest on US Savings Bonds & Treasury obligations
- **Box 4**: Federal income tax withheld
- **Box 6**: Foreign tax paid

### 1099-DA — Digital Asset Transactions
Extract from summary page:
- Short-term: proceeds, cost basis, net gain/loss (Form 8949 type G/H)
- Long-term: proceeds, cost basis, net gain/loss (Form 8949 type J/K)
- Federal income tax withheld

**Domain trigger:** Always note for `/tax-domain-crypto`.

### 1099-MISC — Miscellaneous Income
Extract:
- **Box 1**: Rents
- **Box 2**: Royalties
- **Box 3**: Other income
- **Box 4**: Federal income tax withheld

**Domain trigger:** If Box 1 (Rents) has a value, note for `/tax-domain-rental`.

## How to Parse

1. Read the file using the Read tool
   - If it's a PDF: Read tool handles PDFs — extract text and look for labeled fields
   - If it's an image (JPG/PNG): Read tool handles images — read the visual content
   - If it's a text/JSON file: parse directly
2. Identify the document type (W2, 1099-B, etc.) from the content
3. Extract all applicable fields listed above
4. Show the extracted data to the user and ask them to confirm or correct
5. Save to `data/tax-profile.json`

## Data Merge

- Read existing `data/tax-profile.json` if it exists (create with default structure if not)
- W2s: Add to `income.w2s[]`. If same employer EIN exists, ask to replace.
- 1099s: Add to `income.otherIncome[]`. If same source+type exists, ask to replace.
- Write back the updated file

## Output

After import, display a clean summary appropriate to the document type:

```
[Document Type] Imported:
  Source: [name] (EIN/TIN: XX-XXXXXXX)
  [Key fields with values]
  Federal Tax Withheld: $X,XXX.XX
```

If domain triggers were detected, mention them.

Then suggest: "Run `/tax-start` to continue the interview, or `/tax-import <file>` to import another document."
