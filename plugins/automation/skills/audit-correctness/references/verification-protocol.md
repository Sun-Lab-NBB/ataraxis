# Verification protocol

The two checks every candidate finding of `/audit-correctness` passes after the guards and before the
report, together with the triage header the report opens with.

Both checks are external. Check 1 tests the finding against the source file, and Check 2 tests it
against a reader who never saw the sweep. Re-reading your own reasoning is no substitute for either,
because that reasoning is the thing under test.

---

## Contents

- Check 1: Citation verification
- Check 2: Adversarial refutation
- The triage header
- Confidence placement in the report
- The report's own prose

---

## Check 1: Citation verification

Runs against EVERY surviving finding, with no sampling.

For each finding, open the cited `<path>` at the cited line range and confirm that every quoted string
appears there character for character. The strings to check are the Contract quote, the Implementation
quote, and every line number named inside the Trigger and inside the Result.

Delete the finding when a quote does not appear at the cited location, and when the cited line holds
different code. Repairing the citation is FORBIDDEN here. A citation that drifted is evidence that the
finding was assembled from recollection rather than from the file, which makes the claim resting on it
unreliable for the same reason. A deleted finding is free to be re-derived from scratch in a later
audit.

Record the count of findings checked and the count deleted.

---

## Check 2: Adversarial refutation

Runs against every CRITICAL and HIGH finding that survived Check 1. Running it after Check 1 means a
finding deleted for a bad citation never consumes a sub-agent.

Spawn one `general-purpose` sub-agent per finding. Give it the finding text, the paths the finding
cites, and nothing else from the sweep. A fresh context is what makes the check independent, so
handing over the ledgers, the pass output, or the reasoning that produced the finding defeats it.

Give the sub-agent this task:

```text
Refute this audit finding by reading the cited source. Return REFUTED when any of these holds:
- No reachable path supplies the stated trigger to the cited line
- A guard that dominates the cited line excludes the trigger value
- The traced result differs from the result the finding claims
- The cited contract does not state what the finding says it states
- The cited line does not do what the finding says it does
Return CONFIRMED only after tracing the trigger to the cited line yourself and reaching the stated
result. Answer REFUTED whenever you are uncertain.
```

Discard every refuted finding. Record the counts of findings put through this check, confirmed, and
refuted.

A refuted CRITICAL or HIGH finding is discarded rather than demoted, because the refutation attacked
the trigger and the result, which are the same evidence a lower severity would rest on.

---

## The triage header

The report opens with this header, above the coverage ledger, so a reader sees the shape of the report
before reading any finding:

```text
Findings: <total> reported

| Severity | HIGH confidence | MEDIUM confidence | LOW confidence |
|----------|-----------------|-------------------|----------------|
| CRITICAL | <n>             | <n>               | <n>            |
| HIGH     | <n>             | <n>               | <n>            |
| MEDIUM   | <n>             | <n>               | <n>            |
| LOW      | <n>             | <n>               | <n>            |

Discarded by the false-positive guards: <n>
Deleted by citation verification: <n> of <n> checked
Refuted by adversarial verification: <n> of <n> checked
```

These counts are the audit's own precision record. A run that discards nothing at any stage has either
found an unusually clean file set or skipped the stage, and stating the numbers is what lets a reader
tell those apart.

---

## Confidence placement in the report

HIGH and MEDIUM confidence findings occupy the body of the report, grouped file, then category, then
severity, ordered most severe first.

LOW confidence findings go into one trailing section titled `Appendix: LOW confidence`, ordered most
severe first, rather than interleaved into the file groups. Every finding there still carries its full
evidence and still passed every guard and both checks above. The appendix exists so the body of the
report reads at one confidence level, and so a reader who wants only the settled findings knows where
to stop.

---

## The report's own prose

Fill each authored line to 120 characters before breaking it, under the wrap width rule
`/python-style` defines, so a line ending before column 100 while its next word would still fit is
re-flowed. A line ending early because the sentence ends, or because it holds a table row, a list item,
or a code span, is already correct. Verbatim quotes, triggers, and line-numbered interleavings are
exempt, because they are copied rather than written.
