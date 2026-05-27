# review-pr — a PR reviewer that verifies its own findings

An agent skill for reviewing GitHub pull requests. The differentiator: it **fact-checks
its own output before showing it to you**, and it **forces search-based checks to cite
evidence** instead of guessing. Built for reviewers who don't trust an LLM's first draft.

## Why this exists

Most LLM PR reviewers produce a confident list of "issues" — and a chunk of them are
hallucinated: wrong line numbers, problems that aren't in the diff, severities inflated,
or absence-claims ("no test for X") that were never actually checked. This skill is
structured to kill those failure modes:

- **Two fresh-context subagents.** One finds functional gaps and self-classifies each as
  CONFIRMED or dismissed (already-mitigated / false-positive / implicitly-covered). A
  second, with no memory of the first, fact-checks every surviving finding against the
  real diff: VERIFIED / RETRACT / DOWNGRADE / UNVERIFIABLE.
- **Evidence ledger.** Rules that can only be verified by a search (grep for callers,
  count claims against the diff, read a sibling file) must paste the search output into a
  ledger first. A search-class rule with no ledger entry is reported as **NOT RUN**, never
  PASS — so "looks fine" can't masquerade as "checked".
- **Negative verification.** The auditor re-runs the search-class checks the review marked
  PASS and adds a **MISSED** finding if it turns up something the review skipped. This
  catches the most dangerous case: a real gap silently passed over.
- **Coverage against stated requirements.** A coverage table is built from a plan, ticket,
  or the PR body, with an explicit P0 verdict — review is tied to acceptance criteria, not
  vibes.

## How it works

```
clarify → fetch diff → detect type → lint → whitespace filter
       → evidence ledger (search pass) → Q1-Q30 quality rules
       → coverage table → gap-analysis subagent → compile
       → claim-auditor subagent → print review → (optional) post to GitHub
```

The review is printed in chat: a one-line verdict, a check rollup, and a per-concern block
for each finding with **both** the current code and the proposed fix. Nothing is written to
disk or posted to GitHub unless you explicitly ask.

## Requirements

- An agent harness that supports spawning fresh-context subagents (e.g. Claude Code).
- `gh` (GitHub CLI), authenticated.
- A project linter if you want the lint step to run (golangci-lint, eslint, ruff, …).

## Layout

```
SKILL.md                        # the orchestration engine (13 steps + guardrails)
references/checklist-q.md       # Q1-Q30 general quality rules
references/subagent-prompts.md  # gap-analysis + claim-auditor instructions
references/github-posting.md    # gh API templates for inline comments (opt-in)
```

## Usage

Point the skill at a PR and answer the clarifying questions it asks first:

```
review-pr https://github.com/ORG/REPO/pull/1234
```

Optionally pass a ticket URL, a test-plan path, or an old-code path for parity comparison.

## The rule set

Q1-Q30 are language-agnostic engineering rules (examples shown in Go). A few highlights:

- Comments explain WHY not WHAT; soft-TODOs need a tracking link (Q1, Q24, Q28)
- No abstractions without consumers; no vendored copies of upstream code (Q19, Q2)
- Pin external versions; guard type assertions; no fixed sleeps in tests (Q21, Q22, Q23)
- Doc/description must match code and the diff must match the PR body (Q16, Q29, Q30)
- Skip vs fail discipline; iterate the full variant set; scope discipline (Q4, Q8, Q18)

See `references/checklist-q.md` for the full set with fire-conditions and examples.

## Customising

The rules are pluggable. To add a framework- or team-specific profile (e.g. Ginkgo/k8s
test conventions), drop a `checklist-<x>.md` in `references/`, add a step in SKILL.md that
loads it for the relevant PR type, and add its search-class rules to the Step 6 ledger
list. The engine — clarify, ledger, coverage, gap analysis, claim audit, print — stays the
same.
