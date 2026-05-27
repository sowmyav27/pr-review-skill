# Q-rules: General Quality (all PR types)

Q-rules apply to **every PR type** — Go, JS/TS, Python, YAML, shell, mixed.

Apply to **added or changed lines only**, after the whitespace filter.
Items marked **BLOCKING** are hard failures. **WARN** are suggestions.

For each rule, explicitly state PASS / FAIL / N/A — do not skip silently.
Examples are shown in Go, but the rules are language-agnostic.

---

### Q1 — Comments explain WHY, not WHAT [WARN]

Comments should justify non-obvious decisions, document hidden constraints, warn
about subtle invariants, or explain edge cases. Comments that restate what the
code does — narrating the next block, naming the category the type/function name
already conveys, or paraphrasing the call — should be removed.

**Fires when:** an added comment paraphrases the immediately following code
without adding constraint, motivation, or warning that isn't visible from the
code itself.

**Related:** Q6 — Q1 removes WHAT comments; Q6 adds missing WHY comments. A single
comment that paraphrases without justifying violates both — fix once, don't
double-count.

---

### Q2 — Don't vendor copies of upstream code that has a canonical source [BLOCKING]

When a script, binary, schema, or other resource has an authoritative published
version (release asset, package registry, framework export), fetch or import it at
runtime — don't embed a local copy. Vendored copies drift silently, double the
maintenance surface, and let the code diverge from the real upstream without anyone
noticing.

**Fires when:** an added file is a verbatim or near-verbatim copy of code, scripts,
or assets that exist at a stable upstream URL or in an importable package.

**Exception:** copies are allowed when upstream is unstable, unauthenticated access
is blocked, or you must pin a historical version upstream may rewrite — comment the
deviation and link to a tracking issue.

---

### Q3 — Naming consistency for sibling identifiers [WARN]

When multiple identifiers represent the same kind of thing in the same scope (config
values for related resources, names for parallel fixtures, options for similar
functions), pick one naming pattern and apply it. Mixed naming forces readers to
check whether the difference is meaningful when usually it isn't.

**Fires when:** two or more newly-added identifiers in the same file/package describe
semantically parallel concepts but use divergent suffixes/prefixes (e.g.
`xxxConfig` vs `xxxValues`, `getXxx` vs `xxxFetch`).

---

### Q4 — Skip is for unsupported environments; Fail is for invalid input [BLOCKING]

A skip is appropriate when a test cannot run meaningfully on the current environment
(wrong OS, missing capability, version mismatch with the system under test). When the
**inputs** to the test are invalid (contradictory flags, malformed config, version
pairs that violate a precondition), the test must **fail** so the misconfiguration is
loud. Skipping on invalid input silently passes runs that should have failed.

**Fires when:** a skip is conditioned on a value derived from test inputs (env vars,
flags, computed config) rather than on a property of the runtime environment.

---

### Q5 — Resolve and validate derived config up-front [WARN]

When code depends on multiple computed values (resolved references, version strings,
paths, credentials), resolve them and validate their consistency at the top of setup,
before any resources are created. Late discovery of an invalid config leaves partial
state behind, makes failures hard to triage, and means cleanup runs against half-built
state.

**Fires when:** derived/computed values are repeatedly recomputed across setup and
execution phases, OR configuration validation (version-pair sanity, credential
presence) happens after the first resource is created.

---

### Q6 — Non-obvious operations must justify themselves [WARN]

Code that performs an unusual or framework-specific operation must explain WHY in a
comment a future maintainer can understand without reading framework source. A comment
describing WHAT ("packages the chart with version X") is not a justification — the
reader still doesn't know why it's necessary. If answering "why is this here?" requires
upstream context, write it down.

**Fires when:** an added block performs a non-obvious side effect (re-packaging an
artifact, mutating host state, copying files between containers, vendoring a config)
and the surrounding comment paraphrases the code without explaining the necessity.

**Related:** Q1 — see cross-reference there.

---

### Q7 — Magic literals belong in named identifiers [WARN]

Strings, URLs, version tags, config paths, numeric thresholds, port numbers, and
timeout values that don't carry their meaning at the call site should be extracted to
named constants or variables. Applies to:

- Repeated literals (the same value in 2+ places)
- One-shot magic values whose meaning isn't obvious from context (e.g. `120` for
  timeout seconds, `0` for "no restarts allowed")
- Long sequences of bare positional arguments where each has a distinct semantic role

**Fires when:** a call passes 3+ string literals as positional arguments, OR the same
literal appears in more than one place without a named identifier, OR a bare numeric
value is used as a threshold/timeout/limit without a name.

---

### Q8 — Iterate the full set of variants [BLOCKING when variants are the unit of test]

When code asserts a property across a set of variants (types, providers, OS, distros,
language versions), and the variants are the unit-of-test (the reason they exist is
that they may behave differently), the assertion must iterate over every variant in
scope. Asserting on one and trusting the others erases the value of having multiple
variants.

**Fires when:** N variants are created in setup but a verification step references only
a strict subset, with no explicit reason given for the omission.

---

### Q9 — Function/block length [WARN]

A function or block exceeding ~80-100 lines is a structural smell. Extract per-concern
helpers; the parent reads as a sequence of named operations.

**Fires when:** an added function/block body exceeds 80 lines, OR a modified function
grows by ≥40 lines in this diff.

**Two-stage check:** the line-count flag fires here; the gap-analysis subagent (Step 9)
decides whether the length is justified or whether the function mixes unrelated concerns
and should be extracted.

---

### Q10 — No inline copy-paste [WARN]

Two near-identical blocks (`if`/`else` branches, YAML steps, shell stanzas) that differ
in 1-3 fields should be parameterised, not duplicated. The duplicated structure rots
independently and reviewers can't easily see what differs.

**Fires when:** the diff contains two or more added blocks of ≥8 lines that share ≥85%
of their tokens after whitespace and identifier normalisation.

---

### Q11 — Don't split a single concern across languages [WARN]

When the same logic exists in both workflow YAML/shell and the application code (version
resolution, environment setup, source selection, state sniffing), one is the source of
truth and the other should be a thin pass-through. Sniffing for state set up by the
*other* language is split-brain — make the contract explicit (env var, file marker,
file-existence check) or move the concern into one place.

**Fires when:** the same decision/computation appears in both shell/YAML and code in
the diff, OR code reads runtime state set by an out-of-band setup step without an
explicit contract.

---

### Q12 — Names must describe what the value actually means [WARN; BLOCKING when misuse follows]

Identifier names are a contract. A boolean named `needsXForY` that actually gates X, Y,
and Z is a lying name and will mislead every future reader. A name describing a narrower
domain than the value's actual usage signals either an incorrect implementation or an
outdated rename.

**Fires when:** an added identifier's name describes a narrower domain than the value's
actual usage in the diff.

**Severity escalation:** WARN by default. Becomes BLOCKING when the misleading name
leads to incorrect call-site behaviour (e.g. a flag whose name says "enable X" but
actually toggles Y, leading to misuse).

---

### Q13 — Use the simplest primitive that fits; no pointless ceremony [WARN]

Use the simplest construct that satisfies the need. A concurrency primitive used with a
single worker, an "ordered" grouping without ordering dependencies, a retry wrapper
around an already-blocking call, a helper that wraps a single one-liner — all add
ceremony without meaning.

**Fires when:** an added construct's distinguishing feature is unused — parallelism
without parallel work, ordering without dependencies, retry without async, helper
indirection without reuse.

---

### Q14 — No double-timeout layering [BLOCKING]

When an outer wait wraps an inner operation that already polls or has its own deadline
(e.g. a retry loop around `kubectl wait --for=condition=X --timeout=Ns`, or around an
SDK call with its own context deadline), pick one. Two layers means the effective
timeout is the outer's and the inner's `--timeout` is a misleading floor that hides real
failure modes.

**Fires when:** an added retry/poll wrapper invokes an inner operation that already has
a `--timeout` flag, a retry loop, or built-in polling.

---

### Q15 — Use the SDK if you already imported it [WARN]

If the code imports an SDK for a tool (kubectl / helm / docker / a cloud vendor) and
uses it elsewhere, shell-outs to those binaries for the same operations are inconsistent
and fragile (PATH, flags, output parsing, error visibility). Pick one abstraction layer
and stick to it.

**Fires when:** an added shell-out to a CLI has an equivalent already-imported SDK call
available.

---

### Q16 — Doc and code must agree [BLOCKING for net-new docs; WARN for pre-existing drift]

Top-of-block doc-comments that enumerate counts, names, or numbered sequences ("9 specs:
1-9", "supports 5 providers") must match the code immediately following. Drift in
counting confuses readers and grows with every future change.

**Fires when:** an added doc-comment lists a count or numbered sequence that doesn't
match the code immediately following.

**Severity:** BLOCKING when the PR introduces a fresh mismatch (added doc doesn't match
added code). WARN when pre-existing drift is exposed but not introduced by the PR.

---

### Q17 — No tight coupling to opaque output formats [WARN]

Hardcoding output filenames, regex captures of CLI output, or paths derived from package
internals (e.g. `app-<version>.tgz` derived from a `name:` field defined elsewhere)
couples the code to non-contract details. Derive from actual command output (parse
stdout, list dir) so renames don't fail silently.

**Fires when:** an added literal mirrors a value defined in a sibling file (a manifest,
package descriptor, etc.) without explicit reference, OR a literal matches CLI output
format that isn't documented as stable.

---

### Q18 — Scope discipline: diff matches stated purpose [WARN]

The diff should contain only changes that serve the PR's stated purpose. Unrelated
cleanups (timeout bumps in unrelated spots, comment fixes in unrelated files, helper
refactors not used by the stated change) should be split out. Bundling makes review
harder, dilutes blame, and forces reviewers to either approve unrelated work or block
on it.

**Fires when:** an added/modified file is touched without a clear connection to the PR
title/description, OR a coverage-table row would not be served by the change.

---

### Q19 — No new abstractions without consumers [BLOCKING]

A new exported function, option, type, constant, or workflow input must be used by at
least one call site that's also added or modified in this diff. Speculative API surface
("we'll need this later") rots before its second consumer arrives, drifts from the
canonical pattern, and signals the change isn't really about the stated work.

**Fires when:** the diff adds an exported identifier (capitalised name / workflow input
/ CLI flag) and grep across the diff finds zero callers in changed files.

---

### Q20 — Logs must surface state changes that affect outcome [WARN]

When code branches based on runtime state (resource exists → use it vs fall back; OS
gate; version gate; empty-vs-set env var), the chosen path must be logged so post-mortem
doesn't require re-running with verbose mode. Silent fallbacks hide that the wrong path
ran.

**Fires when:** an added conditional changes behaviour (source selection, skip path,
fallback, error treatment) without an accompanying log line describing the choice.

---

### Q21 — External resources must pin a version [BLOCKING]

URLs, image refs, chart refs, and scripts pulled from registries, releases, or repos
must specify an explicit version or tag pin. `latest`, `main`, or `master` are
acceptable only with a comment justifying why a moving target is intentional.

**Fires when:** an added URL/image/chart reference uses `latest`, no tag, or a branch
name (vs a release tag, semver, or sha pin).

---

### Q22 — Type assertions must be ok-checked or guarded [BLOCKING]

`x := y.(T)` panics on type mismatch with a confusing trace. Use `x, ok := y.(T)` and
fail loudly with context, OR pre-assert non-nil-and-correct-type before the cast. The
single-return form is acceptable only for assertions on values you've just placed
yourself in the same scope. (Language-specific: applies to any unchecked downcast.)

**Fires when:** an added single-return type assertion (`x.(T)`) reads from a context
value, a `map[string]any`, an unstructured object, or any source not statically
guaranteed by the immediate prior line.

---

### Q23 — No fixed sleeps in test logic [WARN]

Hardcoded sleeps are the canonical flake source. Poll for the actual condition, watch
for it, or — if neither is feasible — document why a specific delay is unavoidable.
Sleep is acceptable as the polling interval inside a retry/poll loop, or in a cleanup
retry already handled by framework helpers.

**Fires when:** an added fixed sleep (`time.Sleep`, `sleep N`, `setTimeout` as a barrier)
sits outside a polling loop and acts as a barrier before the next step.

---

### Q24 — TODO/FIXME requires a tracking link or owner [BLOCKING for net-new TODOs]

A TODO/FIXME comment in added code must include an issue link or an explicit owner. Bare
TODOs without context are deferred indefinitely and ship by accident.

**Fires when:** an added line contains `TODO` or `FIXME` and the same comment block
contains no `https://` URL, ticket ID (`#123`, `PROJ-456`), or `@owner` mention.

---

### Q25 — Privileged operations must justify themselves [WARN]

Code that creates privileged pods, mounts host paths, runs `sudo`, modifies host
firewall rules, uses `--insecure` flags, or escalates beyond its own credentials should
explain why the privilege is necessary in a comment near the call site. Without it,
future maintainers can't tell whether the privilege is load-bearing or vestigial — and
security review can't bound the blast radius.

**Fires when:** an added line contains `Privileged: true`, `sudo `, host-path mounts,
`--insecure`, `InsecureSkipTLSVerify`, or similar escalation, and the surrounding 5
lines have no comment describing why.

---

### Q26 — Local constant aliases for existing named constants are redundant [WARN]

A `const` block that re-names existing package-level constants (e.g.
`fooTimeout = constants.PollingTimeoutLong`) adds indirection without meaning. It hides
that two "different" names are the same value, forces readers to look up two levels of
naming, and diverges from the established pattern in sibling files.

**Fires when:** an added `const` block contains entries whose right-hand side is a
reference to an existing named constant rather than a literal value or a computation that
meaningfully restricts the domain.

**Exception:** acceptable when the local constant carries genuine business context not
present in the generic name — but even then prefer a concrete literal + WHY comment over
aliasing.

---

### Q27 — Reference parity: match verification steps, not just structure [WARN]

When a reference implementation is named (by the user or PR description), read it
line-by-line and extract every verification step it performs — every log step, wait,
assertion, and readiness check. Then verify each one is present in the new code or
explicitly omitted with justification.

Matching the reference's block structure (same setup/exercise/teardown shape, same
ordering) is not sufficient. The gaps live in the verification content, not the skeleton.

**Fires when:** a PR names a reference file and the new code matches the reference's
shape but omits one or more specific wait or assertion steps, with no stated reason.

**How to apply:** for each wait / assertion in the reference, ask "does the new code
have an equivalent step?" Flag every missing one.

---

### Q28 — Soft-TODO comments need a tracking link or owner [WARN]

A comment that defers cleanup, replacement, or follow-up work is a TODO even when it
doesn't use the literal token `TODO` / `FIXME`. Phrasings like "after X lands", "until X
ships", "switch to Y when …", "remove once …", "pending X", "temporarily …" all describe
deferred work. Without a ticket link or owner, they outlive whatever they were waiting
for, and the next reader can't tell whether the workaround is still load-bearing.

**Fires when:** an added comment contains a deferral phrase (`after …`, `until …`,
`when …`, `once …`, `pending …`, `temporarily …`, `for now`, `remove when …`,
`revisit …`) referring to future work, AND the same block contains no ticket ID,
`https://` URL, or `@owner` mention.

**Examples:**

FAIL:
```go
// Use the old chart — switch to the new one after the chart-name bug fix.
// HACK: drop the polling once the controller fix ships.
// Pin to v1.29 until the upstream regression is patched.
```

PASS:
```go
// Pin to v1.29 until PROJ-4521 lands.
// Drop this branch once https://github.com/foo/bar/issues/2310 closes.
// Temporarily disabled — @alice owns the re-enable next sprint.
```

**Related:** Q24 fires on the literal `TODO` / `FIXME` token; Q28 catches the same intent
in natural language. A single comment can violate both — fix once, don't double-count.

---

### Q29 — PR description must match the diff [WARN]

The PR description is part of the review surface, not external documentation. When the
body enumerates counts, named items, or a matrix ("N specs", "covers X / Y / Z", "test
matrix: N pairs"), each enumerated entity must exist in the diff. Drift between
description and diff means reviewers comparing the two see a mismatch — either the
description is stale or scope shrank silently and the author didn't update the body.

**Fires when:** the PR body contains a count noun phrase ("9 specs", "6 providers"), an
enumerated list ("covers / supports / handles X, Y, Z"), a numbered roadmap, or a
tabular matrix — AND the diff does not contain a matching number of test blocks,
matching named items, or matching matrix rows.

**Examples:**

FAIL:
- Body: *"Adds 9 specs covering lifecycle"* — diff has 6 test blocks.
- Body: *"Supports providers AWS, GCP, Azure"* — diff implements only AWS.

PASS:
- Body: *"Adds 6 specs covering …"* — diff contains exactly 6 test blocks.
- Body: *"Supports AWS (Azure follow-up tracked in PROJ-XXXX)"* — diff implements AWS
  only and the omission is explicit.

**How to apply:** when reading the PR body in Step 2, extract every count phrase and
"covers / supports / handles" list. Cross-check each against the diff. Pre-existing prose
carried over from a template doesn't count — only enumerations the author added or edited.

---

### Q30 — Input / flag / value descriptions must match behavior [WARN]

Descriptions on workflow inputs, config values, and CLI flags are documentation users
read before passing a value. When the description claims a default behavior (`Empty = X`,
`Defaults to Y`, `If unset, auto-detects Z`), the matching code in the same diff must
implement that behavior. Stale descriptions mislead the next operator and hide that a
fallback was removed.

**Fires when:** a changed file contains an input / flag / value description with
`Empty =`, `Defaults to`, `Default:`, `If unset`, `When unset`, `Optional` (with claimed
fallback), or `auto-detect*` wording — AND the diff removes, fails to add, or contradicts
the corresponding fallback / default logic.

**Examples:**

FAIL — input claims auto-detect but the fallback was deleted:
```yaml
inputs:
  base-version:
    description: "Empty = latest stable from registry"
    required: false
```
…paired with a code change that removed the `if baseVersion == "" { lookupLatest() }`
block.

PASS — description matches the new contract:
```yaml
inputs:
  base-version:
    description: "Required. The stable version to install as the base."
    required: true
```

**How to apply:** grep changed config / values / flag definitions for the keyword set
above. For each match, follow the input through the changed code and verify the claimed
behavior is implemented. The rule cuts both ways — description added without code, and
code removed without updating the description.
