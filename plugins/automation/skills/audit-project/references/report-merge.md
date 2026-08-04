# Report merge

How `/audit-project` combines four reports into one: the collision rules that decide which audit owns
a construct two of them found, the deduplication rules, and the merged triage header.

This step removes duplicates and settles ownership. It NEVER re-rates a finding, because severity,
impact, confidence, and evidence belong to the audit that derived them.

---

## Contents

- The collision rules
- Deduplication
- The merged triage header
- The combined coverage ledger
- Report ordering

---

## The collision rules

Four collisions exist. Each audit already states its own side, and this table is where a caller sees
both sides at once.

### Collision 1: A documentation claim disagrees with the code

Owners: `/audit-facts` and `/audit-correctness`.

`/audit-correctness` runs the ownership ladder and routes rung 4 to `/audit-facts`, so under this
orchestrator the ladder has already run against wave 1's verdicts. Apply its result.

| Ladder result | Keep                 | Drop                                            |
|---------------|----------------------|-------------------------------------------------|
| Rungs 1 to 3  | `/audit-correctness` | The `/audit-facts` finding at the same location |
| Rung 4        | `/audit-facts`       | Nothing, correctness never reported it          |
| Rung 5        | Both, cross-linked   | Nothing, and the AMBIGUOUS marker stays         |

A rung 5 pair is the one case where both reports keep an entry. Cross-link them so a reader sees one
decision to make rather than two findings to triage.

### Collision 2: A construct has both a form and a runtime consequence

Owners: `/audit-style` and `/audit-performance`.

The canonical cases are a positional `dtype` argument, a bare array annotation, a missing
`cache=True`, a function-local import, and a missing `slots=True`.

| Situation                               | Keep                                                      |
|-----------------------------------------|-----------------------------------------------------------|
| Both audits reported the same construct | The performance finding, citing the style rule as context |
| Only `/audit-style` reported it         | The style finding, unchanged                              |
| Only `/audit-performance` reported it   | The performance finding, unchanged                        |

Collapsing to the performance finding is right because it carries strictly more information. It holds
the style rule AND the cited runtime consequence, so keeping both would report one construct twice.

### Collision 3: A timing property

Owners: `/audit-correctness` and `/audit-performance`.

`/audit-performance` owns cost and speed. `/audit-correctness` owns a timing property that forms part
of a CONTRACT, meaning a documented deadline, a watchdog interval, a real-time acquisition rate, or a
timeout the code itself computes.

Keep the correctness finding where a contract is cited with the line declaring it. Keep the
performance finding otherwise. Where both fired on one line and the contract citation is present, keep
correctness and carry the cost arithmetic as a tag.

### Collision 4: A parallelization race

Owners: `/audit-correctness` and `/audit-performance`.

`/audit-correctness` owns the race as CONCURRENCY_DEFECT. `/audit-performance` carries it only as a
tag on its NUMBA_CONFIGURATION_DEFECT finding, recording that the parallelization is invalid.

Keep both, because they say different things. The correctness finding says the result is wrong, and
the performance finding says the configuration enabling it is also misconfigured. Cross-link them.

---

## Deduplication

After the collisions are settled, deduplicate what remains.

**Fingerprint.** Normalize every finding to `<path>:<line>:<category>` and compare. An exact match
across two audits that no collision rule covered is a routing failure in one of them, so keep the
finding whose audit's Scope section claims the category and record the other as suppressed.

**One root cause, many sites.** Each audit already collapses repeats within its own report. Collapse
ACROSS audits only when the same root cause produced findings in two audits at the same sites, which
is rare and always a collision the rules above settle.

**Never merge across severity.** Two findings at one location with different categories and different
severities are two findings. Merging them would hide the more severe one behind the less severe one's
ordering.

---

## The merged triage header

Open the report with this, above the combined coverage ledger:

```text
Mode: <FULL | CHANGE, base <revision>>
Gate: <PASSED | BLOCKED | ADVISORY ONLY | CAPPED after 3 rounds>   (change mode only)
Round: <n> of 3                                                     (change mode only)

| Audit               | Ran | Blocking | Advisory | Total | Not run because           |
|---------------------|-----|----------|----------|-------|---------------------------|
| /audit-facts        | yes | <n>      | <n>      | <n>   |                           |
| /audit-style        | yes | <n>      | <n>      | <n>   |                           |
| /audit-correctness  | no  | 0        | 0        | 0     | DECLINED at Step 0        |
| /audit-performance  | no  | 0        | 0        | 0     | EMPTY bound file set      |

Adjudicated away by the collision rules: <n>
Suppressed as cross-audit duplicates: <n>
Discarded by the false-positive guards: <n> across all audits
Deleted by citation verification: <n> of <n> checked
Refuted by adversarial verification: <n> of <n> checked
```

The per-audit discard counts stay available inside each audit's own section. This header sums them, so
a reader sees the whole run's precision record in one place.

An audit that did not run carries exactly one of three reasons. DECLINED means the user answered the
Step 0 wave 2 election against it. EMPTY means its bound file set held nothing. A routing row means
change mode skipped it. Keep the three distinct, because a reader who cannot tell a user's choice from
an audit that had no work reads the same blank as a coverage gap.

A blocking count is meaningful in change mode alone. In full mode, report the column as the count of
findings that WOULD block, which tells a reader what a change-mode run over the same code would stop.

---

## The combined coverage ledger

Merge each audit's coverage ledger into one table, keeping the per-audit rows rather than summing
them, because the audits count different things.

```text
| Audit               | Files in scope | Files audited | Files skipped | Notes                             |
|---------------------|----------------|---------------|---------------|-----------------------------------|
| /audit-facts        | <n>            | <n>           | <n>           | metadata <n>, in-source <n>       |
| /audit-style        | <n>            | <n>           | <n>           | gates: <tools>, layout: <status>  |
| /audit-correctness  | <n>            | <n>           | <n>           | tiers swept T0-T3                 |
| /audit-performance  | <n>            | <n>           | <n>           | passes as its ledger reports them |
```

Each row is filled from the audit's own coverage ledger rather than recounted here. `/audit-style`
reports files SWEPT and files marked UNAUDITED, one row per style binding, so its row here sums those
rows and its two counts fill the audited and the skipped columns.

`/audit-performance` reports its pass range per language, because Pass 9 runs over C++ and C# alone,
so copy each language row rather than stating one range for the audit.

List every skipped file once, with the audits that skipped it and the reason. A file skipped by one
audit and covered by another is not a gap, and the ledger must show that rather than implying one. A
file `/audit-style` marked UNAUDITED is listed by path here alongside them.

Name every tool an audit ran in its Notes cell, and mark each tool that FAILED to run beside it. A
deterministic gate that produced no diagnostic because it crashed is a coverage gap rather than a
clean result, and a Notes cell that hides the failure reports the gap as coverage.

Carry the `/audit-style` project-scope layout pass status verbatim, as `run`,
`skipped-not-a-project-root`, or `skipped-no-created-or-deleted-files`. That pass sweeps the directory
tree once per run, so a skip is expected for a package or a single-file target and for a change set
that edits files in place. A skip over a whole repository in full mode is a coverage gap.

State the sub-agent or batch count each audit reported, and, for a run narrowed to a change set, the
base revision each audit resolved that narrowing against.

---

## Report ordering

Group by AUDIT, then preserve each audit's own output format unchanged inside its section, including
its internal ordering and its trailing `Appendix: LOW confidence` section.

Order the audit sections by the FIX order rather than the run order, because the report exists to be
acted on:

1. `/audit-facts`
2. `/audit-correctness`
3. `/audit-performance`
4. `/audit-style`

In change mode, place a `Blocking findings` section ahead of all four, listing every blocking finding
by file with a pointer to its full entry below. That section is what the fix pass works from, and
grouping it by file rather than by audit means each file is opened once.

Every finding that survived a collision rule carries one extra line:

```text
Adjudicated: <audit that yielded> yielded under <collision rule name>
```
