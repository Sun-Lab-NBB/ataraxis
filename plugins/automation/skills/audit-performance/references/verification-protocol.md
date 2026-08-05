# Verification protocol

The two checks every candidate finding of `/audit-performance` passes after the guards and before the report, together
with the triage header the report opens with.

Both checks are external. Check 1 tests the finding against the source file, and Check 2 tests it against a reader who
never saw the sweep. Re-reading your own reasoning is no substitute for either, because that reasoning is the thing
under test.

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

For each finding, confirm two things by opening the files it names.

**The quote.** Open the cited `<path>` at the cited line range and confirm the quoted current state appears there
character for character.

**The multiplicity source.** Open the `source: <path>:<line>` the Multiplicity line names and confirm that line holds
the loop header or the call site the finding attributes the heat to, and that its bound expression is the one the
finding quotes. A multiplicity whose source line holds something else is an unproven heat claim wearing a citation.

A PUBLIC_API finding cites the `__all__` entry that exports the symbol in place of a call site, so confirm that entry
appears in the distribution's top-level `__init__.py` and that the finding rests on per-call cost. A PUBLIC_API finding
that asserts a downstream call frequency fails this check, since Guard 2 leaves that assertion disqualifying.

Delete the finding when either check fails. Repairing the citation is FORBIDDEN here. A citation that drifted is
evidence that the finding was assembled from recollection rather than from the file, which makes the cost arithmetic
resting on it unreliable for the same reason. A deleted finding is free to be re-derived from scratch in a later audit.

Record the count of findings checked and the count deleted.

---

## Check 2: Adversarial refutation

Runs against every HIGH impact finding that survived Check 1. Running it after Check 1 means a finding deleted for a bad
citation never consumes a sub-agent.

Spawn one `general-purpose` sub-agent per finding. Give it the finding text, the paths the finding cites, and nothing
else from the sweep. A fresh context is what makes the check independent, so handing over the multiplicity table, the
pass output, or the reasoning that produced the finding defeats it.

Give the sub-agent this task:

```text
Refute this audit finding by reading the cited source. Return REFUTED when any of these holds:
- The cited call sites do not establish the claimed multiplicity, or resolve only inside tests, unless the finding is
  marked PUBLIC_API, which rests on the per-call cost of an exported symbol rather than on a call site
- The finding is marked PUBLIC_API and its cost arithmetic depends on how often a downstream project calls the symbol
- The loop bound is a literal, an enum length, or a configuration value in the low tens
- The cost arithmetic does not follow from the shapes, dtypes, and trip counts in the source
- The construct already carries the form the finding proposes, or a guarded equivalent of it
- The proposed fix would change the result the current code produces
Return CONFIRMED only after tracing the multiplicity to a call site yourself and re-deriving the cost
arithmetic. For a PUBLIC_API finding, trace the export instead and re-derive the cost of a single call.
Answer REFUTED whenever you are uncertain.
```

Discard every refuted finding. Record the counts of findings put through this check, confirmed, and refuted.

A refuted HIGH finding is discarded rather than demoted to MEDIUM, because the refutation attacked the multiplicity and
the cost arithmetic, which are the same evidence a lower impact rating would rest on.

---

## The triage header

The report opens with this header, above the coverage ledger, so a reader sees the shape of the report before reading
any finding:

```text
Findings: <total> reported, <n> STATIC and <n> MEASUREMENT-PENDING

| Impact | HIGH confidence | MEDIUM confidence | LOW confidence |
|--------|-----------------|-------------------|----------------|
| HIGH   | <n>             | <n>               | <n>            |
| MEDIUM | <n>             | <n>               | <n>            |
| LOW    | <n>             | <n>               | <n>            |

Discarded by the false-positive guards: <n>
Deleted by citation verification: <n> of <n> checked
Refuted by adversarial verification: <n> of <n> checked
```

These counts are the audit's own precision record. A run that discards nothing at any stage has either found an
unusually clean file set or skipped the stage, and stating the numbers is what lets a reader tell those apart.

---

## Confidence placement in the report

HIGH and MEDIUM confidence findings occupy the body of the report, grouped file, then category, then impact, with the
STATIC section ahead of the MEASUREMENT-PENDING section.

LOW confidence findings go into one trailing section titled `Appendix: LOW confidence`, ordered by impact, rather than
interleaved into the file groups. Every finding there still carries its full evidence and still passed every guard and
both checks above. The appendix exists so the body of the report reads at one confidence level, and so a reader who
wants only the settled findings knows where to stop.

---

## The report's own prose

Hold the report's own prose to the documentation-quality rules this family enforces. Keep every sentence in a Wrong,
Fix, Impact, or Choice bullet under 40 words. Separate clauses with full stops and commas rather than semicolons or
em-dashes, and state what the fix does rather than what the code fails to do. Verbatim quotes and cost arithmetic are
exempt, because they are copied rather than written.

Fill each authored line to 120 characters before breaking it, under the wrap width rule `/python-style` defines, so a
line ending before column 100 while its next word would still fit is re-flowed. A line ending early because the sentence
ends, or because it holds a table row, a list item, or a code span, is already correct.

Before presenting, split the report's authored fields on sentence boundaries, excluding every verbatim quote and every
arithmetic or complexity expression, and count the words in each sentence. Any sentence over 40 words, and any semicolon
or em-dash joining two independent clauses, is rewritten before the report is handed over.
