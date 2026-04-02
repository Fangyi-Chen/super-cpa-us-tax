---
name: tax-evolve
description: Analyze comparison diffs, classify discrepancies, and generate fixes to skill files and references. Logs patterns to evolution-log.json and invokes tax-publish when fixes are ready. Use after running tax-compare.
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep]
---

# Tax Agent Self-Evolution

Analyze discrepancies found by `/tax-compare`, classify root causes, generate fixes to skill files and references, and submit improvements as a GitHub PR.

## Prerequisites

- `data/comparison-{year}.json` must exist. If not, tell the user to run `/tax-compare` first.
- `data/tax-calculation.json` must exist (for tax year context).

## Privacy Contract (CRITICAL — read before every step)

**NEVER include user PII in any generated file changes.** This means:
- Never write actual dollar amounts from the user's return into skill files or references.
- Never include the user's name, SSN, address, employer, or any personal data.
- Only include: which 1040 line was wrong (by line number), the direction of the error (+/-), the percentage error, and the corrected calculation rule.
- When logging to `evolution-log.json`, store only line numbers, deltas, and classification — never raw values that could identify a taxpayer.

## Steps

### Step 1: Load Comparison Data

Read `data/comparison-{year}.json`. Identify all entries with `status: "mismatch"` or `status: "missing_ours"`. If there are no mismatches, tell the user:
> "Great news — no discrepancies found! No fixes needed."
And stop.

### Step 2: Classify Each Discrepancy

For each mismatch, classify it into one of three categories by reasoning about the line:

**`calculation_error`** — The interview collected the right data, but the formula in `tax-calculate` or its references (`brackets-2025.md`, `credits-2025.md`) produced the wrong result.
- Signs: Income lines (1a, 11, 15) are correct but credits or deductions lines (19, 21, 24) differ.
- Action: Edit `skills/tax-calculate/SKILL.md` or the relevant reference file.

**`missing_data`** — The interview did not ask a question needed to correctly compute this line.
- Signs: A line is `missing_ours` (zero in our return, non-zero in reference) for a deduction or credit the user is entitled to.
- Action: Propose new interview questions for `skills/tax-start/SKILL.md`.

**`missing_domain`** — The discrepancy involves an income or deduction type that has no dedicated domain skill yet (e.g., Schedule C self-employment, HSA, student loan interest).
- Signs: An entire schedule or section is zero in our return but non-zero in reference.
- Action: Propose a new `tax-domain-*` skill scaffold.

Document the classification for each mismatch before proceeding to fixes.

### Step 3: Check Evolution Log for Patterns

Read `data/evolution-log.json` (create it if it does not exist with `{"entries": []}`).

Look for lines that have appeared as mismatches in previous runs (same `line` field). If any line has appeared 2 or more times:
> "Line {N} ({description}) has appeared as a mismatch in {count} previous runs. This suggests a systemic gap — proposing a domain skill scaffold rather than a one-off fix."

Upgrade the classification of repeat offenders to `missing_domain` if appropriate.

### Step 4: Generate Fixes

For each classified discrepancy, generate the actual file changes. Do NOT write files yet — collect all proposed changes first.

#### For `calculation_error`:

1. Identify which reference file or SKILL.md section contains the incorrect formula.
   - Use `Grep` to find the relevant calculation step in `skills/tax-calculate/SKILL.md`.
   - Check `skills/tax-calculate/references/credits-2025.md` for credit phase-out rules.
   - Check `skills/tax-calculate/references/brackets-2025.md` for bracket amounts.
2. Draft the corrected rule. Example:
   - Wrong: `CTC = number_of_qualifying_children * 2000` (no phase-out applied)
   - Fixed: `CTC = number_of_qualifying_children * 2000, reduced by $50 for each $1,000 of MAGI above $400,000 (MFJ) / $200,000 (other)`
3. Prepare an `Edit` operation for the relevant file.

#### For `missing_data`:

1. Identify which section of the `tax-start` interview would logically include the missing question.
2. Draft the new question(s) to add. Example:
   - Missing: Estimated tax payments (Line 26)
   - New question: "Did you make any estimated tax payments during the year? If so, enter the total amount paid."
3. Prepare an `Edit` operation for `skills/tax-start/SKILL.md`.

#### For `missing_domain`:

1. Identify the income/deduction type.
2. Draft a new skill scaffold following the domain skill template from the spec:
   ```markdown
   ---
   name: tax-domain-{name}
   description: [domain] tax rules. Invoked by core skills when [trigger].
   allowed-tools: [Read, Write, Edit, AskUserQuestion]
   ---
   ## Trigger Conditions
   ## Interview Questions
   ## Data Schema
   ## Calculation Rules
   ## References
   ```
3. Prepare `Write` operations for `skills/tax-domain-{name}/SKILL.md` and an empty `skills/tax-domain-{name}/references/` placeholder.

### Step 5: Show Proposed Changes to User

Before writing any files, present all proposed changes for review:

```
Proposed fixes for 3 discrepancies:
============================================================

1. [calculation_error] Line 19 — Child Tax Credit
   File: skills/tax-calculate/references/credits-2025.md
   Change: Add MAGI phase-out rule for CTC above $200,000/$400,000

2. [missing_data] Line 26 — Estimated Tax Payments
   File: skills/tax-start/SKILL.md
   Change: Add question about estimated quarterly tax payments

3. [missing_domain] Line 17 — Rental Income (Schedule E)
   New file: skills/tax-domain-rental/SKILL.md
   Change: New domain skill scaffold for rental property income

============================================================
Proceed with these changes? (yes/no/review <n>)
```

- If the user says **yes**, proceed to Step 6.
- If the user says **no**, stop and do not write anything.
- If the user says **review 2**, show the full diff for change #2 and ask again.
- Allow the user to exclude specific changes before proceeding.

### Step 6: Apply Approved Fixes

Apply only the changes the user approved. Use `Edit` for modifications to existing files, `Write` for new files.

After applying changes, run a quick sanity check:
```bash
ls skills/tax-calculate/references/
ls skills/tax-start/SKILL.md
```

### Step 7: Update Evolution Log

Append an entry to `data/evolution-log.json`:

```json
{
  "date": "2026-04-02",
  "comparisonFile": "data/comparison-2025.json",
  "lines": [
    {
      "line": "19",
      "description": "Child tax credit",
      "deltaDirection": "under",
      "deltaPercent": 17.6,
      "classification": "calculation_error",
      "resolved": true,
      "fixFile": "skills/tax-calculate/references/credits-2025.md"
    }
  ]
}
```

Fields to include per line entry:
- `line` — 1040 line number (string, e.g., "19")
- `description` — human-readable description
- `deltaDirection` — `"over"` (our value higher) or `"under"` (our value lower)
- `deltaPercent` — absolute percentage difference, e.g., `17.6` for 17.6% off
- `classification` — one of `calculation_error`, `missing_data`, `missing_domain`
- `resolved` — `true` if a fix was applied this run, `false` if skipped
- `fixFile` — the file that was changed (if resolved), else `null`

**Do NOT store actual dollar amounts in the log.**

### Step 8: Invoke tax-publish

If at least one fix was applied, tell the user:
> "Fixes applied. Ready to submit as a GitHub PR. Invoking `/tax-publish`..."

Then invoke `/tax-publish evolve-{year}-fixes`.

If no fixes were applied (user declined all), say:
> "No changes were applied. You can run `/tax-evolve` again after reviewing the comparison data."

## Error Handling

- If `data/comparison-{year}.json` is missing or has no mismatch entries, tell the user and stop.
- If `data/evolution-log.json` is corrupted, create a fresh one with `{"entries": []}` and warn the user.
- If a referenced skill file does not exist (e.g., `skills/tax-calculate/references/credits-2025.md`), tell the user which file needs to be created first and stop.
- If the user declines to proceed in Step 5, do not write any files.
