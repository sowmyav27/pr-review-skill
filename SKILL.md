---
name: review-pr
description: >
  Single-entry PR reviewer that verifies its own findings before showing them to
  you. Asks clarifying questions, detects PR type, runs the project linter, applies
  Q1-Q30 generic quality rules, builds a coverage table against the stated
  requirements, and uses two fresh-context subagents — one to find functional gaps
  (self-classifying each as confirmed or dismissed) and one to fact-check every
  finding (and re-run search-based checks to catch what the review missed). Prints
  the review in chat — verdict, check rollup, and per-concern blocks with both
  current-code and fix-code snippets. Never posts to GitHub unless you ask.
---

You are reviewing a GitHub pull request. Follow every step below exactly, in order.

The detailed rule definitions live in `references/`. SKILL.md is the
orchestration; reference files are the rule libraries. Load each reference only
when its step actually fires.

------------------------------------------------------------
INPUTS
------------------------------------------------------------

Required:
- PR URL (e.g. https://github.com/ORG/REPO/pull/1234)

Optional (provide in your message or answer in Step 1):
- Issue / ticket URL — for requirement extraction
- Old code path — for parity comparison against new code
- Test plan / acceptance criteria — for coverage assessment

------------------------------------------------------------
STEP 0 — PARSE INPUTS
------------------------------------------------------------

1. Extract the PR URL from the argument or user message.
   If no GitHub URL is found, ask: "Please provide a GitHub PR URL to review."

2. From the URL, derive:
   - `ORG` and `REPO`
   - `PR_NUMBER` (the numeric ID)

3. Note any secondary inputs the user provided:
   - Issue / ticket URL → set `HAS_TICKET=true`
   - Old code path → set `HAS_OLD_CODE=true`
   - Test plan / acceptance criteria → set `HAS_PLAN=true`

Do not fetch the PR yet. Proceed to Step 1.

------------------------------------------------------------
STEP 1 — ASK CLARIFYING QUESTIONS
------------------------------------------------------------

STOP and wait for answers before continuing. Ask only what is not already obvious
from the inputs:

1. What is the purpose of this PR? (new feature, bug fix, test addition,
   refactor, docs, config change, etc.)
2. What should the review focus on? (correctness, test coverage, requirements
   alignment, code quality)
3. Is there a test plan, acceptance criteria, issue, or design doc to check
   against? Paste content or URL.
4. (Only if `HAS_OLD_CODE=false` and this looks like a rewrite/replacement) Old
   version of the code to compare against? Share path.

Record answers as `CONTEXT` for the final report.

------------------------------------------------------------
STEP 2 — FETCH PR DIFF AND METADATA
------------------------------------------------------------

```bash
gh pr view "$PR_NUMBER" --repo "$ORG/$REPO" \
  --json title,author,headRefName,baseRefName,body,changedFiles,headRefOid

gh pr diff "$PR_NUMBER" --repo "$ORG/$REPO" --name-only

gh pr diff "$PR_NUMBER" --repo "$ORG/$REPO"
```

From `--name-only`, build three lists:
- Added files (new paths not in base branch)
- Deleted files (removed paths)
- Modified files

Read the full diff carefully. Record the head SHA as `HEAD_SHA` and the base SHA
as `BASE_SHA` for later file reads.

------------------------------------------------------------
STEP 3 — DETECT PR TYPE
------------------------------------------------------------

Apply rules in order; the first match wins.

| Priority | Condition | Type |
|---|---|---|
| 1 | Only docs / markdown / comment changes | `docs` |
| 2 | Only config / manifest changes (YAML, JSON, TOML) with no code logic | `config` |
| 3 | Changed files are predominantly test files (`*_test.*`, `*.spec.*`, `test/`, `tests/`, `spec/`) | `test` |
| 4 | PR title/body indicates a fix AND source files change | `bugfix` |
| 5 | Diff adds a new exported API / feature surface | `feature` |
| 6 | Everything else | `general` |

Print clearly:
```
Detected PR type: <type>
```

If the type contradicts the user's Step 1 answer, ask them to confirm before
continuing. The type sets emphasis only — every rule below still applies.

------------------------------------------------------------
STEP 4 — RUN THE PROJECT LINTER
------------------------------------------------------------

Skip if the project has no configured linter or no code files changed.

Run the project's linter on the changed files. Use whatever the repo already
configures; do not invent flags. Examples:

```bash
# Go
golangci-lint run ./... 2>&1 | head -80
# JS / TS
npx eslint <changed files>
# Python
ruff check <changed files>
```

If lint errors are found:
- List each error with `file:line` and the linter / rule name.
- Flag them as **BLOCKING** under category `[LINT]` in the final report.
- Continue to Step 5 — but note that quality review is meaningful only after lint
  is clean.

------------------------------------------------------------
STEP 5 — WHITESPACE-ONLY DIFF FILTER
------------------------------------------------------------

Before running quality checks, filter out indentation-only changes.

For each line that appears as both `-` and `+` in the same hunk, strip leading
whitespace and compare. If the trimmed content is identical, treat as unchanged
context — do not flag it in any check below.

This prevents flagging closure / block-nesting refactors and other indentation
shifts as new violations.

------------------------------------------------------------
STEP 6 — SEARCH PASS (BUILD THE EVIDENCE LEDGER)
------------------------------------------------------------

Run this BEFORE the Q-rule step. Purpose: force the search-based rules to produce
evidence instead of a verdict from impression. A rule whose verification needs a
grep, a count, or reading another file cannot be marked PASS on inspection — it
must cite a ledger entry built here.

**SEARCH-class rules** (verified by an active search, not by reading the changed
line): **Q2, Q3, Q5, Q7, Q8, Q10, Q11, Q15, Q16, Q17, Q18, Q19, Q27, Q29, Q30.**

Run the searches below over the checked-out PR branch (the same working copy Step 4
lints; if there is no local checkout, read files via
`gh api repos/$ORG/$REPO/contents/PATH?ref=$HEAD_SHA`). Paste each command's actual
output into the ledger.

- **Q19** (new exported identifier has no consumer): for each exported func/type/
  const added in the diff, run `git grep -nw '<ident>' -- '<source globs>'` and
  subtract the definition line. Zero remaining matches means the rule fires.
- **Q16, Q29** (doc / description counts vs diff): extract every count phrase from
  the doc-comment / PR body, then count the matching test blocks or named items in
  the diff. Record claimed-N vs actual-N.
- **Q17, Q30** (literal mirrors a sibling file / description vs behaviour): open the
  cited sibling file (e.g. a manifest, package descriptor, or flag definition) and
  read the source value or the implemented behaviour.
- **Q11** (same fact computed in two places): `git grep -n '<value-or-fact>'`
  across CI config and source to confirm whether both compute it.
- **Q3, Q10** (sibling naming / duplication): grep the added identifiers and blocks
  to enumerate the siblings being compared.

For SEARCH-class rules that are judgment, not a command (Q5, Q7, Q8, Q15, Q18, Q27),
the ledger entry is the explicit enumeration you performed (files compared, call
sites listed, variants counted), not a command output.

Print the ledger before Step 7, one line per in-scope SEARCH-class rule:

```
Evidence Ledger
Q19  new exported idents N=<n>; callers each: <ident=count, ...>; zero-caller: <list|none>
Q29  body counts: <phrase=N,...>; diff actual: <N>; match: <yes|no>
Q17  sibling file <path> read=yes; literal <x> source value: <y>; match: <yes|no>
...
```

**Contract:** in Step 7 a SEARCH-class rule may be marked PASS or FAIL only if it
has a ledger entry above. A SEARCH-class rule with no ledger entry is reported as
NOT RUN, never PASS. Annotate the summary line, e.g.
`Q1-Q30: FAIL on Q19 · NOT RUN: Q27 (no search evidence) · 22 PASS`.

------------------------------------------------------------
STEP 7 — Q-RULES: GENERAL QUALITY (ALL PR TYPES)
------------------------------------------------------------

Read `references/checklist-q.md`. Apply each rule Q1 through Q30 to added or
changed lines only (after the whitespace filter from Step 5).

Report format — one summary line only:
```
Q1-Q30: FAIL on Q4, Q11 — 28 PASS, 0 N/A
```
If no rules fail: `Q1-Q30: all PASS`

Do NOT emit a per-rule PASS or N/A line. Full detail for each FAIL rule goes only
in the concerns section.

------------------------------------------------------------
STEP 8 — COVERAGE TABLE AGAINST REQUIREMENTS
------------------------------------------------------------

Always run this step. Build a coverage table from a scope source.

**Scope source priority:**

1. If `HAS_PLAN=true` or user provided a test plan in Step 1 → use scenarios from
   the plan.
2. If `HAS_TICKET=true` → read the linked issue. Extract the problem statement,
   expected fix/feature behaviour, and acceptance criteria.
3. If the user provided requirements in Step 1 → use those.
4. Otherwise → derive from the PR title, description, and diff. State scope as
   "(derived from PR description — no external source)".

When extracting from a ticket or thread, label the items so the user can correct
them before you proceed:
- **R1, R2, …** — stated requirements ("should", "must", "needs to", "AC:")
- **D1, D2, …** — design decisions ("agreed", "decided", "won't do", "out of scope")

**Coverage table:**

| Scenario | Covered? | How |
|---|---|---|
| <P0 scenario — the core fix or feature> | yes / no | <one sentence> |
| <additional scenario> | yes / no | ... |

For test PRs: each distinct test case / path is a row.
For feature PRs: each user-facing behaviour change is a row.

State a clear verdict:
- **P0 covered** — core scenario addressed
- **P0 NOT covered** — core fix/feature is missing or incomplete

List any uncovered scenarios as gaps for the subagent step.

------------------------------------------------------------
STEP 9 — SUBAGENT: FUNCTIONAL GAP ANALYSIS + SELF-VERIFICATION
------------------------------------------------------------

Before spawning the subagent, build a **stripped diff**: keep only lines prefixed
with `+` or `-` and hunk headers (`@@ ... @@`). Drop all context lines. Use this
stripped diff for the subagent call. If the subagent needs surrounding context for
a specific finding, it should read the file directly via
`gh api repos/$ORG/$REPO/contents/PATH?ref=$HEAD_SHA`.

Spawn a single fresh-context `general-purpose` subagent.

Read `references/subagent-prompts.md` (Gap Analysis section) and pass the
instructions verbatim, along with:
- The stripped diff (+ and - lines + hunk headers only)
- The CONTEXT recorded in Step 1
- The coverage table from Step 8
- The PR type from Step 3

The subagent finds candidate gaps AND self-classifies each as:
CONFIRMED / ALREADY-MITIGATED / FALSE-POSITIVE / IMPLICITLY-COVERED.

Use only CONFIRMED gaps in the final report. Record the dropped-gap count and
reasons under "Subagent verification summary".

------------------------------------------------------------
STEP 10 — COMPILE THE REPORT
------------------------------------------------------------

**Per-concern format (full detail; used for the file report and GitHub comments):**

```
### <ID> — <one-line title>

[Internal only, not in GitHub comment: Severity: BLOCKING / Non-blocking (score N/5)]

**What's the concern?**
- <bullet: what is wrong and where>
- <bullet: what behaviour or rule this violates>
- <bullet: any additional context>

**Why does it matter?**
- <bullet: what breaks or degrades if this is not fixed>
- <bullet: blocking condition — "Becomes blocking if <specific downstream failure>".
  Drop the concern if you cannot state one.>

**Code (current)**
```
<the offending code from the diff>
```

**Proposed fix**
<one or two plain-language sentences>

```
<minimal corrected code snippet>
```

**How this fixes the concern**
<one sentence — what the fix changes about behaviour or coverage>
```

**Severity is tracked internally for report ordering only.** It must not appear in
GitHub comment bodies. Scale (ordering only):

- **1** — style nit
- **2** — minor improvement
- **3** — real gap, core feature still works
- **4** — should fix this PR cycle
- **5** — borderline blocking
- **BLOCKING** — must be fixed before merge

**Final report structure:**

```
## PR Review: <title>

Type: <detected type>
Author: <author>
Branch: <headRefName> → <baseRefName>

---

### Context
<bullets summarising the user's Step 1 answers>

### Checks run
<bulleted list: Lint / Q1-Q30 / Coverage / Subagent / Claim audit>

### Lint
<PASS or FAIL with details>

### Q-rules (Q1-Q30)
<summary line, e.g. "FAIL on Q4, Q11 — 28 PASS" or "all PASS">

### Coverage assessment
Scope source: <plan / ticket / PR description>

| Scenario | Covered? | How |
|---|---|---|
| ... | ... | ... |

P0 verdict: <covered / NOT covered>

### Subagent verification summary
<N candidate gaps found; M confirmed; K dropped (reasons)>

---

### Concerns (BLOCKING first, then by score descending)

<per-concern format for each item>

---

### Verdict

LGTM — ready to merge
Changes needed — see concerns above
LGTM with concerns — no blocking concerns; see report

### Audit retractions / downgrades   (if any from Step 11)
### Audit additions                  (if any MISSED findings from Step 11)
```

Verdict rules:
- Any BLOCKING → "Changes needed"
- Only non-blocking concerns → "LGTM with concerns"
- Nothing flagged → "LGTM"

------------------------------------------------------------
STEP 11 — AUDIT ALL FINDINGS (CLAIM VERIFICATION)
------------------------------------------------------------

Before printing anything, spawn a fresh-context `general-purpose` subagent to
fact-check the report against the actual diff and source files. This step exists
because the main thread can produce plausible-sounding claims that are wrong when
checked against real line numbers or current file state.

Read `references/subagent-prompts.md` (Claim Auditor section) and pass the
instructions verbatim, along with:
- The full compiled report from Step 10 (all findings, with file paths and lines)
- The stripped diff from Step 9 (+ and - lines + hunk headers only)
- The head SHA (for `gh api` file reads if surrounding context is needed)
- The Step 6 Evidence Ledger and the SEARCH-class rule list (so the auditor can
  re-run the searches and verify the negatives, not just the findings)

The auditor returns **two** results:

A. For each existing finding, one of:
   - **VERIFIED** — the cited code exists at the cited location and the claim is accurate
   - **RETRACT** — the claim is factually wrong (wrong line/version, code doesn't say
     what the finding claims, or the "problem" isn't in the diff)
   - **DOWNGRADE** — directionally correct but overstated (severity inflated, or
     applies to pre-existing code rather than the diff)
   - **UNVERIFIABLE** — depends on runtime behaviour or external state; cannot confirm
     from the diff alone

B. A separate negative-verification pass: for each SEARCH-class rule the review
   marked PASS, re-run the search. If the re-run finds a real issue the review
   missed, emit a new **MISSED** finding with quoted evidence.

**After receiving results:**
- RETRACT → remove the finding; note it under "Audit retractions".
- DOWNGRADE → adjust severity; note the change.
- UNVERIFIABLE → add a "(unverifiable locally)" caveat to the finding.
- VERIFIED → no change.
- MISSED → add as a new CONFIRMED concern; note it under "Audit additions".

Update the compiled report before Step 12. List RETRACT / DOWNGRADE under "Audit
retractions / downgrades" and MISSED under "Audit additions".

------------------------------------------------------------
STEP 12 — PRINT REVIEW IN CHAT (no file written)
------------------------------------------------------------

Print the review directly in the conversation. Do NOT write a report file unless
the user explicitly asks for one. The chat is the primary deliverable.

Print, in this order:

1. **One-line verdict line:**
   `Verdict: <LGTM | LGTM with concerns | Changes needed> — <N> blocking, <M> non-blocking`

2. **One-line check rollup:**
   `Lint: <PASS|FAIL> · Q: <FAIL on …|all PASS> · Coverage: <P0 covered|P0 NOT covered>`

3. **Concerns list**, BLOCKING first then by score descending. Each finding uses
   this exact block (concern → proposed fix → BOTH code snippets):

   ```
   ### <ID> — <one-line title>
   File: <path>:<line>

   **Concern:** <one or two sentences — what is wrong, where, and which rule it violates>

   **Proposed fix:** <one or two sentences — the concrete change to make>

   Current:
   ```
   <minimal real snippet from the diff>
   ```

   Fix:
   ```
   <minimal corrected snippet>
   ```
   ```

4. Final line: `Nothing posted to GitHub.`

Rules for this step:
- Both code blocks are mandatory for code findings. "Current" must quote real code
  from the diff verbatim. "Fix" must show the concrete replacement, not pseudo-code.
- Keep each snippet minimal — usually 3-12 lines. If the fix is "delete this line",
  show the line in `Current` and the surrounding context with it gone in `Fix`.
- The "Concern" and "Proposed fix" lines are short — full rationale, blocking
  condition, and "How this fixes the concern" stay in reserve for the GitHub comment
  body (Step 13) or the on-request file report.
- For documentation-only findings, omit the code blocks and use a single
  `Suggested edit:` before/after block of the prose.
- Skip the concerns list only if there are zero concerns.
- Do not write any review file unless the user explicitly says "save" / "write to
  disk" / "give me the report file". When asked, write the full per-concern report
  (Step 10 format) to `output/pr-review-<ORG>-<REPO>-<PR_NUMBER>.md`.

------------------------------------------------------------
STEP 13 — POST INLINE COMMENTS TO GITHUB (requires user sign-off)
------------------------------------------------------------

**Never post anything automatically.** Always present comments for review first.

Print each proposed GitHub comment in this format (severity labels included here so
you can prioritise):

```
--- Comment <N> of <total> ---
[BLOCKING / Score: N/5] <one-line title>
File: <path>  Line: <N>

**What's the concern?**
- ...

**Why does it matter?**
- ...

**Proposed fix**
<text + code>
```

Then ask: "Review the <N> comments above. Reply 'post all', 'post <IDs>', or 'skip'
to decide what gets posted."

Wait for the user's reply before calling any GitHub API. When the user approves,
read `references/github-posting.md` for the `gh api` command template. **Strip the
`[BLOCKING / Score: N/5]` line from the comment body before posting** — GitHub
comments must be plain, no severity labels.

------------------------------------------------------------
GUARDRAILS
------------------------------------------------------------

1. **Always stop at Step 1.** Do not fetch the PR or run any analysis until the user
   has answered the clarifying questions. The PR description is not a substitute for
   the user's stated focus.

2. **Confirm every check.** For Q1-Q30, report a summary line only. Full detail for
   FAILs goes in concerns only.

3. **Diff scope only.** Do not flag pre-existing issues in unchanged code unless the
   changed lines directly activate or depend on them.

4. **One finding per issue.** Do not report the same concern twice. Pick the higher
   severity.

5. **Comments must stand alone.** Flag code comments that reference what was there
   before ("replaced X with Y", "previously …"). The commit message is where change
   context belongs.

6. **One fix recommendation, not a menu.** If multiple fixes are possible, pick the
   strongest and explain why. Don't shift the decision to the reviewer.

7. **Each concern must state its blocking condition.** Include "Becomes blocking if
   <specific downstream failure>." Drop the concern if you cannot state one.

8. **Ground every fix in existing patterns.** Before recommending a fix, check how
   similar problems are solved elsewhere in the same file or codebase. State the
   precedent or say none exists.

9. **Gap self-verification is non-negotiable.** The gap-analysis subagent must
   classify every candidate as CONFIRMED / ALREADY-MITIGATED / FALSE-POSITIVE /
   IMPLICITLY-COVERED by reading the diff directly. Do not include any gap not
   self-classified as CONFIRMED.

10. **Never post to GitHub without explicit user approval.**

11. **Severity labels never appear in GitHub comment bodies.**

12. **Chat is the primary output; do not write files by default.**

13. **Whitespace-only changes are not new code.** Strip leading whitespace before
    comparing `+`/`-` pairs in a hunk.

14. **Step 11 is non-negotiable.** Every finding must pass the claim auditor before
    the review is printed. A RETRACT verdict removes the finding — do not reinstate
    it without re-reading the cited code yourself.

15. **Negative claims require full enumeration.** Before asserting something is
    absent ("no coverage", "no cleanup", "nothing handles X"), enumerate and check
    every place it could plausibly exist. A negative based on a single file is
    inadmissible.

16. **Search-based rules must cite evidence (Step 6 ledger).** A SEARCH-class rule
    with no ledger entry is NOT RUN, not PASS. This is the forcing function that
    operationalises Guardrail 15 — it is not advice.
