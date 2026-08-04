# Verification protocol

The two checks every candidate finding of `/audit-style` passes after the guards and before the
report, together with the triage header the report opens with.

Both checks are external. Check 1 tests the finding against the checklist and the file it cites, and
Check 2 tests it against a reader who never saw the sweep. Re-reading your own reasoning is no
substitute for either, because that reasoning is the thing under test.

A finding produced by a Step 3 deterministic gate skips both checks. The tool that produced it already
is the external check, and it cites a rule code rather than a checklist quote.

## Contents

- Check 1: Citation verification
- Check 2: Adversarial refutation
- The triage header
- The coverage ledger
- Confidence placement in the report

---

## Check 1: Citation verification

Runs against EVERY surviving sweep finding, with no sampling.

Each finding carries two citations, and both are checked.

**The checklist point.** Find the quoted rule in the loaded checklist or reference file the finding
names and confirm it appears there character for character. Confirm the quote is the WHOLE rule rather
than a clause of it, because a rule truncated before its exemption reverses its meaning.

**The current state.** Open the cited `<path>` at the cited line range and confirm the Current state
quote appears there character for character. For a collapsed finding carrying several line citations,
check every line the finding lists.

**The consumer set**, for a Pass 11 finding alone. Re-run the repository-wide search the finding names
and confirm it returns what the finding says it returned. This citation is checked by RUNNING rather
than by reading, because the claim is about the whole repository rather than about the cited line, and
a stale search is how a symbol that acquired a caller last week is reported as unused today.

Delete the finding when either quote fails to appear at its cited location. Repairing the citation is
FORBIDDEN here. A citation that drifted is evidence that the finding was assembled from recollection
rather than from the checklist, which makes the rule application resting on it unreliable for the same
reason. A deleted finding is free to be re-derived from scratch in a later audit.

Record the count of findings checked and the count deleted.

---

## Check 2: Adversarial refutation

Runs against every BLOCKING and CONFLICT finding that survived Check 1. Those two categories are the
ones that stop a release or accuse the checklists of disagreeing, so they carry the cost of being
wrong, and running the check after Check 1 means a finding deleted for a bad citation never consumes a
sub-agent.

Spawn one `general-purpose` sub-agent per finding. Give it the finding text, the cited file, the cited
checklist, and nothing else from the sweep. A fresh context is what makes the check independent, so
handing over the rule ledger or the reasoning that produced the finding defeats it.

Give the sub-agent this task:

```text
Refute this style finding by reading the cited checklist and the cited file. Return REFUTED when any
of these holds:
- The cited rule does not say what the finding says it says, or an exemption in it covers this case
- The cited rule is not BLOCKING, meaning the checklist neither says MUST nor says it blocks release
- The construct is a different kind than the finding assumed, so a different rule governs it
- The construct sits in a generated block, a vendored tree, or a test file the configuration relaxes
- For a CONFLICT, the two cited rules can both be satisfied at once
- For a symbol usage finding, a consumer exists that the stated Consumer set missed. Search the
  repository yourself rather than trusting the finding's search, and treat a runtime registration, an
  interface the symbol implements, and an entry in the top-level `__all__` as consumers
Return CONFIRMED only after reading the full rule including its exemptions and confirming the cited
line breaks it. Answer REFUTED whenever you are uncertain.
```

Discard every refuted finding. A refuted BLOCKING finding is discarded rather than demoted to
STANDARD, because the refutation attacked whether the rule applies at all. Record the counts of
findings put through this check, confirmed, and refuted.

---

## The triage header

The report opens with this header, so a reader sees the shape of the report before reading any
finding:

```text
Findings: <total> reported, <n> from tools and <n> from the sweep

| Category      | HIGH confidence | MEDIUM confidence | LOW confidence |
|---------------|-----------------|-------------------|----------------|
| BLOCKING      | <n>             | <n>               | <n>            |
| INCONSISTENCY | <n>             | <n>               | <n>            |
| CONFLICT      | <n>             | <n>               | <n>            |
| STANDARD      | <n>             | <n>               | <n>            |

Deterministic gates run: <tool list>
Deterministic gates configured but unable to run: <tool list, with the rules left unchecked>
Discarded by the false-positive guards: <n>
Deleted by citation verification: <n> of <n> checked
Refuted by adversarial verification: <n> of <n> checked
```

These counts are the audit's own precision record, and the tool lines are what let a reader tell a
clean result from an unrun gate. A run that discards nothing at any stage has either found an
unusually compliant file set or skipped the stage, and stating the numbers is what lets a reader tell
those apart.

---

## The coverage ledger

The ledger follows the triage header and records what was swept, so a thin pass is visible rather than
silent:

```text
| Binding        | Files in scope | Files swept | Files UNAUDITED |
|----------------|----------------|-------------|-----------------|
| /python-style  | 47             | 47          | 0               |
| /readme-style  | 1              | 1           | 0               |
| (no binding)   | 3              | 0           | 3               |
```

Alongside the table, state each of the following:

- Every UNAUDITED file by path, with the binding-table reason it matched no checklist.
- The sub-agent and batch count the run used, recorded as `1 (main agent)` for a Small or Medium tier.
- The deterministic gates that ran and the gates that failed to run, naming each tool.
- The project-scope layout pass status, as `run`, `skipped-not-a-project-root`, or
  `skipped-no-created-or-deleted-files`.
- The symbol usage pass status, as `run`, `run-partial` naming the tiers left unjudged under a
  package-directory target, or `skipped-reference-table-incomplete`. State the count of declared
  symbols and the count of reference sites the pass reconciled, because those two numbers are what
  separate a reconciliation that ran from one that reported nothing because it saw nothing.
- The revision the run resolved against, whenever the scope was narrowed to a change set.

A Large-tier audit that produced no findings still reports a non-zero swept count, which distinguishes
compliant files from an unrun pass.

---

## Confidence placement in the report

HIGH and MEDIUM confidence findings occupy the body of the report, grouped file, then category, with
the categories ordered BLOCKING, INCONSISTENCY, CONFLICT, STANDARD.

LOW confidence findings go into one trailing section titled `Appendix: LOW confidence`, ordered by the
same category sequence, rather than interleaved into the file groups. Every finding there still
carries its verbatim checklist quote and still passed every guard and both checks above. The appendix
exists so the body of the report reads at one confidence level, and so a reader who wants only the
settled findings knows where to stop.
