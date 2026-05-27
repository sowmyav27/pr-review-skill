# Subagent Prompts

This file holds the verbatim instruction blocks passed to subagents. SKILL.md spawns
them; this file is the source of truth for what they receive. Both subagents run with
fresh context and have no memory of earlier steps — pass everything they need.

---

## Gap Analysis — Functional Gap Analysis + Self-Verification (Step 9)

Spawn a single fresh-context `general-purpose` subagent.

Pass to the subagent:
- The stripped diff (+ and - lines + hunk headers only)
- The CONTEXT recorded in Step 1
- The coverage table from Step 8
- The PR type from Step 3
- The lens list below (the subagent must apply each)

### Lens list

```
1. Negative / boundary tests — for every restriction (permission, access, limit), is
   there a test that it actually blocks? For every allow rule, is there a symmetric
   test that the allowed action still works?

2. Lifecycle completeness — are resources created by the change explicitly verified to
   be cleaned up after deletion? Are cleanup assertions real, or just best-effort
   deletes with no assertion?

3. Functional end-to-end — does the test prove the feature works end-to-end, or only
   that setup succeeded? A resource reaching a Ready/Running state is not proof that it
   is functional — verify the system can actually do what it is supposed to do after
   setup completes. If a reference implementation is named, extract every verification
   step from it and check each is present in the new code, not just that the structure
   matches.

4. Security controls — are security-critical deny rules tested at runtime? Is privilege
   escalation prevention verified by actually attempting it?

5. Isolation — if multiple instances coexist, is there a test that they do not interfere
   with each other?

6. Test correctness — wrong identifiers or patterns, assertions skipped due to an
   auth/early-return path, cleanup blocks that delete but never assert deletion.

7. Variant / provider parity — if the change supports multiple variants or providers,
   are gaps between them noted?

8. Unrelated changes bundled in — changes that don't match the PR's stated purpose
   (timeout bumps, comment fixes, config tweaks).

9. Concurrency / race-condition claim accuracy — when a PR fixes parallelism or a TOCTOU
   bug, do the comments accurately describe the scope of the protection?

10. New abstraction consistency — when a PR introduces a helper / getter, is it applied
    at all call sites, not just some?

11. Comment correctness — do log/step descriptions and code comments accurately describe
    what the code does? Flag comments referencing what was there before ("replaced X with
    Y", "previously did Z") — those belong in commit messages.

12. Logic flow — is the end-to-end sequence correct? Are async side effects waited for
    before the next step proceeds? Are stale objects re-fetched where state matters? Are
    cleanup registrations ordered correctly (LIFO)?

13. Helper function correctness — does each helper do only what its name says? Are there
    silent no-op paths? Nil-dereference risks?

14. Skip vs Fail semantic (Q4) — when code branches on input or env, is a skip used where
    a failure should be? Skip is for environment incapability; fail is for invalid input.

15. Justification gaps (Q6) — does any added block perform a non-obvious side effect
    (re-packaging an artifact, mutating host state, copying files, vendoring a config)
    whose comment paraphrases the code without explaining the necessity?

16. Variant coverage symmetry (Q8) — when N variants are created in setup, does every
    verification step iterate all N? Strict subsets without explicit reason are gaps.

17. Cross-language split-brain (Q11) — do the workflow YAML/shell and the code both
    compute or sniff the same fact (version resolution, feature gate, state existence)?
    Flag as needing a single source of truth.

18. SDK abstraction consistency (Q15) — does any added shell-out to a CLI have an
    equivalent already-imported SDK call available?

19. Branch-state logging (Q20) — when the code takes path A vs B based on runtime state
    (resource found / OS gate / version gate / fallback chosen), is the chosen path
    logged?

20. Privilege justification (Q25) — does any added privileged operation (`Privileged:
    true`, host mounts, `sudo`, `--insecure`) explain in a comment near the call site why
    the privilege is necessary?

21. Structural smell (Q9 follow-up) — for any function flagged by the Q9 line-count rule
    (>80 lines added or +40 grown), does it mix unrelated concerns that should be
    extracted to per-concern helpers?

22. Pointless ceremony (Q13) — is any added primitive's distinguishing feature unused
    (parallelism with a single worker, ordering without dependencies, retry without
    async, helper indirection without reuse)?
```

### Return format

For each candidate gap you find, immediately self-classify it before returning. Read
the cited location in the diff directly to verify.

Classification criteria:
- **CONFIRMED** — gap is real, not mitigated, not covered elsewhere
- **ALREADY-MITIGATED** — diff handles this; note exactly where
- **IMPLICITLY-COVERED** — covered by an upstream wait/assertion; note how
- **FALSE-POSITIVE** — issue does not exist in the diff; note why

Return **only CONFIRMED gaps** in this format:

```
GAP-1
File: <path>
Location: <function or block name>
Lens: <which lens triggered>
Issue: <one sentence>
Why it matters: <one sentence>
Suggested fix: <one sentence>
Verified: <one sentence — what you checked in the diff to confirm this is real>
```

Also return a single summary line at the end:
```
Dropped: N ALREADY-MITIGATED, N FALSE-POSITIVE, N IMPLICITLY-COVERED
```

---

## Claim Auditor (Step 11)

Spawn a fresh-context `general-purpose` subagent.

**Critical:** this subagent has no memory of earlier steps. Pass only:
- The full compiled report (all findings: BLOCKING + non-blocking)
- The PR diff (full)
- The head SHA
- The Step 6 Evidence Ledger and the SEARCH-class rule list
- The instructions below

### Instructions

```
You are fact-checking a PR review report. For every finding (both BLOCKING and
non-blocking), verify the claim by reading the actual diff or file directly.

For each finding:

1. Locate the cited file and line number in the diff or local repo.
   - If the finding cites a specific line, read that line.
   - If it cites a code pattern, grep for it.
   - If it references a version string or identifier, check what the diff actually
     says — do not rely on the report's description of it.

2. Ask: does the code at that location match what the finding claims?
   - Does the offending pattern exist?
   - Is the severity correct (is this rule actually BLOCKING, or WARN)?
   - Is the "current code" / "proposed code" quoted accurately?

3. Classify each finding as one of:
   - VERIFIED   — claim is accurate; code exists as described; severity correct
   - RETRACT    — claim is factually wrong (code doesn't exist as described, wrong
                  file/line, "problem" isn't in the diff, or describes pre-existing
                  code not introduced by this PR)
   - DOWNGRADE  — directionally correct but overstated (severity too high, or applies
                  to pre-existing code rather than new code in the diff)
   - UNVERIFIABLE — depends on runtime behaviour or external state not present in the
                  diff; note exactly why it cannot be confirmed

Return Table A (existing findings):

| ID | Title (abbreviated) | Verdict | One-line reason |
|---|---|---|---|
| B1 | ... | VERIFIED | ... |
| NB3 | ... | RETRACT | code at cited line does not show X |

For RETRACT and DOWNGRADE, quote the specific line content that contradicts the claim.
Do not apply your own judgment about whether the code is good or bad — only whether the
specific factual claim is accurate.

4. Verify the negatives (SEARCH-class rules marked PASS).
   You are given the SEARCH-class rule list and the Evidence Ledger. For each
   SEARCH-class rule the review marked PASS:
   - If it has no ledger entry, treat the PASS as unproven.
   - Re-run the search yourself: grep callers for each new exported identifier, search
     for an existing helper/constant before a reimplementation, count PR-body claims
     against the diff, or fetch and enumerate the old file for a rewrite.
   - If your re-run finds a real issue the review missed (e.g. an exported function with
     zero callers, or a description claiming a default the code no longer implements),
     add a NEW finding with verdict MISSED and quote the evidence.

Return Table B (negatives):

| Rule | Marked | Ledger? | Re-run result | Verdict |
|---|---|---|---|---|
| Q19 | PASS | no | newHelperFunc: 0 call sites in changed files | MISSED |
| Q29 | PASS | yes | body "6 specs" vs diff 6 blocks | VERIFIED |
```
