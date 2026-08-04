# False-positive guards

Every candidate finding of `/audit-facts` passes through these guards in order before it enters the report. The
authoritative-source guard runs first and removes the most candidates. Record the count of discarded candidates for the
report's triage header.

This audit's characteristic false positive is a mismatch that is not a mismatch. It is produced by reading the signature
instead of the body, by resolving a name against its definition instead of its re-export, or by judging a paraphrase
against the wording it paraphrases.

---

## Contents

- Guard 1: The claim is authoritative and the repository is not
- Guard 2: Read the whole body, and follow the delegation
- Guard 3: A re-export is the module the documentation names
- Guard 4: SEMANTIC is a match
- Guard 5: Aspirational, subjective, and pedagogical prose is not a claim
- Guard 6: Style, formatting, and documentation quality belong to /audit-style
- Guard 7: Absent documentation is a style finding
- Guard 8: Defects and costs belong to the sibling code audits
- Guard 9: The code is the wrong side
- Guard 10: A suppression comment needs the linter to be judged
- Guard 11: Generated and vendored documentation is out of scope
- Guard 12: An omission needs a partially covered surface
- Guard 13: UNVERIFIABLE needs a stated search
- Guard 14: One claim, one finding

---

## Guard 1: The claim is authoritative and the repository is not

Facts the audited repository cannot derive from its own source are AUTHORITATIVE by default. This covers external
toolchain version floors, installer requirements, cross-repository version pins, environment prerequisites, hardware
specifications, and vendor documentation the file cites.

Report such a claim only when the file contradicts itself about it, and set the Suggested fix to "leave as-is,
authoritative external requirement" rather than a removal or a change. `/readme-style` is the source of the canonical
install-section requirements, so a claim matching those is settled there rather than here.

The failure this prevents is an audit that deletes a true requirement because the repository holds no copy of it.

---

## Guard 2: Read the whole body, and follow the delegation

Before reporting any BEHAVIOR claim, state that you read the full body of the documented callable, class, or module, and
that you followed every call it delegates to.

A summary that looks wrong at the signature is frequently satisfied deeper in the body, and behavior a docstring
describes is frequently performed by a helper. Delegated behavior still belongs to the documented callable, so a claim
is a finding only after the delegation chain is walked and no statement anywhere in it satisfies it.

---

## Guard 3: A re-export is the module the documentation names

When the documentation says X lives in module Y, and Y re-exports X from elsewhere, the claim is EXACT.

Check the package `__init__` before reporting any missing symbol. The framework requires a symbol consumed outside its
defining package to be exported from that package's `__init__` and imported through the package namespace, so re-export
is the normal case rather than the exception.

Tests are the exception the framework states. A docstring or comment under `tests/` naming a private member, or
importing directly from a submodule, is correct rather than a broken reference, because the framework permits both
there.

---

## Guard 4: SEMANTIC is a match

A claim that states the same fact in different words is SEMANTIC, which is a match and stays out of the report entirely.

Reserve DRIFT and WRONG for a claim no statement satisfies. Rewording a correct sentence is a style concern, and this
audit makes no wording recommendations.

---

## Guard 5: Aspirational, subjective, and pedagogical prose is not a claim

Subjective quality language, future-tense and aspirational statements, motivational prose, and design rationale carry
nothing the source can confirm or deny, so they never entered the claim ledger and never enter the report.

Pedagogical "why" prose is a claim only where it contains a verifiable factual statement, and then the statement is the
claim rather than the passage holding it.

---

## Guard 6: Style, formatting, and documentation quality belong to /audit-style

Section ordering, mood, sentence length, length proportionality, redundancy with the type signature, narrate-the-code
comments, separator punctuation, and every other property of the FORM of the prose belong to `/audit-style`.

A docstring or a comment enters this report only when a fact inside it disagrees with the code.

---

## Guard 7: Absent documentation is a style finding

A callable, class, module, or file carrying no documentation at all, and a documentation section missing entirely, are
both style concerns.

This audit reports what documentation SAYS. It never reports the absence of documentation.

---

## Guard 8: Defects and costs belong to the sibling code audits

A defect, an edge case, a race, a leak, and a runtime cost are never findings here, whatever the documentation says
about them. `/audit-correctness` and `/audit-performance` own them.

Where verification of a claim uncovers a defect, note the claim's verdict here and leave the defect to its owner rather
than reporting both.

---

## Guard 9: The code is the wrong side

The implementation is authoritative over the documentation, so the Suggested fix edits the DOCUMENTATION by default.

Before reporting a mismatch where the code looks like the side to fix, run the ownership ladder in `/audit-correctness`.
A caller, a test, or an external schema that is correct only under the documented reading, an independently enforced
guard, and an observable failure for a documented-supported input each move the finding to that skill. Report it here
only when the implementation is self-consistent and every caller already matches it, and say so in the Suggested fix
when the code still looks wrong.

---

## Guard 10: A suppression comment needs the linter to be judged

A `# noqa` or `# type: ignore` code is a claim about a diagnostic, and only the project's linter or type checker can
settle whether the line still produces it.

Report the suppression as DRIFT only when that tool is available and confirms the diagnostic is gone. Without the tool,
leave the suppression unreported rather than guessing.

Run that tool in its READ-ONLY form only. `ruff check --no-fix` and `mypy .` settle the claim. Bare `tox` and `tox -e
lint` are FORBIDDEN during an audit, because the `lint` environment reformats the source, auto-fixes it, and purges its
stubs, which mutates the very code the claims are verified against. Where no read-only invocation is available, leave
the suppression unreported and record the tool as unavailable in the Step 9 coverage ledger.

---

## Guard 11: Generated and vendored documentation is out of scope

Stub files, generated API pages, and vendored third-party documentation are regenerated rather than edited, so a
divergence in them is no finding.

Audit nothing inside a virtual environment, site-packages, a tox working directory, or a build directory. Read them as
authority for an external API's contract, and report findings only against this repository's own files.

---

## Guard 12: An omission needs a partially covered surface

An omission is reportable only where the documentation already covers PART of a surface and misses pieces of it, such as
an `Args:` section documenting three of five parameters.

Enumerate the full surface from the source and name the specific members the section misses. A surface the documentation
never undertook to cover produces nothing, and neither does a surface outside the per-file source scope Step 1 fixed.

---

## Guard 13: UNVERIFIABLE needs a stated search

UNVERIFIABLE records that the audit looked and failed, so it carries what was searched for and where it was searched. A
verdict without both is an unexamined claim rather than an unverifiable one.

Search the package `__init__`, the installed library under the environment, and the version control index before
assigning it. A claim too vague to state as a testable proposition is dropped from the ledger rather than reported as
UNVERIFIABLE.

---

## Guard 14: One claim, one finding

A single stale fact repeated across several files is ONE finding carrying the site list and a count, in the same style
the sibling audits use for repeated violations.

Equally, one claim that several passes flag is reported once, under the verdict that describes it most precisely, with
the others listed as tags. WRONG outranks DRIFT, and both outrank OMISSION.
