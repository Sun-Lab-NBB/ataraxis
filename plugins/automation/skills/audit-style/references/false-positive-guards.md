# False-positive guards

Every candidate finding of `/audit-style` passes through these guards in order before it enters the
report. The rule-ledger guard runs first and removes the most candidates. Record the count of
discarded candidates for the report's triage header.

This audit's characteristic false positive is an INVENTED CONVENTION, meaning a rule that reads like
good practice, that no loaded checklist states, and that the audit therefore has no authority to
enforce. Several guards below exist for that one failure alone.

## Contents

- Guard 1: No ledger row, no finding
- Guard 2: The tools own what the tools decide
- Guard 3: Resolve the construct's kind before judging it
- Guard 4: Documented exceptions and exemptions are checked, never assumed away
- Guard 5: Generated, vendored, and auto-generated blocks are out of scope
- Guard 6: Tests carry their own rules
- Guard 7: Console enable and disable calls are correct
- Guard 8: An unbound file produces nothing
- Guard 9: Factual findings belong to /audit-facts
- Guard 10: Defects and costs belong to the sibling code audits
- Guard 11: An inconsistency needs the checklist to permit both forms
- Guard 12: A conflict is surfaced rather than resolved
- Guard 13: One rule, one finding
- Guard 14: Layout paths carry their own exemptions

---

## Guard 1: No ledger row, no finding

Every finding cites a rule identifier from the Pass 1 rule ledger, together with that rule's verbatim
text. A candidate matching no ledger row is DELETED rather than reported at low confidence.

"This would read better", "the convention elsewhere is", and "most projects do" are all unreportable.
A convention absent from every loaded checklist is not a violation, however sound it is. Where the
convention genuinely should exist, say so to the user outside the report and leave the checklist to
its owning style skill.

This guard removes more candidates than the rest combined, so apply it first and without mercy.

---

## Guard 2: The tools own what the tools decide

A rule the Step 3 gates decide is reported from the TOOL'S output or not at all. Line length,
indentation, blank-line counts, quote and string form, trailing commas, import sorting and grouping,
and every rule carried by a ruff code the project enables all belong to the tool.

Re-deriving one of those by reading is forbidden, because the tool is exact and a reading is not. A
tool that ran and stayed silent on a line has cleared that line, so a hand-derived finding contradicting
a tool that ran is deleted rather than reported.

Where a configured tool failed to run, its rules stay unchecked and the report says so. It does not
hand them back to the sweep.

---

## Guard 3: Resolve the construct's kind before judging it

Casing, visibility, and annotation rules differ per kind in every one of these languages, so an
identifier judged as the wrong kind produces a finding that does not exist.

Resolve the kind at the declaration before reporting. A constant differs from a variable, a
module-level function differs from a method, an enum member differs from a class attribute, a property
differs from a field, and a type alias differs from a class. Apply the same resolution to a receiver
before judging a call, because a project type carrying a familiar method name is a different call
entirely.

---

## Guard 4: Documented exceptions and exemptions are checked, never assumed away

A checklist that exempts a form exempts it wherever the form appears. Before reporting, find the
exemption clause and confirm the construct sits outside it, quoting what you checked.

The recurring exemptions are the sanctioned error-reporting patterns and their trailing returns, the
keyword-argument rules and their documented exceptions, the dataclass and enum declaration forms, and
the per-archetype C++ rules where the embedded prohibitions bind embedded firmware alone. Determine
the archetype from the build files before applying any of the last group.

---

## Guard 5: Generated, vendored, and auto-generated blocks are out of scope

Stub files, the typing marker, generated API pages, and vendored third-party trees are produced by a
tool rather than hand-authored, so a style divergence in them is regenerated rather than fixed.

Audit nothing inside a virtual environment, site-packages, a tox working directory, or a build
directory. Where a hand-authored file carries an auto-generated block or a documented exception, note
the exception and skip its enclosing range rather than reporting inside it.

---

## Guard 6: Tests carry their own rules

The test suite is the one place the framework permits private access and direct submodule imports, and
the lint configuration ignores the corresponding codes there.

Confirm which rules the project's own configuration relaxes under `tests/` before reporting anything
in it, and report only the rules that still apply.

---

## Guard 7: Console enable and disable calls are correct

`console.enable()` and `console.disable()` calls are correct at every library tier, including component
and dependency libraries. Mention such a call to the user for awareness at most, and never report it as
a violation or propose removing it.

---

## Guard 8: An unbound file produces nothing

A file matching no row of the Step 1 binding table has no applicable checklist, so it is marked
UNAUDITED in the plan and the report and produces zero findings.

Judging it against a checklist bound to a different file type is an invented convention wearing a
citation.

---

## Guard 9: Factual findings belong to /audit-facts

A stale reference to a renamed symbol, a removed feature, or a closed issue, a docstring claim that
disagrees with the signature or the observable behavior, and a suppression comment whose diagnostic no
longer fires are all FACTUAL, because settling them requires reading the implementation or running a
tool.

This audit judges the FORM of the prose. Where a block breaks both a form rule and a fact, report the
form rule here and leave the fact to its owner.

---

## Guard 10: Defects and costs belong to the sibling code audits

A defect, an edge case, a race, a leak, and a runtime cost are never findings here.
`/audit-correctness` and `/audit-performance` own them.

The boundary that decides most disputes is that this audit owns the FORM of a construct and
`/audit-performance` owns its numeric or temporal consequence. A positional `dtype` argument, a bare
array annotation, a missing `cache=True`, and a function-local import are style findings here, and
each enters that report only with a cited runtime consequence attached.

---

## Guard 11: An inconsistency needs the checklist to permit both forms

INCONSISTENCY is reportable only where the loaded checklist permits both forms, separately, and the
file set mixes them. Quote the clause that permits each.

Where the checklist prescribes one form outright, the deviating file is an ordinary finding under the
pass owning that rule, so report it there instead. Where the checklist states nothing, Guard 1 deletes
the candidate.

---

## Guard 12: A conflict is surfaced rather than resolved

Where two loaded checklists state the same point incompatibly, the finding is the CONFLICT itself.
Quote both rules with their skill and reference file, and leave the decision to the user.

Silently picking one skill's rule and reporting the other's violation converts a documentation defect
into a false finding against the code.

---

## Guard 13: One rule, one finding

Repeated violations of one checklist point within a file collapse into ONE finding carrying the site
list and a count.

Equally, one construct that several passes flag is reported once, under the pass owning the rule that
describes it most precisely, with the others listed as tags. BLOCKING outranks STANDARD wherever both
apply to the same construct.

---

## Guard 14: Layout paths carry their own exemptions

Every Pass 10 candidate clears this guard before it enters the report, because a tree diff flags paths
the repository is right to lack and paths it is right to hold.

Four classes are discarded outright:

1. **Gitignored build output.** `dist/`, `build/`, `reports/`, the coverage exports, and the generated
   stubs and markers the archetype tree marks as release-phase are absent or present by the phase the
   repository sits in, so neither direction is a finding.
2. **Paths the archetype tree marks optional.** A tree entry annotated `(optional)` is absent by
   permission, so its absence supports no finding.
3. **Generated and vendored directories.** A directory a tool produces or a third party ships is
   regenerated rather than restructured, so neither its presence nor its contents are reportable.
4. **Absent paths under a low-confidence archetype.** Where the key indicators matched partially or
   contradicted one another, the required-path set is unsettled, so report the ambiguous archetype to
   the user and leave the absent paths unjudged.

A stray path still reports under a low-confidence archetype when every candidate archetype forbids it,
and waits on the archetype question otherwise.
