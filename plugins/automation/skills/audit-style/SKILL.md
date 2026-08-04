---
name: audit-style
description: >-
  Performs a thorough style-compliance audit of source code, configuration, or documentation files
  against the applicable ataraxis framework style skill checklists. Walks every line in scope and
  reports only non-compliant findings with verbatim checklist citations. Use when auditing a Python
  package, config file, README, or any project file for style, formatting, naming, or convention
  compliance, or when the user invokes /audit-style.
user-invocable: true
---

# Style compliance audit

Audits files against the authoritative ataraxis framework style skill checklists, reporting only
non-compliant findings with verbatim checklist citations.

You MUST read this entire skill, and load each reference file and each style skill checklist at the
step that names it, before acting on that step. The verification checklist at the end is mandatory
before submitting findings.

---

## Scope

**Covers:**
- Auditing single files, directories, or full project trees against applicable style skills
- Running the project's own linters, formatters, and type checkers in their read-only form, and
  folding their diagnostics into the report as findings
- Structural style: element ordering, imports, formatting, naming, type annotations,
  error-handling patterns, file-section ordering
- Symbol visibility and usage: each symbol's declared tier against the widest boundary its consumers
  actually cross, the package export lists against the set of packages importing from them, and assets
  that no consumer references
- Project-scope layout: the repository directory tree against the archetype tree its indicators
  resolve to, for a project-root target
- Comment and docstring quality: typos, sentence length, length proportionality, redundancy
  with the type signature, narrate-the-code comments, separator punctuation, and behavioral
  scope
- Cross-file consistency: naming, ordering, and idiom drift across the file set
- Cross-skill conflicts where two loaded style skills disagree on the same point

**Does not cover:**
- Factual accuracy of documentation against source code, including stale references and
  docstring claims that disagree with the implementation (see `/audit-facts`)
- Active and latent bugs, and behavior that breaks the contract the code states
  (see `/audit-correctness`)
- Cost, speed, memory use, and dtype predictability, which covers the runtime consequence of a
  construct whose form this skill judges (see `/audit-performance`)
- Code modifications or style fixes (this skill produces findings only), which includes every
  auto-fixing and reformatting tool invocation
- Re-deriving by reading any rule the project's own tools already decide
- Inventing new conventions not present in any loaded style skill checklist
- Codebase exploration (see `/explore-codebase`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

Copy this progress checklist into your response and check off items as you complete them:

```text
Audit Progress:
- [ ] Step 0: Plan produced and confirmed
- [ ] Step 1: Target resolved, style skills bound, tier selected
- [ ] Step 2: Style skill checklists loaded and rule ledger built
- [ ] Step 3: Deterministic gates run and their findings collected
- [ ] Step 4: Project-scope layout sweep run, or skipped and recorded
- [ ] Step 5: Line-by-line sweep complete (all four dimensions)
- [ ] Step 6: Findings categorized
- [ ] Step 7: False-positive guards applied
- [ ] Step 8: Findings verified (citation, refutation)
- [ ] Step 9: Coverage ledger assembled
- [ ] Step 10: Report produced
```

### Step 0: Produce audit plan and pause

Emit a plan before any sweep work fires. The plan must list:

- Files in scope (resolved absolute paths)
- Style skills bound to those files (per the binding table in Step 1)
- Tier classification (small, medium, or large)
- Whether the Step 4 project-scope layout sweep runs, and the reason when it does not
- Whether the Pass 11 symbol usage sweep runs in full, runs partially, or is skipped, with the reason
- Expected finding categories

Pause for user confirmation or a "proceed" signal. This catches misidentified targets before
tokens burn on the wrong scope.

### Step 1: Resolve target and bind style skills

Resolve the target (a single file, a package directory, or a project root) into the set of
files in scope. For directory or project-root targets, every file under the target is in scope.
There is no "covered area" reduction.

Whole-repository coverage is the default and stays the default. Narrow to a change set ONLY when the
user asks for that in the invocation, resolving it with `git diff --name-only <base>...HEAD` for a
branch, `git diff --name-only <commit>` for one commit, or `git status --porcelain` for the working
tree. A narrowed run still reads every surviving file in full, because ordering, visibility grouping,
and length proportionality are properties of a whole file. It also still builds the Step 5 reference
table across the WHOLE repository, because a symbol's tier is decided by consumers a change set does
not contain, and Guard 16 skips the usage pass outright rather than deciding an absence from a partial
table. Record the narrowing and the revision it resolved against in the report, so it states what it
did not cover.

For each file in scope, identify the applicable style skill using the binding table:

| File pattern                                                              | Style skill          |
|---------------------------------------------------------------------------|----------------------|
| `*.py`                                                                    | `/python-style`      |
| `*.cs`, `.editorconfig`, `.csharpierrc.yaml`, `.csharpierignore`          | `/csharp-style`      |
| `*.h`, `*.hpp`, `*.cpp`, `.clang-format`, `.clang-tidy`, `CMakeLists.txt` | `/cpp-style`         |
| `README.md`                                                               | `/readme-style`      |
| `pyproject.toml`                                                          | `/pyproject-style`   |
| `tox.ini`                                                                 | `/tox-config`        |
| `platformio.ini`, `library.json`                                          | `/platformio-config` |
| `docs/**/*.rst`, `docs/**/conf.py`, `Makefile`, `make.bat`, `Doxyfile`    | `/api-docs`          |
| `SKILL.md`, skill `references/*.md`, `CLAUDE.md`, `AGENTS.md`             | `/skill-design`      |
| `.github/ISSUE_TEMPLATE/*.yml`                                            | `/project-layout`    |
| `envs/*`                                                                  | `/project-layout`    |
| Project directory tree                                                    | `/project-layout`    |

Where a file matches more than one row, the MOST SPECIFIC pattern wins, which resolves
`docs/source/conf.py` to `/api-docs` rather than to any broader row it also matches. The
`Project directory tree` row binds no file at all and is executed by the Step 4 layout sweep rather
than by the per-file passes.

If a file in scope matches no binding row, mark it UNAUDITED in the plan and report (no
applicable style skill) and flag no findings against it.

Classify the audit tier, and note that the three rows partition the file count with no overlap and no
gap:

| Tier   | Indicators                         | Execution                                              |
|--------|------------------------------------|--------------------------------------------------------|
| Small  | 1 file                             | Main agent, sequential                                 |
| Medium | 2 to 9 files                       | Main agent, file-by-file                               |
| Large  | 10 or more files or a project root | Parallel `general-purpose` sub-agents over file batches |

### Batching isolates the style guides

A Large-tier audit BATCHES BY BINDING, so no sub-agent carries more than one or two checklists. This
is the rule that decides what this audit costs. Every sub-agent re-receives the checklists its files
bind to, and the loaded checklists of a mixed project root run to tens of thousands of tokens, so a
sub-agent holding every binding pays for guides it never applies, once per sub-agent.

Build the batches under three rules:

1. **One binding per batch, two at the absolute most.** Group files by the style skill the table above
   assigned. A batch is Python files, or C++ files, or C# files, and never a mixture.
2. **A single-file binding gets its own sub-agent.** `README.md`, `pyproject.toml`, and `tox.ini` each
   bind to a checklist nothing else uses. `platformio.ini` travels with `library.json`, and `CLAUDE.md`
   with `AGENTS.md`. A skill is one such unit rather than one per file,
   so its `SKILL.md` and its `references/*.md` travel together, because the progressive-disclosure
   rules judge a reference file against the `SKILL.md` that loads it. The documentation package
   under `docs/` is one sub-agent holding `/api-docs` alone.
3. **Roughly eight files per source batch**, sharing a package or a directory. Forty sub-agents cap
   the run, twelve run at once, and units beyond forty merge by shared checklist rather than
   dropping files.

Each sub-agent loads ONLY the checklists its own batch binds to, and receives ONLY the rule-ledger
rows Step 2 built from those checklists.

Only the per-file sweep fans out. Every other step runs on the main agent, because the rule ledger, the
cross-file consistency pass, the symbol usage pass, categorization, the guards, the verification, and
the report each need the whole file set in one view or sit on a trust boundary.

The fan-out therefore carries a SECOND return value. Every batch sub-agent returns its findings AND the
declaration and reference rows Pass 11 defines, covering each symbol its files declare and each symbol
its files reference. Findings alone would leave the main agent unable to run Pass 11 at all, because a
symbol declared in one batch and consumed in another is invisible to both sub-agents while the main
agent never reads their files.

Do NOT use the `Explore` agent type for sweep work. Explore returns summaries rather than
verbatim citations and breaks the "verbatim checklist quote" discipline.

### Step 2: Load applicable style skill checklists

Run Pass 1 from [detection-passes.md](references/detection-passes.md), which builds the rule ledger
every later pass reports against.

For every distinct style skill the bindings resolve to, invoke that skill and load its full
verification checklist along with every reference file the skill mentions. The loaded
checklists are the only source of truth for "applicable style point." A convention not present
in any loaded checklist is NOT a violation.

Tag every ledger row with the batch that will consume it, so Step 5 hands each sub-agent its own rows
rather than the whole ledger.

### Step 3: Run the deterministic gates

The project's own tools already decide every rule that can be decided mechanically, and they decide it
correctly every time. Run them FIRST, read their output as findings, and spend the sweep on the rules
no tool can check.

Run only the READ-ONLY forms. Bare `tox` and `tox -e lint` are FORBIDDEN during an audit, because the
`lint` environment reformats the source, auto-fixes it, and purges its stubs, which mutates the very
code under audit.

| Tool                                          | Read-only invocation                            |
|-----------------------------------------------|-------------------------------------------------|
| ruff lint rules, for Python files             | `ruff check --no-fix --output-format=json .`    |
| ruff formatting, for Python files             | `ruff format --diff .`                          |
| mypy, where the project configures it         | `mypy .`                                        |
| clang-format, for C++ files                   | `clang-format --dry-run --Werror <files>`       |
| clang-tidy, where a `.clang-tidy` file exists | `clang-tidy <files>`                            |

A tool the project does not configure is skipped, and its absence is no finding. Report a tool that
failed to run as a gap in the coverage the report states rather than as a clean result.

Fold each diagnostic into the report as an ordinary finding, citing the tool and its rule code in
place of the checklist quote, at HIGH confidence. A diagnostic a tool produced needs no adversarial
verification in Step 8, because the tool IS the external check.

Then narrow the Step 5 sweep. The passes still run, and their scope shrinks to what the tools cannot
decide:

- **Delegated to the tools, so the sweep reports nothing on its own authority.** Line length,
  indentation, blank-line counts, quote and string form, trailing commas, import sorting and grouping,
  and every rule carried by a ruff code the project enables.
- **Kept for the sweep, because no tool decides them.** Documentation quality in every form, which is
  length proportionality, redundancy with the signature, behavioral scope, sentence length, mood,
  separator punctuation, positive description, and spelling. Identifier vocabulary, meaning full words
  against the abbreviations the checklist enumerates. Element and section ORDERING where the checklist
  states an order the formatter does not enforce. Visibility placement. Cross-file consistency. Every
  cross-skill conflict. Symbol visibility against actual usage, together with every asset no consumer
  references, which ruff reaches only for imports outside `__init__.py`, for locals, and for arguments,
  and carries no rule for at module level.

State in the report which tools ran and which rules the sweep therefore delegated.

### Step 4: Sweep the project layout

Run Pass 10 from [detection-passes.md](references/detection-passes.md), which defines the four-part
procedure and the finding shape this sweep reports in. The directory tree is a style surface no
per-file pass can reach, because a pass reading a file cannot report the file that is MISSING. The
sweep runs ONCE, on the main agent, before the per-file batches fan out, never inside a batch sub-agent.

Run it ONLY when the resolved target is a project root, meaning a whole repository. A package directory
or a single file carries no tree to judge, so the sweep is SKIPPED, and the Step 0 plan and the Step 9
coverage ledger each record it as `skipped-not-a-project-root`. In change mode, run it only when the
change set CREATES or DELETES files, because only those alter the tree, and record an edit-only change
set as `skipped-no-created-or-deleted-files`. Silence is never coverage, so an unrecorded skip reads as
a clean tree that nothing ever checked. Every layout finding passes through the Step 7 guards like
every other candidate.

### Step 5: Line-by-line sweep

Run passes 2 through 9 and pass 11 from [detection-passes.md](references/detection-passes.md) in order,
over ONE traversal of each file rather than one traversal per pass. Passes 2 through 6 cover
Dimension A, passes 7 and 8 cover Dimension B, pass 9 covers Dimension C, and pass 11 covers
Dimension D.

For every file in scope, walk top to bottom. For every line, evaluate against every applicable
checklist item. Track four parallel dimensions.

**Dimension A — Structural style:** Element ordering, imports, formatting, naming, type
annotations, error-handling patterns, and file-section ordering. The source of truth is the
loaded style skill's main checklist.

**Dimension B — Comment and docstring quality:** Apply the loaded style skill's docstring and
comment checklist to every comment, docstring, and inline annotation. Typical findings include
typos, grammar errors, and prose padded with restatements or trivia the reader can infer from the
code. They also include documentation whose length tracks the size of the code instead of the
difficulty of understanding it, docstrings that restate the type signature, and comments that
narrate obvious code behavior. Documentation that describes how the asset is used in the project,
such as the pipeline stage that calls it or the feature that depends on it, is a finding whenever
the text leaves the behavior of the asset itself. A common content issue is a sentence exceeding
40 words. Findings further include prose that separates clauses with a semicolon or an em-dash
where only full stops and commas belong, together with contrastive or historical framing ("does X,
not Y" or "formerly did Y") that should state present behavior positively.

This dimension judges the form of the documentation. A stale reference to a renamed symbol, a
removed feature, or a closed issue, and a docstring claim that disagrees with the signature or the
observable behavior, are factual findings that belong to `/audit-facts`, because confirming them
requires reading the implementation.

**Dimension C — Cross-file consistency:** Naming, ordering, and idiom drift across the file
set. Examples include the same field named differently in two sibling classes, or one module
following a convention that adjacent modules ignore.

**Dimension D — Symbol visibility and usage:** Each symbol's declared tier against the widest boundary
its consumers actually cross, and each symbol's consumer set against emptiness. The five findings are a
private name on a symbol another module references, a public name on a symbol only its own module
references, a cross-package symbol missing from its package's export list, an export list entry no
outside package imports, and an asset nothing references at all. Every one is a claim about ABSENCE, so
each is confirmed by a repository-wide search before it is reported, and Guard 15 discards the
candidates resting on a consumer no written reference reveals.

While traversing each file, record the symbols it DECLARES and the symbols it REFERENCES. Pass 11
reconciles the two tables on the main agent after the traversal closes, so the traversal collects them
rather than paying for a second reading of every file in scope.

List ALL violations in each section. Do NOT stop at the first.

For Large-tier audits, spawn one `general-purpose` sub-agent per batch under the Step 1 rules. Each
sub-agent receives its batch's file paths, ONLY the checklists those files bind to, ONLY the
rule-ledger rows built from those checklists, the Step 3 diagnostics for those files, and the
categorization rules. Sub-agents return findings in the output format defined below, together with the
Pass 11 declaration and reference rows for their own files. The main agent synthesizes after all
sub-agents complete, then runs passes 9 and 11 over the merged result.

For Small and Medium tiers, the main agent performs all sweep work sequentially.

### Step 6: Categorize findings

Categorize every violation using one of:

| Category      | Meaning                                                              |
|---------------|----------------------------------------------------------------------|
| BLOCKING      | Checklist explicitly states "MUST" or "blocks release"               |
| STANDARD      | Checklist convention without an explicit blocking flag               |
| INCONSISTENCY | File or file set mixes schemes the checklist permits only separately |
| CONFLICT      | Two loaded style skills disagree on the same point                   |

Also assign a confidence tier to every finding:

| Confidence | Meaning                                                                |
|------------|------------------------------------------------------------------------|
| HIGH       | Verbatim checklist quote and verbatim source quote both present        |
| MEDIUM     | One quote verbatim, the other requires interpretation                  |
| LOW        | Pattern detected but checklist/source mapping is inferred, not literal |

### Step 7: Apply the false-positive guards

Walk every candidate through every guard in
[false-positive-guards.md](references/false-positive-guards.md), in order. The rule-ledger guard runs
first and removes the most candidates. Discard everything a guard rejects, and record the count of
discarded candidates for the report's triage header.

### Step 8: Verify the surviving findings

Run the two checks in [verification-protocol.md](references/verification-protocol.md), in order:

1. **Citation verification**, against every surviving finding with no sampling. Confirms the Checklist
   point quote appears in the loaded checklist and the Current state quote appears at the cited line.
2. **Adversarial refutation**, against every BLOCKING and CONFLICT finding. A fresh `general-purpose`
   sub-agent per finding, instructed to refute it and to answer REFUTED under uncertainty.

Both checks are external, testing the finding against the files and against a reader who never saw the
sweep. They catch the failure mode this audit produces most often, which is misapplying a checklist
rule. A Step 3 diagnostic skips both checks, because the tool that produced it already is the external
check. Record every count the protocol names, because the report's triage header carries them.

### Step 9: Assemble the coverage ledger

Build the coverage ledger the report carries under its triage header, in the shape
[verification-protocol.md](references/verification-protocol.md) defines. It records the per-binding
count of files in scope, files swept, and files UNAUDITED, then names every UNAUDITED file by path,
the sub-agent and batch count, the gates that ran and the gates that failed to run, the layout pass
status, the symbol usage pass status with the declaration and reference counts it reconciled, and the
revision whenever the scope was narrowed to a change set. It exists so a thin pass is visible rather
than silent.

### Step 10: Produce the findings report

Use the output format below. Open with the triage header from
[verification-protocol.md](references/verification-protocol.md), which carries the finding counts by
category and confidence, the tools Step 3 ran, and every discard count.

Skip compliant items entirely. Report every surviving finding at every confidence tier by default,
which covers LOW alongside HIGH and MEDIUM. Narrow the report to HIGH and MEDIUM only when the user
explicitly asks for it via `--min-confidence medium` or equivalent invocation.

The confidence tier stays on every finding, so a reader triages by tier rather than by trusting that
the report was filtered. LOW means the checklist and source mapping is inferred rather than literal,
and it never excuses a finding from the verbatim checklist quote the Discipline section requires. LOW
findings sit in the trailing `Appendix: LOW confidence` section the protocol defines rather than
interleaved into the file groups, so the body of the report reads at one confidence level.

---

## Output format

Open the report with the triage header from
[verification-protocol.md](references/verification-protocol.md), then the Step 9 coverage ledger. For
multi-file targets, group the HIGH and MEDIUM confidence findings hierarchically: file -> category ->
findings. Within each file, order findings by severity: BLOCKING -> INCONSISTENCY -> CONFLICT ->
STANDARD. Collect LOW confidence findings into the trailing `Appendix: LOW confidence` section, ordered
by the same sequence.

Step 4 layout findings belong to no file, so they sit in a leading `Project layout` group ahead of the
per-file groups, headed by the archetype the sweep resolved and the indicators that resolved it.

Each finding uses this structure:

```text
[Category]: <BLOCKING | STANDARD | INCONSISTENCY | CONFLICT>
[Confidence]: <HIGH | MEDIUM | LOW>
Skill: <skill name, or the tool name for a Step 3 diagnostic>
Checklist point: "<verbatim quote from the skill's checklist or reference file, or the tool's rule code>"
Location: <path>:<line>-<line>
Current state: "<verbatim quote from the file>"
Required state: <concrete example or the checklist's "should be" form>
Suggested fix: <concrete textual edit>
Approval: <REQUIRED when the fix breaks the public API or alters public behavior, naming what breaks>
```

The Approval trigger covers REMOVAL alongside renaming and re-signaturing. Deleting a symbol, dropping
an `__all__` entry, and demoting a public name to an underscore each break a caller exactly as a rename
does, and a Pass 11 fix is a removal in three of its five forms.

When the same checklist point is violated multiple times within a file, collapse to a single
finding with a count and representative line citations:

```text
[Category]: STANDARD
[Confidence]: HIGH
Skill: /python-style
Checklist point: "Full words used (no abbreviations like pos, idx, val)"
Location: module.py:47, 89, 112, 134 (4 occurrences)
Current state: "idx" used as loop variable
Required state: "index"
Suggested fix: Rename idx -> index throughout module.py.
```

A Step 4 layout finding uses its own shape, defined under Pass 10 in
[detection-passes.md](references/detection-passes.md), because `Location: <path>:<line>` cannot cite a
file that does not exist.

A Pass 11 symbol usage finding uses the shape above with one added `Consumer set` field, defined under
Pass 11 in [detection-passes.md](references/detection-passes.md). `Current state` stays a verbatim
quote of the declaration or of the `__all__` block, and the added field carries the consumer evidence
together with the search that established it.

---

## Discipline

You MUST adhere to the following discipline during every audit.

- Anchor every finding to a verbatim checklist quote. No checklist quote, no finding.
- Never invent conventions. If a behavior is not in a loaded checklist, it is not a style
  violation.
- Never flag factual errors, missing content, or source-code mismatches. Those belong to
  `/audit-facts`.
- Never flag a defect, an edge case, or a runtime cost. Those belong to `/audit-correctness` and
  `/audit-performance`.
- Never restructure, restyle, or refactor. This skill produces findings only.
- Never flag subjective preferences (tone, ordering, terminology) unless the loaded checklist
  explicitly requires the convention.
- If a file contains an auto-generated block or a documented exception, note the exception and
  skip its enclosing range.

---

## Related skills

| Skill                | Relationship                                                                    |
|----------------------|----------------------------------------------------------------------------------|
| `/audit-project`     | Orchestrator that runs this audit in wave 1 and merges it with the siblings     |
| `/audit-facts`       | Sibling audit for factual accuracy of documentation against source code         |
| `/audit-correctness` | Sibling audit for active and latent bugs and broken stated contracts            |
| `/audit-performance` | Sibling audit for cost, speed, memory use, and dtype predictability             |
| `/python-style`      | Provides the Python checklist, loaded when scope contains Python files          |
| `/cpp-style`         | Provides the C++ checklist, loaded when scope contains C++ files                |
| `/csharp-style`      | Provides the C# checklist, loaded when scope contains C# files                  |
| `/readme-style`      | Provides the README checklist, loaded when scope contains README files          |
| `/pyproject-style`   | Provides the pyproject.toml checklist, loaded when that file is in scope        |
| `/tox-config`        | Provides the tox.ini checklist, loaded when that file is in scope               |
| `/platformio-config` | Provides the platformio.ini and library.json checklist, loaded for those files  |
| `/api-docs`          | Provides the Sphinx docs checklist, loaded when scope contains docs files       |
| `/skill-design`      | Provides the skill, CLAUDE.md, and AGENTS.md checklist, for those files         |
| `/project-layout`    | Provides the directory and issue template checklist, for roots and .github      |
| `/explore-codebase`  | Provides project structure context, invoke first on an unfamiliar codebase      |

---

## Proactive behavior

Invoke this skill when the user asks to audit a file, package, or project for style compliance.

Apply its findings LAST, after the fixes from `/audit-facts`, `/audit-correctness`, and
`/audit-performance`, when auditing the same file end to end. Style fixes applied before factual
corrections waste effort on prose that may be rewritten, and correctness and optimization fixes change
the code whose form this skill judges. That is a FIX order, and `/audit-project` decides the RUN order.

Do NOT make code or documentation changes during the audit. Present findings and wait for user
direction.

---

## Verification checklist

You MUST verify the audit output against this checklist before presenting it to the user.

```text
Style Compliance Audit Output:
- [ ] Step 0 plan produced and confirmed by user before sweep began
- [ ] Plan stated whether the symbol usage sweep runs in full, runs partially, or is skipped
- [ ] Tier classified (small/medium/large) and agent allocation matched the table
- [ ] For Large tier, batches built by binding with no batch carrying more than two checklists
- [ ] For Large tier, every single-file binding, each skill unit, and the docs package given its own sub-agent
- [ ] For Large tier, each sub-agent loaded only its own batch's checklists and rule-ledger rows
- [ ] Sub-agents held to 40 for the run and 12 in flight, merging to fit rather than dropping files
- [ ] Scope narrowed to a change set only on explicit request, with the revision recorded in the report
- [ ] Step 1 file binding executed (every file in scope mapped to its applicable style skill)
- [ ] Step 2 checklists loaded for every applicable style skill
- [ ] Step 3 deterministic gates run in their read-only form for every tool the project configures
- [ ] Bare `tox` and `tox -e lint` never run during the audit
- [ ] Tool diagnostics folded in as findings citing the tool and rule code, at HIGH confidence
- [ ] Rules the tools decide delegated to them rather than re-derived by the sweep
- [ ] Report states which tools ran, which failed to run, and which rules were delegated
- [ ] Step 4 layout sweep run once on the main agent for a project-root target, before the batches fanned out
- [ ] Layout sweep skipped and recorded for a package, single-file, or edit-only change-set target
- [ ] Archetype resolved from the /project-layout key indicators, with the deciding indicators recorded
- [ ] Archetype tree loaded from project-layout/references/archetype-trees.md and diffed against the real tree
- [ ] Layout checklist presence and absence items applied (envs/, .github/ISSUE_TEMPLATE/, .netlify-site)
- [ ] Every file in scope walked top to bottom
- [ ] Rule ledger built in Pass 1, with every rule copied verbatim from a loaded checklist
- [ ] Detection passes 2 through 9 run in order, with pass 9 run on the main agent over the whole file set
- [ ] Pass 10 run on the main agent for a project-root target, or skipped with the reason recorded
- [ ] Pass 11 run on the main agent over the whole file set, or skipped with the reason recorded
- [ ] Declaration and reference rows collected during the single traversal, and returned by every batch sub-agent
- [ ] Reference table built across the whole repository even where the sweep was narrowed to a change set
- [ ] Every symbol's declared tier compared against the widest boundary its consumers actually cross
- [ ] Each package export list compared against the set of packages importing from it, in both directions
- [ ] Every Pass 11 candidate confirmed by a repository-wide search, with the search result quoted
- [ ] Every Pass 11 finding carries a Consumer set field, with Current state left a verbatim quote
- [ ] Citation verification re-ran each Pass 11 search rather than accepting the recorded result
- [ ] Guard 15 applied, so curated public API, runtime registrations, interface conformance, and generated
      declarations produced no usage finding, and test references counted as no consumer
- [ ] Unused imports, locals, and arguments left to ruff rather than re-derived by the usage pass
- [ ] All four dimensions evaluated (structural, comment/docstring quality, cross-file consistency,
      symbol visibility and usage)
- [ ] Every finding anchored to a verbatim checklist quote, or to a tool rule code
- [ ] Every finding cites <path>:<line>, or the expected or offending path for a layout finding
- [ ] Findings categorized (BLOCKING, STANDARD, INCONSISTENCY, CONFLICT)
- [ ] Every finding assigned a confidence tier (HIGH, MEDIUM, LOW)
- [ ] Repeated violations of the same checklist point collapsed with counts
- [ ] Every false-positive guard applied in order, with the discarded-candidate count recorded
- [ ] Citation verification run against every finding, with the checklist quote and source quote confirmed
- [ ] Every finding whose quote or line failed citation verification deleted rather than repaired
- [ ] Adversarial refutation run against every BLOCKING and CONFLICT finding, in fresh sub-agents
- [ ] Every refuted finding discarded, and the confirmed and refuted counts recorded
- [ ] Step 9 coverage ledger assembled, carrying files in scope, files swept, and files UNAUDITED by path
- [ ] Ledger states the sub-agent and batch count, the gates that ran, and the gates that failed to run
- [ ] Ledger states the layout pass status, and the revision whenever the scope was narrowed to a change set
- [ ] Ledger states the symbol usage pass status with the declaration and reference counts it reconciled
- [ ] Triage header present, carrying the category by confidence counts and every discard count
- [ ] Every confidence tier reported, with LOW included unless the user narrowed the report
- [ ] LOW confidence findings placed in the trailing appendix rather than interleaved
- [ ] No compliant items appear in the report
- [ ] No factual errors, missing content, or source mismatches appear (those belong to /audit-facts)
- [ ] No findings invented outside the loaded checklists
- [ ] Findings ordered: BLOCKING -> INCONSISTENCY -> CONFLICT -> STANDARD
- [ ] Cross-skill conflicts surfaced rather than silently resolved
- [ ] Suggested fixes are concrete textual edits, each carrying an Approval verdict
- [ ] Every fix that removes a symbol, an export entry, or a public name carries Approval: REQUIRED
- [ ] No file modifications made during the audit
```
