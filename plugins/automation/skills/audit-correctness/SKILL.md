---
name: audit-correctness
description: >-
  Performs a thorough correctness audit of source code, hunting for active and latent bugs the test suite leaves
  uncaught and for behavior that breaks the contract the code itself states. Ranks work by test coverage and reports
  only findings carrying a concrete trigger and result. Use when auditing a Python package, C++ firmware, or C# code for
  bugs, edge cases, races, or leaks, or when the user invokes /audit-correctness.
user-invocable: true
---

# Code correctness audit

Audits source code against the contract it states, reporting only defects that carry a concrete trigger, a concrete
result, and verbatim source citations.

You MUST read this entire skill, and load each reference file at the step that names it, before acting on that step. The
verification checklist at the end is mandatory before submitting findings.

---

## Scope

**Covers:**
- Auditing single files, directories, or full project trees for active and latent defects
- Behavior that fails the contract stated by a name, a signature, an annotation, or documentation, where the contract is
  the intended behavior and the code is the side to fix
- Pure implementation defects with no documentation involved, such as races, leaks, and off-by-one errors
- Edge cases, numeric representation defects, lifecycle and ordering defects, and API misuse
- Ranking the sweep by the regions the project's coverage machinery leaves unexercised
- Behavior the test suite executes without pinning, where a plausible wrong value would ship undetected

**Does not cover:**
- Cost, speed, memory use, and dtype predictability (see `/audit-performance`)
- Style, formatting, naming, and convention compliance (see `/audit-style`)
- Documentation claims that disagree with the code where the documentation is the side to fix (see `/audit-facts`)
- Code modifications, bug fixes, and new tests (this skill produces findings only)
- Codebase exploration (see `/explore-codebase`)

---

## Ownership ladder

Four audits partition the same files by which side is wrong and what the fix edits. `/audit-facts` owns a documentation
claim that disagrees with the code where the fix edits the DOCUMENTATION, and its own discipline states that the
implementation is authoritative by default. This skill owns the mirror case, where the stated contract is the intended
behavior and the fix edits the CODE.

When a docstring and an implementation disagree, walk this ladder in order and stop at the first hit.

| Rung | Test                                                                                 | Owner                |
|------|--------------------------------------------------------------------------------------|----------------------|
| 1    | A caller, a test, or an external schema is correct only under the documented reading | `/audit-correctness` |
| 2    | The documented behavior is enforced independently, by a guard or a shared constant   | `/audit-correctness` |
| 3    | The mismatch fails observably for an input the documentation declares supported      | `/audit-correctness` |
| 4    | The implementation is self-consistent and every caller already matches it            | `/audit-facts`       |
| 5    | Nothing above resolves it                                                            | Report as AMBIGUOUS  |

Rung 4 is the common case and produces nothing in this report. A rung 5 finding is reported at MEDIUM confidence, marked
AMBIGUOUS, with both candidate fixes stated side by side, an explicit note that the owner must decide, and a
cross-reference to `/audit-facts` so the same mismatch is filed once. Rung 5 should stay rare. More than roughly one
finding in ten landing there means the ladder is being applied lazily.

---

## Language coverage

The audit applies to Python, C++, and C# equally, and a project holding no Python at all is fully in scope. Passes 1
through 7 and Pass 10 are language-neutral, because contracts, boundaries, types, unwinding, sharing, and call sequences
are properties of any program. Pass 8 holds the defects only one language can produce. Pass 9 consumes the coverage
ranking, which each language instruments differently.

Two mechanisms carry per-language instruments, and neither one may be treated as a prerequisite for a language that
lacks it.

| Concern           | Python                                       | C++                                           | C#                                            |
|-------------------|----------------------------------------------|-----------------------------------------------|-----------------------------------------------|
| Coverage ranking  | coverage.py data and the tox coverage report | The PlatformIO or CTest suite, read directly  | The Unity Test Framework suite, read directly |
| Declared contract | Annotations and the Google-style docstring   | The signature, `const`, and the Doxygen block | The signature, nullability, and the XML doc   |

Where a language ships no coverage instrument, build the ranking by reading its test suite and mapping each test to the
symbols it exercises, then treat every symbol no test reaches as T0. Embedded firmware frequently carries no unit tests
at all, which puts its entire runtime path at T0 rather than out of scope.

---

## Evidence model

**Trigger and result** carry every finding. A finding is reportable only when the trigger is written as an executable
expression, a numbered call sequence, or a line-numbered interleaving, AND the result is written as a concrete value,
exception, corruption, or hang. A candidate that resists being written that way is discarded rather than softened. This
filter removes the most candidates of any rule in the skill.

**Coverage tier** records why a region escaped the test suite. The tiers are language-neutral, each language fills them
from its own instrument, and [detection-passes.md](references/detection-passes.md) defines all four alongside the pass
that consumes them. Read the project's actual `[tool.coverage.report] fail_under` and `[tool.coverage.run] branch`
values in Step 1 rather than assuming either, because the `branch` value decides whether the T3 tier holds anything at
all. A C++ or C# target has no equivalent gate, so its tiers come from reading the test suite directly.

**Severity** orders the report, and [finding-catalog.md](references/finding-catalog.md) defines its four levels
alongside the per-category guidance that assigns them.

---

## Workflow

You MUST follow these steps when this skill is invoked.

Copy this progress checklist into your response and check off items as you complete them:

```text
Audit Progress:
- [ ] Step 0: Plan produced and confirmed
- [ ] Step 1: Target resolved, audit tier selected, prerequisites recorded
- [ ] Step 2: Coverage ranking established
- [ ] Step 3: Contract, state, and callgraph ledgers built
- [ ] Step 4: Sweep passes 2 through 10 complete
- [ ] Step 5: Ownership adjudicated and findings categorized
- [ ] Step 6: False-positive guards applied
- [ ] Step 7: Findings verified (citation, refutation)
- [ ] Step 8: Coverage ledger assembled
- [ ] Step 9: Report produced
```

### Step 0: Produce audit plan and pause

Emit a plan before any sweep work fires. The plan must list:

- Files in scope (resolved absolute paths), grouped by language
- The project archetype, and whether coverage artifacts are present, stale, or absent
- Tier classification (small, medium, or large)
- Expected finding categories

Pause for user confirmation or a "proceed" signal. This catches misidentified targets before tokens burn on the wrong
scope. A user who narrows the scope here has that narrowing recorded in the Step 8 coverage ledger.

### Step 1: Resolve target and record prerequisites

Resolve the target into the set of files in scope. For a directory or project-root target, every source file under the
target is in scope. There is no "covered area" reduction. Enumerate the audited files and, separately, the files read as
authority rather than audited:

```bash
git ls-files '*.py' '*.pyi' '*.h' '*.hpp' '*.cpp' '*.cs'   # audited
git ls-files 'tests/*' 'pyproject.toml' 'tox.ini'          # authority
```

Whole-repository coverage is the default and stays the default. Narrow to a change set ONLY when the user asks for that
in the invocation, resolving it with `git diff --name-only <base>...HEAD` for a branch, `git diff --name-only <commit>`
for one commit, or `git status --porcelain` for the working tree. A narrowed run still reads every surviving file in
full, because a partial read hides the context the passes depend on. Record the narrowing and the revision it resolved
against in the Step 8 coverage ledger, so the report states what it did not cover.

Bind each file to the style skill that supplies its citable authority:

| File pattern            | Authority       |
|-------------------------|-----------------|
| `*.py`, `*.pyi`         | `/python-style` |
| `*.h`, `*.hpp`, `*.cpp` | `/cpp-style`    |
| `*.cs`                  | `/csharp-style` |

Record the prerequisites that apply to the languages in scope, before any verdict:

1. The project archetype, read from the `envlist` in `tox.ini`. Full Python and C++ extension projects carry
   `{pyXXX}-test` and `coverage` environments. Reduced Python projects omit both by design, and C++ docs-only projects
   carry `envlist = docs` alone.
2. Python only. The actual test matrix. Core libraries use `{py312, py313, py314}-test` and applications use a single
   version, so read the matrix rather than assuming one.
3. C++ only. The archetype of every C++ file, embedded (a `platformio.ini` at the project root) or extension (nanobind
   headers under a CMake build), plus the target boards, because `int` width and therefore every promotion result varies
   across them.
4. Python only. The coverage settings, read from `pyproject.toml`. Record the `[tool.coverage.run] branch` value, which
   decides whether T3 exists at all, the `[tool.coverage.report] fail_under` value, which some projects leave unset, and
   the `[tool.coverage.run] omit` list, which enumerates the T0 modules.
5. C# only. Whether the project is a Unity project, which decides whether the component lifecycle hazards of Pass 8
   apply, and where its test assemblies live.
6. Every language. The test suite's location and shape, which supplies the Step 2 ranking wherever no coverage
   instrument exists.

Classify the audit tier:

| Tier   | Indicators                         | Execution                                               |
|--------|------------------------------------|---------------------------------------------------------|
| Small  | 1 file                             | Main agent, sequential                                  |
| Medium | 2 to 9 files                       | Main agent, file-by-file                                |
| Large  | 10 or more files or a project root | Parallel `general-purpose` sub-agents over file batches |

A Large-tier audit BATCHES rather than fanning out per file. Every sub-agent re-receives the whole instruction payload,
so fanning out per file pays that payload once per file and costs more than the sweep it parallelizes. Build the batches
under two rules:

1. **One authority per batch.** Group by the style skill the binding table above assigned, so a batch holds Python files
   or C++ files or C# files, never a mixture. A sub-agent then loads one authority rather than three, and it never
   judges a file against another language's rules.
2. **Roughly eight files per batch**, sharing a package or a directory where the authority allows it. Forty sub-agents
   cap the run, twelve run at once, and batches beyond forty merge by authority.

Record the batch count in the Step 8 coverage ledger.

Only the sweep passes fan out. Every other step runs on the main agent, because the coverage ranking, the ledgers,
ownership adjudication, guard application, verification, and the report each need the whole-project view or sit on a
trust boundary.

Do NOT use the `Explore` agent type for sweep work. Explore returns summaries rather than the verbatim quotes and
line-level traces this skill's evidence standard requires.

### Step 2: Establish the coverage ranking

Coverage decides what is examined FIRST. It never decides what is examined at all.

For C++ and C# targets, and for any Python target whose archetype omits the coverage environments, skip straight to the
test-suite reading described at the end of this step. Those languages carry no coverage.py artifacts, and their absence
is a property of the archetype rather than a finding.

For a Python target that has them, read the existing artifacts before executing anything:
`reports/coverage_html/index.html` with its per-file pages, and coverage.py's default `coverage.xml` at the project
root. Query an existing data file only through a non-mutating command:

```bash
coverage report --data-file=reports/.coverage --show-missing --fail-under=0 --keep-combined
```

Both flags are mandatory. The `fail_under = 100` gate fires on every rendering command, and a reporting command without
`--keep-combined` deletes the per-version data files it combines, which would destroy the project's coverage data during
a read-only audit.

When the artifacts are absent or stale, ask the user before regenerating. On approval, run `tox -e <matrix-member>-test`
and then `tox -e coverage`, in that order. Bare `tox` and `tox -e lint` are FORBIDDEN during an audit, because the
`lint` environment reformats the source, auto-fixes it, and purges its stubs, which mutates the very code under audit.

Where no coverage machinery exists, build the ranking from the language's own test suite, as the Language coverage
section above prescribes. Every symbol no test reaches is T0, every symbol a test merely constructs or smoke-calls is
T2, and every branch arm is T3, because no instrument observed any of them.

Assign every line in scope a coverage tier and rank the sweep by tier, T0 first.

### Step 3: Build the ledgers

Run Pass 1 from [detection-passes.md](references/detection-passes.md) on the main agent. It reads every source file top
to bottom once and produces three ledgers, the CONTRACT ledger, the STATE ledger, and the CALLGRAPH ledger. Every later
pass consumes them.

### Step 4: Run the sweep passes

Run passes 2 through 10 from [detection-passes.md](references/detection-passes.md) in ranked order, over ONE traversal
of each file rather than one traversal per pass. Each pass asks one question of every line, and the file also holds the
named CEAI procedure that Pass 2 and several categories call. Pass 8 runs over C++ and C# files alone, and Pass 9
consumes the Step 2 ranking.

For every candidate, classify it against [finding-catalog.md](references/finding-catalog.md), which supplies each
category's definition, mechanical detection procedure, required evidence, and severity guidance. List ALL candidates in
each pass. Do NOT stop at the first.

For Large-tier audits, spawn one `general-purpose` sub-agent per file batch. Each sub-agent receives its batch's file
paths, the Step 2 ranking rows and the Step 3 ledger rows for those files alone, the reference files, and the output
format. Sending a sub-agent ranking or ledger rows for files it does not hold wastes the payload it pays for. The main
agent synthesizes after all sub-agents complete.

### Step 5: Adjudicate ownership and categorize

Run the ownership ladder against every contract-versus-behavior candidate and route rung 4 results to `/audit-facts`
rather than reporting them here. Assign every surviving candidate a category, a severity, and a confidence tier, and
collapse a defect satisfying several categories onto the most specific one, listing the others as tags.

| Confidence | Meaning                                                                   |
|------------|---------------------------------------------------------------------------|
| HIGH       | Contract and implementation both quoted verbatim, the trace is mechanical |
| MEDIUM     | Both quotes present, the trace requires interpretation                    |
| LOW        | Pattern detected but the trigger is inferred rather than derived          |

### Step 6: Apply the false-positive guards

Walk every candidate through every guard in [false-positive-guards.md](references/false-positive-guards.md), in order.
The trigger requirement runs first and removes the most candidates. Discard everything a guard rejects, and record the
count of discarded candidates for the report's triage header.

### Step 7: Verify the surviving findings

Run the two checks in [verification-protocol.md](references/verification-protocol.md), in order:

1. **Citation verification**, against every surviving finding with no sampling. Confirms each quoted string appears at
   the line it is cited to, and deletes the finding when it does not.
2. **Adversarial refutation**, against every CRITICAL and HIGH finding. A fresh `general-purpose` sub-agent per finding,
   instructed to refute it and to answer REFUTED under uncertainty.

Both checks are external, testing the finding against the source and against a reader who never saw the sweep. They
catch the failure mode this audit produces most often, which is asserting a trigger no reachable path supplies. Record
every count the protocol names, because the Step 8 ledger and the report's triage header carry them.

### Step 8: Assemble the coverage ledger

Build the ledger that opens the report:

```text
| Language | Files in scope | Files audited | Files skipped | Tiers swept | Coverage source |
|----------|----------------|---------------|---------------|-------------|-----------------|
| Python   | 24             | 24            | 0             | T0-T3       | existing html   |
| C++      | 6              | 6             | 0             | T0, T2, T3  | test suite read |
```

List every skipped file by path with its reason, state whether coverage data was present, stale, absent, or regenerated,
and state the Large-tier batch count. Skipping is allowed only when the user narrowed the scope in Step 0 or Step 1,
when a file is generated, or when a file is unreadable. A run narrowed to a change set names the revision it resolved
against here.

### Step 9: Produce the findings report

Use the output format below.

Report every surviving finding at every confidence tier by default, which covers LOW alongside HIGH and MEDIUM. Narrow
the report to HIGH and MEDIUM only when the user explicitly asks for it via `--min-confidence medium` or equivalent
invocation.

The confidence tier stays on every finding, so a reader triages by tier rather than by trusting that the report was
filtered. LOW means the trigger is inferred rather than derived, and it never lowers the evidence floor. A candidate
carrying no concrete trigger and no concrete result is still deleted by Guard 1 rather than demoted to LOW. LOW findings
sit in the trailing `Appendix: LOW confidence` section the protocol defines rather than interleaved into the file
groups, so the body of the report reads at one confidence level.

---

## Finding categories

Each category is defined in full in [finding-catalog.md](references/finding-catalog.md). Load that file before
classifying any candidate.

| Category                     | One-line definition                                                      |
|------------------------------|--------------------------------------------------------------------------|
| CONTRACT_VIOLATION           | The body fails the contract its name, signature, or documentation states |
| UNHANDLED_EDGE_CASE          | A legal boundary value the body mishandles                               |
| TYPE_CONTRACT_BREACH         | A declared type permits or promises what the body cannot deliver         |
| NUMERIC_DEFECT               | A result wrong because of the numeric representation                     |
| STATE_LIFECYCLE_DEFECT       | An object used outside the window in which it is valid                   |
| RESOURCE_LEAK                | An acquired resource left unreleased on a reachable path                 |
| DURABILITY_DEFECT            | Persisted bytes a crash or a second writer can leave wrong or partial    |
| CONCURRENCY_DEFECT           | A defect needing two execution contexts to manifest                      |
| ERROR_HANDLING_DEFECT        | The error path itself is wrong                                           |
| SHARED_MUTABLE_STATE_DEFAULT | State unintentionally shared across calls, instances, or importers       |
| ALIASING_MUTATION            | A callable mutates an object it does not own, or leaks a live reference  |
| CONTROL_FLOW_DEFECT          | The shape of the control flow is wrong                                   |
| API_MISUSE                   | A call that violates its callee's own contract                           |
| ORDERING_PROTOCOL_DEFECT     | Operations performed out of a required sequence                          |
| UNIT_SCALE_CONSTANT_DEFECT   | A quantity carried in the wrong unit or scale                            |
| UNTESTED_PATH_DEFECT         | A defect living specifically in an unexercised region                    |
| CPP_LOW_LEVEL_DEFECT         | C++ and embedded defects with no Python analogue                         |
| CSHARP_UNITY_DEFECT          | C# and Unity defects with no Python analogue                             |
| TEST_ORACLE_GAP              | A behavior the suite executes without pinning                            |

---

## Output format

Open the report with the triage header from [verification-protocol.md](references/verification-protocol.md), then the
Step 8 coverage ledger. The header carries the finding counts by severity and confidence together with every discard
count the guards and the Step 7 checks produced. Group HIGH and MEDIUM confidence findings hierarchically: file, then
category, then severity, ordered most severe first. Collect LOW confidence findings into the trailing `Appendix: LOW
confidence` section, ordered most severe first.

Each finding uses this structure:

```text
[Category]: <category name from the catalog>
[Severity]: <CRITICAL | HIGH | MEDIUM | LOW>
[Confidence]: <HIGH | MEDIUM | LOW>
Location: <path>:<line>-<line>
Coverage tier: <T0 | T1 | T2 | T3 | COVERED> or N/A
Contract: "<verbatim quote of the promise>" [NAME | SIG | ANN | DOC | ENF] <path>:<line>
Implementation: "<verbatim quote of the code that breaks it>"
Trigger: <executable expression, numbered call sequence, or line-numbered interleaving>
Result: <concrete wrong value, exception, corruption, or hang>
Suggested fix: <concrete code change, described rather than applied>
Approval: <REQUIRED when the fix breaks the public API or alters public behavior, naming what breaks>
```

When one root cause repeats across several sites, collapse to a single finding with a count and representative line
citations. For a rung 5 AMBIGUOUS finding, add a `Resolution: AMBIGUOUS` line and state both candidate fixes side by
side. For a finding with no documented contract, set the Contract line to the implied promise and tag its source NAME or
SIG.

---

## Discipline

You MUST adhere to the following discipline during every audit.

- Report nothing without a concrete trigger and a concrete result, under the evidence model above and Guard 1.
- Read the full body of every callable, and follow the calls it delegates to, before judging it.
- Quote both the contract and the implementation verbatim, each with its own `<path>:<line>`.
- Keep every sentence the report itself writes, outside a verbatim quote, under 40 words and separated by full stops and
  commas alone.
- Run the ownership ladder before every contract-versus-behavior finding, and route rung 4 results to `/audit-facts`.
- Verify every external library contract by reading the installed package rather than from memory.
- Treat coverage as a ranking. Never report a tier, a percentage, or a missing-line list as a defect, and never report
  the absence of a test.
- Run only read-only commands, plus the two non-mutating tox environments after the user agrees.
- Never invent an exemption. An exemption exists only where a loaded skill writes it down, and you MUST quote that
  clause before applying it. Shared corpus, house convention, text byte-identical in a sibling repository, long-standing
  code, and "it reads fine" are none of them, so a real finding survives wherever else the same text appears.
- Never fix, refactor, or add a test. This skill produces findings only.
- Treat `console.enable()` and `console.disable()` calls as correct at every library tier.

---

## Related skills

| Skill                   | Relationship                                                                           |
|-------------------------|----------------------------------------------------------------------------------------|
| `/audit-project`        | Orchestrator that runs this audit in wave 2 and merges its findings with the siblings  |
| `/audit-facts`          | Sibling audit owning the same mismatch when the documentation is the side to fix       |
| `/audit-performance`    | Sibling audit for cost, speed, memory use, and dtype predictability                    |
| `/audit-style`          | Sibling audit for style, formatting, documentation quality, and convention compliance  |
| `/python-style`         | Supplies the error-handling, None-check, and keyword-argument rules cited as authority |
| `/cpp-style`            | Supplies the fixed-width type and per-archetype embedded rules cited as authority      |
| `/csharp-style`         | Supplies the Unity lifecycle and null-handling rules cited as authority                |
| `/tox-config`           | Defines the coverage environments and the commands this skill may run                  |
| `/pyproject-style`      | Defines the coverage gate, the omit list, and the exclude corpus that set the tiers    |
| `/explore-dependencies` | Provides ataraxis API snapshots, invoke before judging any library call                |
| `/explore-codebase`     | Provides project structure context, invoke first when auditing an unfamiliar codebase  |

---

## Proactive behavior

Invoke this skill when the user asks to find bugs, hunt for defects, or audit code for correctness, edge cases, races,
or leaks. A request that names a directory or a repository covers every source file under it by default, and the Step 0
plan is where the user narrows that scope.

Fix this audit's findings after `/audit-facts` and before `/audit-performance` and `/audit-style`. Settling which side
of a documentation mismatch is authoritative comes first, and optimization and style work apply to code whose behavior
is already correct. That is a FIX order, and `/audit-project` decides the RUN order.

Do NOT make code changes during the audit. Present findings and wait for user direction.

---

## Verification checklist

You MUST verify the audit output against this checklist before presenting it to the user.

```text
Code Correctness Audit Compliance:
- [ ] Step 0 plan produced and confirmed by user before sweep began
- [ ] Step 1 prerequisites recorded for every language in scope, including the archetype, the actual test matrix, and the branch and fail_under settings
- [ ] Tier classified (small/medium/large) and agent allocation matched the table
- [ ] For Large tier, batched by authority with no language mixing, within 40 and 12 in flight, merging to fit
- [ ] For Large tier, each sub-agent received only the ranking and ledger rows for its own batch
- [ ] Scope narrowed to a change set only on explicit request, with the revision recorded in the ledger
- [ ] Coverage ranking built for every language in scope, from artifacts for Python and from the test suite for C++ and C#
- [ ] A language lacking a coverage instrument ranked by reading its tests, never dropped from scope
- [ ] Coverage artifacts read before any command was run
- [ ] Coverage regenerated only after explicit user consent, and only via the test and coverage environments
- [ ] Bare `tox` and `tox -e lint` never run during the audit
- [ ] Every coverage query carried both --fail-under=0 and --keep-combined
- [ ] Coverage used to rank the sweep, never to narrow it
- [ ] Contract, state, and callgraph ledgers built before any defect hunting
- [ ] Full body of every callable read, and delegated calls followed, before judging it
- [ ] Sweep passes 2 through 10 run, with pass 8 restricted to C++ and C# files
- [ ] Ownership ladder applied to every contract-versus-behavior candidate, rung 4 routed to /audit-facts
- [ ] Every finding carries a concrete trigger and a concrete result
- [ ] Every finding quotes both the contract and the implementation verbatim with locations
- [ ] Every finding assigned a category, a severity, a confidence tier, and a coverage tier
- [ ] Every false-positive guard applied in order, with the discarded-candidate count recorded
- [ ] Every external library contract verified against the installed package, not from memory
- [ ] Severity raised one step for T0 and T1 findings, and lowered for none on coverage grounds
- [ ] One root cause reported once, under its most specific category, with repeats collapsed and counted
- [ ] Citation verification run against every finding, with each quote confirmed at its cited line
- [ ] Every finding whose quote or line failed citation verification deleted rather than repaired
- [ ] Adversarial refutation run against every CRITICAL and HIGH finding, in fresh sub-agents
- [ ] Every refuted finding discarded, and the confirmed and refuted counts recorded
- [ ] Triage header present, carrying the severity by confidence counts and every discard count
- [ ] Coverage ledger present, with every skipped file listed by path and reason
- [ ] Every confidence tier reported, with LOW included unless the user narrowed the report
- [ ] LOW confidence findings placed in the trailing appendix rather than interleaved
- [ ] No style, formatting, or convention findings appear (those belong to /audit-style)
- [ ] No cost or speed findings appear (those belong to /audit-performance)
- [ ] No documentation-side findings appear (those belong to /audit-facts)
- [ ] No missing test reported as a defect, and no coverage percentage reported as a defect
- [ ] Findings ordered most severe first
- [ ] Suggested fixes are concrete code changes, each carrying an Approval verdict
- [ ] Every sentence the report itself writes, outside a verbatim quote, is under 40 words and uses only full stops and commas as clause separators
- [ ] Report prose fills each line to 120 characters, with no line ending before column 100 while its next word would
      still fit
- [ ] No file modifications made during the audit
```
