# Execution plan

The shared-context schema `/audit-project` builds once, the wave definitions, and the sub-agent budget
that keeps a four-audit run from multiplying its own instruction payload.

---

## Contents

- The shared context
- Wave definitions
- Parallelize at exactly one level
- Sub-agent budget
- What each audit receives

---

## The shared context

Built once on the main agent in Step 1, written to the session scratch directory, and handed to every
audit as the output of the discovery steps it would otherwise run. Those steps are Step 1 for
`/audit-facts` and `/audit-style`, Steps 1 through 3 for `/audit-correctness`, and Steps 1 and 2 for
`/audit-performance`.

```text
inventory:
  <path>: {language, kind, authority, coverage_tier, in_change_set}
prerequisites:
  archetype:        <full python | reduced python | cpp extension | cpp embedded | cpp docs-only | unity>
  test_matrix:      <the tox envlist members, or N/A>
  coverage:         {branch, fail_under, omit}          # Python targets
  numpy_pin:        <version, and NEP 50 or legacy regime>   # Python targets
  numba:            <present or absent>                  # Python targets
  cpp_archetype:    <embedded or extension, plus target boards>
  csharp:           <unity or plain, the target framework, plus test assembly locations>
  boundary_widths:  <the fixed-width types the wire, packed-struct, and on-disk schemas declare>
  codegraph:        <present or absent>
  test_suite:       <location and shape, per language>
ledgers:
  contract:  <one row per callable, class, and module, tagged NAME | SIG | ANN | DOC | ENF>
  state:     <one row per stateful attribute, with creator, readers, destroyers>
  callgraph: <one row per call site, with callee, in-repo or external, return consumed>
gate_diagnostics:
  <populated by /audit-style in wave 1, read by wave 2>
```

The `kind` field takes one of `source`, `metadata-doc`, `build-config`, `test`, or `generated`, and
Step 2's routing reads it. The `authority` field takes the style skill the file binds to, and the
audits' batching rules read it.

Every audit still RECORDS the prerequisites, because its own verification checklist requires it. It
records what this context hands it rather than deriving them again.

---

## Wave definitions

### Wave 1: `/audit-facts` and `/audit-style`

Concurrent. Neither reads the other's output, and their boundary is stated in both skills: a claim
inside a documentation block belongs to facts, and the form of that block belongs to style.

Wave 1 runs first for two reasons. `/audit-correctness` adjudicates its ownership ladder against the
documentation verdicts facts produces, and `/audit-style` runs the deterministic gates whose
diagnostics wave 2 reads instead of re-deriving.

Collect two outputs into the shared context before wave 2 starts:

- The DRIFT and WRONG verdicts, keyed by `<path>:<line>`, which `/audit-correctness` consumes
- The gate diagnostics, keyed by `<path>:<line>` and rule code

### Wave 2: `/audit-correctness` and `/audit-performance`

Concurrent. Both consume the shared context, and neither consumes the other's output. What sits
between them is a ROUTING rule rather than a data dependency, so it is settled at merge time.

The three routing rules that touch both:

| Construct                        | Owner                | Rule                                         |
|----------------------------------|----------------------|----------------------------------------------|
| A `prange` cross-iteration race  | `/audit-correctness` | Performance Guard 1 carries it only as a tag |
| A missed timing contract         | `/audit-correctness` | Correctness Guard 8's single exception       |
| Cost, allocation, and complexity | `/audit-performance` | Correctness Guard 8                          |

---

## Parallelize at exactly one level

Each audit already fans out over file batches, and each sub-agent re-receives an instruction payload.
Nesting a wave fan-out inside a batch fan-out multiplies the payload by both factors, and a sub-agent
frequently cannot spawn a further sub-agent at all.

| Target size      | Level | Execution                                                              |
|------------------|-------|------------------------------------------------------------------------|
| Under 10 files   | WAVE  | One sub-agent per audit, two per wave, each audit working sequentially |
| 10 or more files | BATCH | Audits one at a time in wave order on the main agent, each batching    |

Tell every wave-level sub-agent that it runs at Small or Medium tier regardless of what its own tier
table would select, because the parallelism is already spent at the wave level.

Tell every batch-level audit that it runs at its own tier and fans out normally.

---

## Sub-agent budget

Two limits govern every run, and they are not the same limit.

| Limit     | Value | Meaning                                                     |
|-----------|-------|-------------------------------------------------------------|
| In flight | 12    | Sub-agents running at one moment, across the whole run      |
| Total     | 40    | Sub-agents started over the whole run, across every audit   |

| Level | Audit sub-agents                              | Verification sub-agents      |
|-------|-----------------------------------------------|------------------------------|
| Wave  | Up to 2 per wave, one per audit the wave runs | One per top-severity finding |
| Batch | One audit at a time, up to 12 alive           | One per top-severity finding |

A wave-level run spends two of the twelve in-flight slots on the audits themselves, which leaves ten
for the verification refutations each audit spawns in its own verification step. Queue whatever
exceeds twelve and start each as a slot frees, because the in-flight limit paces a run rather than
truncating it.

The total is a budget rather than a coverage limit. Where an audit's batching rules produce more units
than the total allows, merge units that share an authority or a checklist until they fit, and record
the merge in that audit's coverage ledger. Dropping files to reach the cap is a coverage error.

---

## What each audit receives

Hand every audit exactly this, and nothing more:

1. The shared context, with the inventory filtered to the files that audit will cover
2. The change-set narrowing, in change mode only, naming the base revision
3. The instruction that its Step 0 plan is satisfied and that it MUST NOT pause for confirmation
4. Its tier, fixed by the parallel level rather than by its own tier table
5. Wave 1 verdicts, for `/audit-correctness` alone
6. The gate diagnostics, for wave 2 alone

Sending an audit inventory rows, ledger rows, or verdicts for files it does not cover wastes the
payload it pays for, which is the same rule each audit applies to its own batches.
