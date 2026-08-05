---
name: audit-facts
description: >-
  Performs a thorough fact-check audit of documentation against its authoritative source code, covering standalone
  documentation files and the docstrings and comments embedded in source code. Verifies every concrete claim and
  surfaces drift, contradictions, and substantive omissions with verbatim source citations. Use when auditing a README,
  CLAUDE.md, SKILL.md, Sphinx page, Python docstring, Doxygen block, or XML doc comment for factual accuracy, or when
  the user invokes /audit-facts.
user-invocable: true
---

# Documentation fact audit

Audits documentation against the authoritative source code, reporting only factual mismatches and substantive omissions
with verbatim source citations.

You MUST read this entire skill, and load each reference file at the step that names it, before acting on that step. The
verification checklist at the end is mandatory before submitting findings.

---

## Scope

**Covers:**
- Verifying claims in metadata documentation against source code (API names, signatures, file paths, behaviors,
  configuration values, workflow steps, cross-references, version pins)
- Verifying claims in in-source documentation against the implementation each block documents (parameters, return
  values, raised exceptions, attributes, units, ranges, invariants, defaults)
- Detecting drift between either documentation class and the current code state
- Identifying substantive omissions in documentation that partially covers a source area
- Verifying cross-file and cross-symbol references for accuracy
- Surfacing internal contradictions within a documentation file

**Does not cover:**
- Style, formatting, section ordering, or convention compliance (see `/audit-style`)
- Documentation quality such as density, length proportionality, typos, third-person imperative mood, separator
  punctuation, type-signature restating, and narrate-the-code comments (see `/audit-style`)
- A callable, class, module, or file that carries no documentation at all, which is a style finding (see `/audit-style`)
- A documentation and code mismatch where the documentation states the intended behavior and the code is the side to
  fix, together with every defect that involves no documentation at all (see `/audit-correctness`)
- Cost, speed, memory use, and dtype predictability (see `/audit-performance`)
- Code modifications or fact corrections (this skill produces findings only)
- Codebase exploration (see `/explore-codebase`)

---

## Documentation classes

Every audited artifact belongs to exactly one of two documentation classes. These are the canonical names used
throughout this skill.

**Metadata documentation** is a standalone file whose entire content is prose about the project. Its authoritative
source lives in other files.

**In-source documentation** is prose embedded inside a source file. It covers Python module, class, function, and
property docstrings and `#` comments, C++ Doxygen blocks (`@brief`, `@param`, `@returns`, `@note`, `@warning`,
`@tparam`) and `//` comments, and C# XML doc comments (`<summary>`, `<param>`, `<returns>`, `<remarks>`, `<exception>`)
and `//` comments. Its authoritative source is the implementation the block sits on, together with the symbols that
implementation calls.

Both classes drift, and both are in scope. A repository-wide audit that inspects only metadata documentation has covered
a fraction of the project's factual surface.

---

## Workflow

You MUST follow these steps when this skill is invoked.

Copy this progress checklist into your response and check off items as you complete them:

```text
Audit Progress:
- [ ] Step 0: Plan produced and confirmed
- [ ] Step 1: Scope resolved, documentation classes bound, tier selected
- [ ] Step 2: Claims extracted
- [ ] Step 3: Claims verified
- [ ] Step 4: Omission pass complete
- [ ] Step 5: Reference pass complete
- [ ] Step 6: Contradictions surfaced
- [ ] Step 7: False-positive guards applied
- [ ] Step 8: Findings verified (citation, refutation)
- [ ] Step 9: Coverage ledger assembled
- [ ] Step 10: Report produced
```

### Step 0: Produce audit plan and pause

Emit a plan before any verification work fires. The plan must list:

- Target file(s) in scope (resolved absolute paths), split by documentation class
- File counts per class, so the user sees the in-source volume before work starts
- Source code area authoritative for each metadata file
- Tier classification (small, medium, or large)
- Expected finding categories

Pause for user confirmation or a "proceed" signal. This catches misidentified targets before tokens burn on the wrong
scope. A user who wants a narrower run says so here, and the narrowing is recorded in the Step 9 coverage ledger.

### Step 1: Resolve target and bind documentation classes

Resolve the target into the set of files in scope. A target that names a single file binds that file to its own class
and stops there.

When the target is a directory or a repository root, **both documentation classes are in scope**. You MUST enumerate the
in-source documentation files and carry them through every subsequent step. Reducing a repository target to its metadata
documentation is a scope error, and it is the most frequent failure of this skill.

Enumerate with the project's version control index, which reports the files that actually ship:

```bash
# Metadata documentation in scope
git ls-files '*.md' '*.rst' '*.toml' '*.ini' '*.json' '*.yml' '*.yaml'

# In-source documentation in scope
git ls-files '*.py' '*.pyi' '*.h' '*.hpp' '*.cpp' '*.cs'
```

For a target outside version control, run the equivalent `find` over the same patterns. Add untracked files reported by
`git status --porcelain` when the audit covers work in progress.

Whole-repository coverage is the default and stays the default. Narrow to a change set ONLY when the user asks for that
in the invocation, resolving it with `git diff --name-only <base>...HEAD` for a branch, `git diff --name-only <commit>`
for one commit, or `git status --porcelain` for the working tree. A narrowed run still reads every surviving file in
full, because a claim is verified against a whole implementation rather than against a hunk. Record the narrowing and
the revision it resolved against in the Step 9 coverage ledger, so the report states what it did not cover.

Bind every enumerated file to its class and its authoritative source:

| File pattern                                                  | Class     | Authoritative source                                                     |
|---------------------------------------------------------------|-----------|--------------------------------------------------------------------------|
| `README.md`, `CLAUDE.md`, `AGENTS.md`                         | Metadata  | The packages, modules, CLI entry points, and registries the file names   |
| `SKILL.md`, skill `references/*.md`                           | Metadata  | The tools, workflows, and conventions the skill describes                |
| `docs/**/*.rst`, `docs/**/conf.py`                            | Metadata  | The documented public API and the project build configuration            |
| `pyproject.toml`, `tox.ini`, `platformio.ini`, `library.json` | Metadata  | The declared dependencies and environments, and the code that reads them |
| `.github/**/*.yml`, `.github/**/*.md`                         | Metadata  | The workflows, environments, and project layout the template references  |
| `tests/**/*.py`, `tests/**/*.cpp`, `tests/**/*.cs`            | In-source | The code under test, together with the test body itself                  |
| `*.py`, `*.pyi`                                               | In-source | The documented module, class, function, and the symbols they call        |
| `*.h`, `*.hpp`, `*.cpp`                                       | In-source | The documented file, class, method, and the symbols they call            |
| `*.cs`                                                        | In-source | The documented file, class, member, and the symbols they call            |

A file matching no row is out of scope. Record it in the coverage ledger as UNBOUND. State whether `tests/` is audited
or read as authority, bind every test path to the tests row rather than to its language row, and give it its own
coverage-ledger row.

For files that reference external libraries, include the installed package location in the authoritative source list for
that file. For ataraxis dependencies, invoke `/explore-dependencies` to obtain a current API snapshot before proceeding.

Classify the audit tier from the union of both classes:

| Tier   | Indicators                         | Execution                                               |
|--------|------------------------------------|---------------------------------------------------------|
| Small  | 1 file                             | Main agent, sequential                                  |
| Medium | 2 to 9 files                       | Main agent, file-by-file                                |
| Large  | 10 or more files or a project root | Parallel `general-purpose` sub-agents over file batches |

A repository-root target is always Large. Group Large-tier work under three rules, which exist so a sub-agent loads one
authority rather than the union of every authority in the repository.

1. **One metadata file, one sub-agent.** `README.md`, `CLAUDE.md`, `pyproject.toml`, `tox.ini`, and `platformio.ini`
   each get a dedicated sub-agent, because each resolves against a different authoritative source. A skill is one batch,
   its `SKILL.md` and `references/*.md` together, because the binding table gives them one source. The `docs/` package
   is one sub-agent covering every page.
2. **In-source work batches by package.** One sub-agent per package or per source directory, holding roughly eight
   files, and never mixing languages inside a batch.
3. **Forty sub-agents cap the run, and twelve run at once.** Every sub-agent re-receives the whole instruction payload,
   so the total bounds what the fan-out costs and the in-flight limit paces it. Batches beyond forty merge by shared
   authoritative source rather than dropping files.

Record the sub-agent count in the Step 9 coverage ledger.

Only the per-file claim verification fans out. Every other step runs on the main agent, because the omission, reference,
and contradiction passes each need the whole claim ledger in one view, and the guards, the verification, and the report
each sit on a trust boundary.

Do NOT use the `Explore` agent type for verification work. Explore returns summaries rather than verbatim citations and
breaks the "verbatim quote" discipline.

### Step 2: Extract verifiable claims

Run Pass 1 from [detection-passes.md](references/detection-passes.md), which builds the claim ledger every later step
consumes and tags each claim with the kind that decides which pass verifies it.

Walk each target file top to bottom and extract every verifiable claim.

In metadata documentation, a claim is any verifiable assertion. That covers API names and signatures, file paths,
directory layouts, canonical filenames, behaviors, configuration values, and defaults. It also covers workflow step
orderings, cross-references, version pins, external-library API references, numerical facts, existence assertions, and
date or version markers.

In in-source documentation, a claim is any assertion the implementation can confirm:

- A summary line stating what the callable, class, module, or file does
- A parameter description, including the parameter's name, its presence, and any default it states
- A return description, including the value's meaning, its units, and its range
- A documented raised exception, and the condition documented as triggering it
- A documented class or dataclass attribute, and the value it is documented to hold
- A stated unit, range, format, shape, dtype, invariant, ordering, or thread-safety guarantee
- An inline comment asserting a value, a magic constant's derivation, an algorithm, or a reason
- A reference from a docstring or comment to another symbol, module, file, issue, or version

Pass 1 states what is NOT a claim, and Guard 5 removes anything that reaches the ledger anyway.

### Step 3: Verify each claim

Run passes 2 through 6 from [detection-passes.md](references/detection-passes.md) in order, over ONE walk of the claim
ledger.

For each claim, locate the authoritative source and compare. You MUST open the source file and verify directly. Do NOT
verify from memory or training data.

For metadata claims, read the module, tool, or configuration file the claim names. For external-library claims, read the
installed library under `.venv`, conda env, or `site-packages`. For ataraxis dependencies, use the API snapshot from
`/explore-dependencies`.

For in-source claims, the implementation is the authority. Passes 3 through 6 define the mechanical comparisons, and
Guards 2 and 10 define what stops a candidate.

Run only the READ-ONLY form of any tool this audit consults, such as `ruff check --no-fix` and `mypy .`, because bare
`tox` and `tox -e lint` are FORBIDDEN here and reformat the code under audit.

For Large-tier audits, spawn the sub-agents the Step 1 grouping rules define. Each sub-agent receives its own files, the
documentation class, the authoritative source scope for those files alone, and the verdict categorization rules.
Sub-agents return findings in the output format defined below. The main agent synthesizes after all sub-agents complete.

Cite each source as `<path>:<line>` or `<path>:<line>-<line>`. For an in-source finding, the file location and the
source location commonly sit in the same file a few lines apart, and both are still required.

Categorize every claim with one of these seven verdicts. This step assigns the first five, and the omission and
contradiction passes in Steps 4 and 6 assign the last two:

| Verdict       | Meaning                                                                    |
|---------------|----------------------------------------------------------------------------|
| EXACT         | Claim matches source verbatim                                              |
| SEMANTIC      | Claim is correct in meaning but uses different wording                     |
| DRIFT         | Claim was true at one point but has changed in source                      |
| WRONG         | Claim is factually incorrect                                               |
| CONTRADICTION | Claim and another claim about the same thing cannot both be true           |
| OMISSION      | Partially populated section misses members of the surface it covers        |
| UNVERIFIABLE  | Source is missing, or claim is too vague to verify after reasonable search |

Also assign a confidence tier to every finding:

| Confidence | Meaning                                                            |
|------------|--------------------------------------------------------------------|
| HIGH       | Verbatim source quote present, mismatch is mechanical              |
| MEDIUM     | Source quote present, mismatch requires interpretation             |
| LOW        | Pattern detected but source/claim mapping is inferred, not literal |

For UNVERIFIABLE findings, state what you searched for and where you looked.

List ALL non-matching claims in each pass, walking a block that yields one WRONG claim to the end of its ledger rows
rather than stopping at the first.

### Step 4: Omission pass

Run Pass 7 from [detection-passes.md](references/detection-passes.md).

Within the per-file source scope from Step 1 only, walk the source code and check whether the documentation mentions
every relevant public API, behavior, and constraint.

An omission is substantive when the documentation already partially covers a surface but misses pieces of it. In
metadata documentation, examples include documenting 3 of 5 public functions in a module the file walks through,
describing a workflow but skipping a required step, or listing registries but missing one. In in-source documentation,
examples include an `Args:` section that documents 3 of 5 parameters and a `Raises:` section that omits an exception the
body raises. They also include an `Attributes:` section missing an attribute the class assigns, and a Doxygen block
carrying `@param` tags for some parameters of a method.

Do NOT flag documentation for lacking an entire section, format, or convention, and do NOT flag a callable, class,
module, or file that carries no documentation at all. Both are style concerns and belong to `/audit-style`.

### Step 5: Reference pass

Run Pass 8 from [detection-passes.md](references/detection-passes.md).

For every "see X" or "documented in Y" reference in metadata documentation, verify that X exists and contains what the
file claims it contains.

For in-source documentation, apply the same check to symbol references: Sphinx cross-reference specifiers in Python
docstrings, `@see` and `@ref` tags in Doxygen blocks, `<see cref="...">` elements in XML doc comments, and any module,
file, or symbol a comment names.

Flag only broken or wrong references.

### Step 6: Internal contradiction pass

Run Pass 9 from [detection-passes.md](references/detection-passes.md).

Identify cases where the documentation makes incompatible claims about the same thing. This includes a metadata file
contradicting itself, and a docstring contradicting a comment or another docstring inside the same module about the same
symbol.

### Step 7: Apply the false-positive guards

Walk every candidate through every guard in [false-positive-guards.md](references/false-positive-guards.md), in order.
The authoritative-source guard runs first and removes the most candidates. Discard everything a guard rejects, and
record the count of discarded candidates for the report's triage header.

### Step 8: Verify the surviving findings

Run the two checks in [verification-protocol.md](references/verification-protocol.md), in order:

1. **Citation verification**, against every surviving finding with no sampling. Confirms the Claim quote appears at the
   cited documentation line and the quoted source reality appears at the cited source line.
2. **Adversarial refutation**, against every WRONG and CONTRADICTION finding. A fresh `general-purpose` sub-agent per
   finding, instructed to refute it and to answer REFUTED under uncertainty.

Both checks are external, testing the finding against the files and against a reader who never saw the verification
pass. They catch the failure mode this audit produces most often, which is confidently misquoting source. Record every
count the protocol names, because the Step 9 ledger and the report's triage header carry them.

### Step 9: Assemble the coverage ledger

Build the ledger that opens the report. It records what was audited, so a thin in-source pass is visible rather than
silent:

```text
| Documentation class | Files in scope | Files audited | Files skipped |
|---------------------|----------------|---------------|---------------|
| Metadata            | 12             | 12            | 0             |
| In-source           | 47             | 47            | 0             |
| Tests               | 9              | 9             | 0             |
```

List every skipped and UNBOUND file by path with its reason, and state the sub-agent count. Skipping is allowed only
when the user narrowed the scope in Step 0 or Step 1, when a file is generated, or when a file is unreadable. A tests
row bound as authority rather than audited is recorded in the skipped list with the reason `authority, not audited`. A
run narrowed to a change set names the revision it resolved against here. A Large-tier audit that produced no in-source
findings still reports a non-zero audited count in the in-source row, which distinguishes clean documentation from an
unrun pass.

### Step 10: Produce the findings report

Use the output format below.

Skip EXACT and SEMANTIC findings entirely. Report every surviving finding at every confidence tier by default, which
covers LOW alongside HIGH and MEDIUM. Narrow the report to HIGH and MEDIUM only when the user explicitly asks for it via
`--min-confidence medium` or equivalent invocation.

The confidence tier stays on every finding, so a reader triages by tier rather than by trusting that the report was
filtered. LOW means the source and claim mapping is inferred rather than literal, and it never excuses a finding from
the citation rules in the Discipline section. LOW findings sit in the trailing `Appendix: LOW confidence` section the
protocol defines rather than interleaved into the file groups, so the body of the report reads at one confidence level.

---

## Output format

Open the report with the triage header from [verification-protocol.md](references/verification-protocol.md), then the
Step 9 coverage ledger. The header carries the finding counts by verdict and confidence together with every discard
count the guards and the Step 8 checks produced. Then report only WRONG, DRIFT, CONTRADICTION, OMISSION, and
UNVERIFIABLE findings, in that order.

When the audit spans multiple files, group the HIGH and MEDIUM confidence findings hierarchically: documentation class
-> file -> finding verdict -> findings. Collect LOW confidence findings into the trailing `Appendix: LOW confidence`
section, ordered by the same verdict sequence.

Every finding uses the shape below, shared by all four audits in this family so one reading habit serves them all.

```text
### <ID> · <WRONG | DRIFT | CONTRADICTION | OMISSION | UNVERIFIABLE> · <one-line statement of the defect>

`<path>:<line>` · <METADATA | IN-SOURCE> · <HIGH | MEDIUM | LOW> confidence · <source `<path>:<line>`, or N/A>

- **Wrong:** <the defect, carrying every quote and citation the evidence floor requires>
- **Fix:** <the concrete change, described rather than applied>
- **Impact:** <what the change alters for callers and downstream, or "None" when nothing observable changes>
- **Choice:** <the options, one clause each, closing with a recommendation>
```

**ID** is a short stable handle, `F1`, `F2`, and so on, numbered in report order, so a reader answers with the
identifier rather than by restating the finding.

**Wrong** carries the whole evidence load as prose rather than as labelled fields, stating the claim quoted verbatim
from the documentation with its own `<path>:<line>`, and the source reality quoted verbatim or summarized with the
citation that establishes it. A table, a ledger, or an interleaving sits directly beneath the bullet.

**Impact** states what the fix alters for a caller or a downstream project, and states "None" when the change is
behavior-preserving. Naming a break here IS the signal that the fix needs the owner's decision.

**Choice** appears only where the audit cannot settle the question, covering a claim the source neither confirms nor
denies, where the owner decides between correcting the prose and removing it. Each option gets one clause, and the
bullet closes with a recommendation.

An UNVERIFIABLE finding replaces the source reality with a description of what was searched and where, so a reader can
extend the search rather than repeat it.

---

## Discipline

You MUST adhere to the following discipline during every audit, and you MUST apply every guard in
[false-positive-guards.md](references/false-positive-guards.md) before reporting.

- Never paraphrase source and present it as a verbatim quote. Use the Read tool and copy.
- Never expand scope to restructure, restyle, or refactor. This skill produces findings only.
- Do not flag subjective preferences (tone, ordering, terminology).
- Never invent an exemption. An exemption exists only where a loaded skill writes it down, and you MUST quote that
  clause before applying it. Shared corpus, house convention, text byte-identical in a sibling repository, long-standing
  code, and "it reads fine" are none of them, so a real finding survives wherever else the same text appears.
- Hold the report's own prose to the rules this family enforces, keeping every authored sentence under 40 words and
  separating clauses with full stops and commas rather than semicolons or em-dashes.
- Fill each authored line to 120 characters before breaking it, under the wrap width rule `/python-style` defines, so a
  line ending before column 100 while its next word would still fit is re-flowed.

---

## Related skills

| Skill                   | Relationship                                                                            |
|-------------------------|-----------------------------------------------------------------------------------------|
| `/audit-project`        | Orchestrator that runs this audit in wave 1 and merges its findings with the siblings   |
| `/audit-style`          | Sibling audit for style, formatting, documentation quality, and convention compliance   |
| `/audit-correctness`    | Sibling audit owning the same mismatch when the code is the side to fix                 |
| `/audit-performance`    | Sibling audit for cost, speed, memory use, and dtype predictability                     |
| `/explore-codebase`     | Provides project structure context. Invoke first when auditing an unfamiliar codebase   |
| `/explore-dependencies` | Provides ataraxis API snapshots. Invoke before verifying external API claims            |
| `/python-style`         | Defines the docstring sections whose contents this skill verifies against Python code   |
| `/cpp-style`            | Defines the Doxygen tags whose contents this skill verifies against C++ code            |
| `/csharp-style`         | Defines the XML doc tags whose contents this skill verifies against C# code             |
| `/readme-style`         | Provides README conventions for context (compliance is handled by `/audit-style`)       |
| `/api-docs`             | Provides Sphinx and Doxygen build conventions for context when auditing `docs/` pages   |
| `/skill-design`         | Provides SKILL.md, CLAUDE.md, and AGENTS.md conventions (compliance via `/audit-style`) |

---

## Proactive behavior

Invoke this skill when the user asks to fact-check, verify, or audit documentation against the source code. A request
that names a directory or a repository covers both documentation classes by default, and the Step 0 plan is where the
user narrows that scope.

Fix this audit's findings FIRST, before those of `/audit-correctness`, `/audit-performance`, and `/audit-style`, when
auditing the same file end to end. Factual corrections may rewrite prose that style would otherwise restyle redundantly,
and settling which side of a documentation mismatch is authoritative comes before auditing the code. That is a FIX
order, and `/audit-project` decides the RUN order.

Do NOT make code or documentation changes during the audit. Present findings and wait for user direction.

---

## Verification checklist

You MUST verify the audit output against this checklist before presenting it to the user.

```text
Documentation Fact Audit Compliance:
- [ ] Step 0 plan produced and confirmed by user before verification began
- [ ] Plan listed per-class file counts, including the in-source count
- [ ] Both documentation classes enumerated for every directory or repository target
- [ ] Every file in scope bound to a class and an authoritative source per the binding table
- [ ] Test files bound explicitly, either audited as their own in-source ledger row or recorded as authority, never
      left to the enumeration to decide
- [ ] Tier classified (small/medium/large) and agent allocation matched the table
- [ ] For Large tier, each metadata file, each skill batch, and the docs package given its own sub-agent
- [ ] For Large tier, in-source batched by package with no language mixing, within 40 and 12 in flight, merging to fit
- [ ] Scope narrowed to a change set only on explicit request, with the revision recorded in the ledger
- [ ] Every claim carries a verdict (EXACT, SEMANTIC, DRIFT, WRONG, CONTRADICTION, OMISSION, UNVERIFIABLE)
- [ ] Every harvested claim carries a verdict, with all non-matching claims in a block reported rather than the
      first one only
- [ ] Every claim assigned a confidence tier (HIGH, MEDIUM, LOW)
- [ ] Every non-EXACT and non-SEMANTIC finding cites a source location <path>:<line>
- [ ] Claim ledger built in Pass 1, with every claim tagged by kind
- [ ] Detection passes 2 through 9 run in order, each against the claims its kind assigns it
- [ ] Every claim verified by reading the source file (no memory-based verification)
- [ ] Full body of every documented callable read before its documentation was judged
- [ ] External library claims verified against installed library, not training data
- [ ] Suppression claims settled only through a read-only tool invocation, with bare `tox` and `tox -e lint` never
      run and any unavailable tool recorded in the ledger
- [ ] Omission pass executed and bounded by Step 1 scope (no out-of-scope omission claims)
- [ ] Cross-file and cross-symbol references verified for accuracy
- [ ] Internal contradictions surfaced
- [ ] Every false-positive guard applied in order, with the discarded-candidate count recorded
- [ ] Citation verification run against every finding, with the claim quote and the source quote both confirmed
- [ ] Every finding whose quote or line failed citation verification deleted rather than repaired
- [ ] Adversarial refutation run against every WRONG and CONTRADICTION finding, in fresh sub-agents
- [ ] Every refuted finding discarded, and the confirmed and refuted counts recorded
- [ ] Triage header present, carrying the verdict by confidence counts and every discard count
- [ ] Coverage ledger present, with every skipped and UNBOUND file listed by path and reason
- [ ] In-source row of the ledger shows a non-zero audited count for repository targets
- [ ] Every confidence tier reported, with LOW included unless the user narrowed the report
- [ ] LOW confidence findings placed in the trailing appendix rather than interleaved
- [ ] No EXACT or SEMANTIC findings appear in the report
- [ ] No style, formatting, convention, or documentation-quality findings appear in the report
- [ ] No wholly undocumented callable, class, module, or file reported as an omission
- [ ] No file modifications made during the audit
- [ ] Findings ordered: WRONG -> DRIFT -> CONTRADICTION -> OMISSION -> UNVERIFIABLE
- [ ] Fix bullets are concrete textual edits, described rather than applied
- [ ] Every finding uses the shared shape, carrying a stable ID, a rank, a location line, and the Wrong, Fix, and
      Impact bullets
- [ ] Every Impact bullet names what the fix alters for callers and downstream, or states None
- [ ] A Choice bullet appears only where the audit cannot settle the question, and it closes with a recommendation
- [ ] Every sentence the report itself writes, outside a verbatim quote, is under 40 words and uses only full stops
      and commas as clause separators
- [ ] Report prose fills each line to 120 characters, with no line ending before column 100 while its next word would
      still fit
```
