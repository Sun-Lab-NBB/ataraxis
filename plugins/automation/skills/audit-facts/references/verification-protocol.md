# Verification protocol

The two checks every candidate finding of `/audit-facts` passes after the guards and before the
report, together with the triage header the report opens with.

Both checks are external. Check 1 tests the finding against the two files it cites, and Check 2 tests
it against a reader who never saw the verification pass. Re-reading your own reasoning is no
substitute for either, because that reasoning is the thing under test.

---

## Contents

- Check 1: Citation verification
- Check 2: Adversarial refutation
- The triage header
- Confidence placement in the report

---

## Check 1: Citation verification

Runs against EVERY surviving finding, with no sampling.

Each finding carries two citations, and both are checked.

**The claim.** Open the documentation file at `Location in file` and confirm the Claim quote appears
there character for character.

**The source reality.** Open the source file at `Location in source` and confirm the Source reality
quote appears there character for character. Where the finding gives a factual summary rather than a
quote, confirm the cited line supports that summary by reading it.

An UNVERIFIABLE finding carries no source citation, so only its claim quote is checked. Its record of
what was searched and where is checked instead by re-running one of the searches it names.

Delete the finding when either quote fails to appear at its cited location. Repairing the citation is
FORBIDDEN here. A citation that drifted is evidence that the finding was assembled from recollection
rather than from the file, which makes the mismatch resting on it unreliable for the same reason. A
deleted finding is free to be re-derived from scratch in a later audit.

Record the count of findings checked and the count deleted.

---

## Check 2: Adversarial refutation

Runs against every WRONG and CONTRADICTION finding that survived Check 1. Those two verdicts assert that
the documentation states something the source denies, which is the assertion most worth attacking, and
running the check after Check 1 means a finding deleted for a bad citation never consumes a sub-agent.

Spawn one `general-purpose` sub-agent per finding. Give it the finding text, the paths the finding
cites, and nothing else from the verification pass. A fresh context is what makes the check
independent, so handing over the claim ledger or the reasoning that produced the finding defeats it.

Give the sub-agent this task:

```text
Refute this documentation finding by reading both cited files. Return REFUTED when any of these holds:
- The implementation satisfies the claim through different wording, which is a match rather than a
  mismatch
- The claim is satisfied deeper in the body, or by a helper the documented callable delegates to
- The named symbol resolves through a re-export, so the module the documentation names does hold it
- The claim states an authoritative external requirement the repository cannot confirm or deny
- The two sides of a claimed contradiction can both be true
Return CONFIRMED only after reading the full body or the full file yourself and finding nothing that
satisfies the claim. Answer REFUTED whenever you are uncertain.
```

Discard every refuted finding. Record the counts of findings put through this check, confirmed, and
refuted.

A refuted WRONG finding is discarded rather than demoted to DRIFT, because the refutation attacked
whether any mismatch exists rather than when it arose.

---

## The triage header

The report opens with this header, above the coverage ledger, so a reader sees the shape of the report
before reading any finding:

```text
Findings: <total> reported, from <n> claims verified

| Verdict       | HIGH confidence | MEDIUM confidence | LOW confidence |
|---------------|-----------------|-------------------|----------------|
| WRONG         | <n>             | <n>               | <n>            |
| DRIFT         | <n>             | <n>               | <n>            |
| CONTRADICTION | <n>             | <n>               | <n>            |
| OMISSION      | <n>             | <n>               | <n>            |
| UNVERIFIABLE  | <n>             | <n>               | <n>            |

Claims resolved EXACT or SEMANTIC and therefore unreported: <n>
Discarded by the false-positive guards: <n>
Deleted by citation verification: <n> of <n> checked
Refuted by adversarial verification: <n> of <n> checked
```

These counts are the audit's own precision record, and the EXACT and SEMANTIC total is what shows how
much of the documentation the audit confirmed rather than merely read. A run that discards nothing at
any stage has either found unusually accurate documentation or skipped the stage, and stating the
numbers is what lets a reader tell those apart.

---

## Confidence placement in the report

HIGH and MEDIUM confidence findings occupy the body of the report, grouped by documentation class,
then file, then finding verdict, with the verdicts ordered WRONG, DRIFT, CONTRADICTION, OMISSION,
UNVERIFIABLE.

LOW confidence findings go into one trailing section titled `Appendix: LOW confidence`, ordered by the
same verdict sequence, rather than interleaved into the file groups. Every finding there still carries
its full evidence and still passed every guard and both checks above. The appendix exists so the body
of the report reads at one confidence level, and so a reader who wants only the settled findings knows
where to stop.
