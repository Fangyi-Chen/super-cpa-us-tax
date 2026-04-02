# Plan 3: Self-Evolution Skills Implementation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create 3 self-evolution skills that allow the tax agent to compare its output against reference returns (TurboTax), analyze discrepancies, generate fixes, and submit improvements as GitHub PRs.

**Architecture:** `tax-compare` reads two PDFs and produces a structured diff. `tax-evolve` classifies discrepancies and generates skill/reference file changes. `tax-publish` handles git branching, committing, pushing, and PR creation via `gh`.

**Tech Stack:** Claude Code skills (Markdown), Python 3 + pypdf (PDF reading), Git + GitHub CLI (PR workflow)

**Spec:** `docs/superpowers/specs/2026-04-02-tax-agent-plugin-design.md`

---

## Task 1: Create tax-compare skill

**File to create:** `skills/tax-compare/SKILL.md`

### Steps

- [ ] Create directory `skills/tax-compare/`
- [ ] Write `skills/tax-compare/SKILL.md` with the complete content below

### Complete `skills/tax-compare/SKILL.md` content

```markdown
---
name: tax-compare
description: Diff two tax return PDFs (ours vs TurboTax or other reference). Extracts field values from both PDFs, compares them line by line against Form 1040 field mapping, and saves a structured diff to data/comparison-{year}.json. Use when the user wants to verify accuracy or contribute improvements.
allowed-tools: [Read, Write, Bash, Glob]
argument-hint: <our-pdf> <reference-pdf>
---

# Tax Return Comparator

Compare this plugin's output against a reference return (e.g., TurboTax) to identify discrepancies.

## Prerequisites

- Python 3 with `pypdf` must be available.
- `data/tax-calculation.json` must exist (provides the expected tax year).
- Both PDF paths must be accessible on the local filesystem.

Check prerequisites:
```bash
python3 -c "from pypdf import PdfReader; print('pypdf OK')"
ls data/tax-calculation.json
```

If pypdf is missing:
```bash
pip3 install pypdf
```

## Arguments

Invocation: `/tax-compare <our-pdf> <reference-pdf>`

- `<our-pdf>` — path to this plugin's generated PDF (e.g., `data/output/1040-2025.pdf`)
- `<reference-pdf>` — path to the TurboTax or other software's PDF

If either argument is missing, ask the user to provide both paths before continuing.

## Steps

### Step 1: Determine Tax Year

Read `data/tax-calculation.json` and extract the `taxYear` field. Use this to name the output file `data/comparison-{year}.json`.

### Step 2: Extract Field Values from Both PDFs

Run the following Python snippet to extract all form field values from each PDF:

```python
from pypdf import PdfReader

def extract_fields(path):
    reader = PdfReader(path)
    fields = reader.get_fields() or {}
    return {
        name: (field.value if hasattr(field, 'value') else str(field))
        for name, field in fields.items()
    }

our_fields = extract_fields("<our-pdf>")
ref_fields = extract_fields("<reference-pdf>")
```

If a PDF has no form fields (e.g., it is a printed/scanned PDF rather than a fillable form), warn the user:
> "The PDF at `<path>` does not contain fillable form fields. Comparison requires a fillable PDF exported directly from TurboTax or this plugin."

### Step 3: Load the 1040 Field Mapping

Read `skills/tax-fill/references/form-1040-fields.md` to get the canonical list of Form 1040 lines and their corresponding PDF field names. This mapping is the authoritative list of fields to compare.

The mapping format is:
```
| Line | Description            | PDF Field Name          | JSON Key                  |
|------|------------------------|-------------------------|---------------------------|
| 1a   | Total wages            | f1_15[0]                | income.totalWages         |
| 11   | AGI                    | f1_32[0]                | adjustedGrossIncome       |
...
```

Build a list of `(line, description, our_field_name, ref_field_name)` tuples to compare. When both PDFs use the same fillable IRS form, field names will match. When the reference PDF uses different field names (e.g., TurboTax-generated), attempt to match by line number from the mapping.

### Step 4: Compare Field by Field

For each mapped field:

1. Extract the numeric value from our PDF and the reference PDF. Strip `$`, `,`, whitespace. Treat empty or missing as `0`.
2. Classify:
   - **match** — values are equal (within $1 rounding tolerance)
   - **mismatch** — values differ
   - **missing_ours** — field present in reference but missing/blank in ours
   - **missing_ref** — field present in ours but missing/blank in reference

Compute `delta = our_value - ref_value` for numeric fields.

### Step 5: Save Comparison to JSON

Write `data/comparison-{year}.json` with the following schema:

```json
{
  "taxYear": 2025,
  "generatedAt": "2026-04-02T00:00:00",
  "ourPdf": "data/output/1040-2025.pdf",
  "referencePdf": "<reference-pdf>",
  "summary": {
    "totalFields": 42,
    "matches": 38,
    "mismatches": 3,
    "missingOurs": 1,
    "missingRef": 0
  },
  "fields": [
    {
      "line": "1a",
      "description": "Total wages",
      "ourValue": 120000,
      "refValue": 120000,
      "delta": 0,
      "status": "match"
    },
    {
      "line": "19",
      "description": "Child tax credit",
      "ourValue": 1648,
      "refValue": 2000,
      "delta": -352,
      "status": "mismatch"
    }
  ]
}
```

### Step 6: Display On-Screen Diff Summary

Print a human-readable summary:

```
Tax Return Comparison — 2025
============================================================
Total fields compared: 42
Matches:    38 ✓
Mismatches:  3 ✗
Missing (ours): 1

MISMATCHES:
  Line 19  Child tax credit       Ours: $1,648   Ref: $2,000   Δ -$352
  Line 25a Estimated tax payments Ours: $0       Ref: $400     Δ -$400
  Line 37  Amount you owe         Ours: $1,200   Ref: $448     Δ +$752

MISSING IN OUR RETURN:
  Line 12b Charitable contribution (no standard ded. limit)

Saved to: data/comparison-2025.json
============================================================
```

### Step 7: Offer Community Contribution

After displaying the diff, ask:

> "Would you like to help improve this tax agent for the community? Your personal tax data stays local — only the line-item deltas and any rule fixes are shared. Run `/tax-evolve` to analyze and propose fixes."

Do not run `/tax-evolve` automatically. Wait for the user to decide.

## Error Handling

- If `data/tax-calculation.json` is missing, tell the user to run `/tax-calculate` first.
- If either PDF path does not exist, show a clear error and stop.
- If pypdf fails to open a PDF (e.g., password-protected), tell the user to export an unlocked copy.
- If no fields are mapped (the field mapping reference is missing), warn and proceed with raw field name matching as a best-effort fallback.
```

---

## Task 2: Create tax-evolve skill

**File to create:** `skills/tax-evolve/SKILL.md`

### Steps

- [ ] Create directory `skills/tax-evolve/`
- [ ] Write `skills/tax-evolve/SKILL.md` with the complete content below

### Complete `skills/tax-evolve/SKILL.md` content

```markdown
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
```

---

## Task 3: Create tax-publish skill

**File to create:** `skills/tax-publish/SKILL.md`

### Steps

- [ ] Create directory `skills/tax-publish/`
- [ ] Write `skills/tax-publish/SKILL.md` with the complete content below

### Complete `skills/tax-publish/SKILL.md` content

```markdown
---
name: tax-publish
description: Create a git branch, stage only skill and reference files, commit, push, and open a GitHub PR with anonymized evidence. Use after tax-evolve applies fixes, or when contributing manual improvements.
allowed-tools: [Read, Write, Bash, Glob]
argument-hint: "[description]"
---

# Tax Agent Publisher

Submit skill improvements as a GitHub pull request. Stages only safe files (never user data), shows a full diff for review, and opens a PR with anonymized context.

## Arguments

Invocation: `/tax-publish [description]`

- `[description]` — optional short description for the branch name (e.g., `fix-ctc-phaseout`)
- If omitted, defaults to `improvement`

## Prerequisites

- Git must be installed: `git --version`
- GitHub CLI must be installed: `gh --version`
- The repository must be initialized: `git status`
- `.claude-plugin/plugin.json` must exist with a `repository` field.

## Steps

### Step 1: Read Repository from Plugin Manifest

Read `.claude-plugin/plugin.json` and extract the `repository` field.

If `repository` is empty or missing:
> "The `repository` field in `.claude-plugin/plugin.json` is not set. Please add your GitHub repository URL (e.g., `https://github.com/fangyic/tax-agent`) and run `/tax-publish` again."

Stop if no repository is configured.

### Step 2: Determine Branch Type and Name

- If invoked from `/tax-evolve` (argument starts with `evolve-`): use prefix `evolve/`
- Otherwise (user-initiated contribution): use prefix `contrib/`

Format the date as `YYYY-MM-DD`.

Branch name: `{prefix}{description}-{date}`

Examples:
- `evolve/fix-ctc-phaseout-2026-04-02`
- `contrib/add-estimated-payments-question-2026-04-02`

```bash
git checkout -b {branch-name}
```

If the branch already exists, append a short random suffix or increment a counter.

### Step 3: Identify Staged Files

Determine which files have been modified since the last commit:

```bash
git status --short
```

Filter to only include safe files — those that match these patterns:
- `skills/*/SKILL.md`
- `skills/*/references/*.md`
- `.claude-plugin/plugin.json`
- `package.json`
- `README.md`
- `forms/*.pdf`

**NEVER stage these paths:**
- `data/` (all user tax data — gitignored, but verify)
- `*.pdf` outside of `forms/` (user-generated forms)
- Any file containing PII (check filenames for: `profile`, `calculation`, `comparison`, `evolution-log`, `output`)

Show the user the list of files that will be staged and ask for confirmation before proceeding.

### Step 4: Show Full Diff for Review

```bash
git diff HEAD -- {safe-files-list}
```

Display the full diff to the user and say:
> "Please review the above diff carefully. If you see any personal information (names, SSNs, dollar amounts from your return), say 'stop' and do not proceed. Otherwise, say 'looks good' to continue."

Wait for the user's explicit confirmation before proceeding.

### Step 5: Stage Safe Files

Stage only the approved files:

```bash
git add {safe-files-list}
```

Verify staging:
```bash
git status --short
```

Confirm that `data/` and any PDF outside `forms/` are NOT staged.

### Step 6: Commit

Write a structured commit message:

```
fix(skills): {short description of what changed}

Changes:
- {file 1}: {what was changed}
- {file 2}: {what was changed}

Root cause: {classification from tax-evolve, if available}
Evidence: {anonymized line deltas — e.g., "Line 19 CTC: under by ~18%"}

No user PII included in this commit.
```

Commit:
```bash
git commit -m "{structured message}"
```

### Step 7: Push to Remote

```bash
git push -u origin {branch-name}
```

If the push fails due to authentication, tell the user:
> "Push failed. Ensure you are authenticated with `gh auth login` and have write access to the repository."

### Step 8: Open Pull Request

Use the GitHub CLI to open a PR:

```bash
gh pr create \
  --title "{short description}" \
  --body "{pr body}" \
  --base main
```

PR body format:

```markdown
## Summary

- {bullet: what changed}
- {bullet: what changed}

## Root Cause

{Classification from tax-evolve: calculation_error / missing_data / missing_domain}

{Brief explanation of the rule that was wrong or missing}

## Evidence (Anonymized)

| Line | Description | Direction | Delta % |
|------|-------------|-----------|---------|
| 19   | Child Tax Credit | under | ~18% |

_No user PII included. Deltas are directional percentages only._

## Test Plan

- [ ] Run `/tax-calculate` on a sample return and verify line {N} now matches expected value
- [ ] Run `/tax-compare` on the sample return to confirm mismatch is resolved
- [ ] Review SKILL.md changes for correctness

🤖 Generated with Claude Code tax-agent self-evolution
```

### Step 9: Show PR URL

After the PR is created, display the URL:
> "PR submitted: {url}"
> "Thank you for helping improve the tax agent! The maintainer will review your proposed fix."

## Error Handling

- If `git` is not available, tell the user to install Git.
- If `gh` is not available, tell the user to install GitHub CLI: `brew install gh` or https://cli.github.com
- If `gh auth status` fails (not logged in), tell the user to run `gh auth login`.
- If the push fails, display the error and tell the user to check network access and repository permissions.
- If the user says "stop" during the diff review step, abort without staging, committing, or pushing anything. Run `git checkout main` to return to the main branch.
- If there are no modified skill files to stage, tell the user:
  > "No skill files have been modified. Nothing to publish."
```

---

## Task 4: Create symlinks and final commit

### Steps

- [ ] Create symlinks in `.claude/skills/` for all 3 new skills
- [ ] Verify all 3 skills appear in the skill list
- [ ] Stage all new skill files and symlinks
- [ ] Create a final commit

### Symlink commands

```bash
# From the repo root
ln -s ../../skills/tax-compare .claude/skills/tax-compare
ln -s ../../skills/tax-evolve .claude/skills/tax-evolve
ln -s ../../skills/tax-publish .claude/skills/tax-publish
```

### Verify discovery

```bash
ls -la .claude/skills/
```

All 3 new skills should appear alongside the existing skills.

### Final commit

Stage and commit all new files:

```bash
git add skills/tax-compare/SKILL.md
git add skills/tax-evolve/SKILL.md
git add skills/tax-publish/SKILL.md
git add .claude/skills/tax-compare
git add .claude/skills/tax-evolve
git add .claude/skills/tax-publish
git commit -m "feat: add self-evolution skills (tax-compare, tax-evolve, tax-publish)"
```

---

## Validation Checklist

After all tasks are complete:

- [ ] `skills/tax-compare/SKILL.md` exists and contains valid YAML frontmatter
- [ ] `skills/tax-evolve/SKILL.md` exists and contains valid YAML frontmatter
- [ ] `skills/tax-publish/SKILL.md` exists and contains valid YAML frontmatter
- [ ] `.claude/skills/tax-compare` symlink points to `../../skills/tax-compare`
- [ ] `.claude/skills/tax-evolve` symlink points to `../../skills/tax-evolve`
- [ ] `.claude/skills/tax-publish` symlink points to `../../skills/tax-publish`
- [ ] `data/` is in `.gitignore` (user tax data never committed)
- [ ] No user PII appears in any skill file
- [ ] The `tax-evolve` skill's privacy contract section is present and clearly states the PII prohibition
- [ ] The `tax-publish` skill's diff review step requires explicit user confirmation before pushing
