# Detection passes

The ordered sweep passes of `/audit-style`. Each pass asks one question of every line in scope, and
each names the mechanical procedure that answers it. Run them in order, because the passes that
resolve a construct's identity gate the passes that judge it.

The passes decompose the workflow steps in the skill. Pass 1 executes Step 2, pass 10 executes Step 4,
passes 2 through 6 cover Dimension A of Step 5, passes 7 and 8 cover Dimension B, pass 9 covers
Dimension C, and pass 11 covers Dimension D.

Every pass draws its authority from the checklists Pass 1 loads. A convention absent from every loaded
checklist is not a violation, whatever a pass below appears to invite.

---

## Contents

- Pass 1: Rule ledger
- Pass 2: File shape sweep
- Pass 3: Naming sweep
- Pass 4: Signature and annotation sweep
- Pass 5: Import and dependency sweep
- Pass 6: Idiom and error-handling sweep
- Pass 7: Documentation form sweep
- Pass 8: Comment and suppression sweep
- Pass 9: Cross-file consistency sweep
- Pass 10: Project layout sweep
- Pass 11: Symbol usage sweep

---

## One traversal, eleven questions

Passes 2 through 8 are a CHECKLIST OF QUESTIONS rather than a schedule of re-reads. Read each file
ONCE and answer every applicable pass during that single traversal, carrying the pass list beside you.
Re-reading the file set once per pass costs six extra traversals of every line in scope and surfaces
nothing the single traversal misses.

Four passes sit outside that traversal. Pass 1 runs first, because it builds the ledger every later
pass reports against. Pass 10 runs once on the main agent at Step 4, before the traversal opens,
because the directory tree belongs to the repository rather than to any file in it. Passes 9 and 11 run
last on the main agent, because each needs the whole file set in one view.

Pass 11 also takes an input the traversal produces. Every file read during passes 2 through 8 records
the symbols it DECLARES and the symbols it REFERENCES. The single traversal therefore supplies the two
tables pass 11 reconciles rather than paying for a second reading of every file in scope.

---

## What the sweep keeps

Every pass defers to the Step 3 deterministic gates. Where a configured tool decides a rule, the
tool's output IS the finding and the pass reports nothing on its own authority. Guard 2 of
`false-positive-guards.md` lists what the tools own, which is line length, indentation, blank-line
counts, quote and string form, trailing commas, import sorting and grouping, and every rule carried by
a ruff code the project enables.

The sweep keeps the rules no tool decides:

- Documentation quality in every form, which is length proportionality, redundancy with the signature,
  behavioral scope, sentence length, mood, separator punctuation, positive description, and spelling.
- Identifier vocabulary, meaning full words against the abbreviations the checklist enumerates.
- Element and section ORDERING where the checklist states an order the formatter does not enforce.
- Visibility placement.
- Cross-file consistency, and every cross-skill conflict.
- Symbol visibility against actual usage, together with every asset no consumer references. Ruff
  reaches that only for imports outside `__init__.py`, for locals, and for arguments, and carries no
  rule for an unused module-level definition or an unused member.

---

## Batching the fan-out

A Large-tier audit BATCHES BY AUTHORITY, so no sub-agent carries more than one or two checklists. This
is the rule that decides what this audit costs. Every sub-agent re-receives the checklists its files
bind to, and the loaded checklists of a mixed project root run to tens of thousands of tokens, so a
sub-agent holding every authority pays for guides it never applies, once per sub-agent.

Build the batches under three rules:

1. **One authority per batch, two at the absolute most.** Group files by the authority the Step 1
   binding table assigned. A batch is Python files, or C++ files, or C# files, and never a mixture.
2. **A single-file authority gets its own sub-agent.** `README.md`, `pyproject.toml`, and `tox.ini`
   each bind to a checklist nothing else uses. `platformio.ini` travels with `library.json`, and
   `CLAUDE.md` with `AGENTS.md`. A skill is one such batch rather than one per file, so its
   `SKILL.md` and its `references/*.md` travel together, because the progressive-disclosure rules
   judge a reference file against the `SKILL.md` that loads it. The documentation package under
   `docs/` is one sub-agent holding `/api-docs` alone.
3. **Roughly eight files per source batch**, sharing a package or a directory. Forty sub-agents cap
   the run, twelve run at once, and batches beyond forty merge by shared checklist rather than
   dropping files.

Each sub-agent loads ONLY the checklists its own batch binds to, and receives ONLY the rule-ledger
rows Step 2 built from those checklists.

Only the per-file sweep fans out. Every other step runs on the main agent. The rule ledger, the
cross-file consistency pass, the symbol usage pass, the severity rating, the guards, the verification,
and the report each need the whole file set in one view or sit on a trust boundary.

The fan-out therefore carries a SECOND return value. Every batch sub-agent returns its findings AND the
declaration and reference rows Pass 11 defines, covering each symbol its files declare and each symbol
its files reference. Findings alone would leave the main agent unable to run Pass 11 at all, because a
symbol declared in one batch and consumed in another is invisible to both sub-agents while the main
agent never reads their files.

Do NOT use the `Explore` agent type for sweep work. Explore returns summaries rather than
verbatim citations and breaks the "verbatim checklist quote" discipline.

---

## Pass 1: Rule ledger

**Question:** Which checklist items actually govern this file?

For every distinct style skill the Step 1 bindings resolve to, load its full verification checklist
together with every reference file the skill names. Then build the ledger the later passes consume:

```text
| rule id | verbatim checklist text | source skill and file | applies to |
```

Assign each rule a short identifier, copy its text VERBATIM, and record which pass below will check
it. A rule copied loosely produces a finding that cannot be defended, so use the Read tool and copy
rather than paraphrasing.

Record separately any rule that two loaded checklists state incompatibly. Those become CONFLICT
findings and are surfaced rather than silently resolved in favor of one skill.

The ledger converts the audit from a search for anything that looks wrong into a finite checklist
walked against a finite file set. Every later pass reports only rule identifiers drawn from it.

---

## Pass 2: File shape sweep

**Question:** Is this construct in the position its checklist requires?

Walk the file's skeleton before its contents, because an ordering violation frequently explains
several downstream observations.

Check, in this order:

1. The file-level section ordering against the checklist's declared order, which covers the module
   docstring, the imports, the constants, the type definitions, and the definitions themselves.
2. Visibility ordering, so public definitions precede private ones at module level and inside every
   class body, with dunder methods holding their conventional position.
3. Type definitions the checklist requires above the code consuming them.
4. Call-hierarchy or by-purpose grouping within each visibility group.
5. Blank-line counts, line length, and indentation width, ONLY where no Step 3 gate covered the file.
   A formatter that ran already decided all three exactly, so its diff is the finding and this pass
   reports nothing on them.

Record the position each construct occupies and the position the checklist requires. A finding names
both, because a reordering suggestion without the target position is not actionable.

Ordering and visibility placement stay with this pass in every case, because no formatter in the
toolchain enforces the checklist's declared section order or its public-before-private rule.

---

## Pass 3: Naming sweep

**Question:** Does this identifier follow the casing and vocabulary its checklist requires?

Extract every identifier the file declares, which covers modules, classes, functions, methods,
parameters, locals, constants, enum members, and fields. Then check each against three separate rules,
because an identifier can satisfy one and break another.

First, casing against the checklist's per-kind convention, which differs by language and by kind.
Second, the visibility prefix, so a member private to its module or class carries the marker the
checklist requires and a member crossing a module boundary does not. Third, vocabulary, so identifiers
use full words rather than the abbreviations the checklist enumerates.

Resolve the kind before judging the casing. A constant and a variable carry different conventions in
every one of these languages, so an identifier judged as the wrong kind produces a false finding.

---

## Pass 4: Signature and annotation sweep

**Question:** Does this declaration carry the annotations and argument conventions its checklist
requires?

For every callable, check that each parameter and the return carry a type annotation where the
checklist requires one, and that the annotation is parameterized rather than bare where the checklist
names a parameterized form.

Then check the call conventions the checklist states, which covers keyword arguments at call sites and
their documented exceptions, boolean flags placed behind a keyword-only separator, and the declaration
forms the checklist prescribes for aliases, dataclasses, and enums.

Resolve each documented exception before reporting. A checklist that exempts one call form exempts it
wherever that form appears, so confirm the construct is outside the exemption rather than assuming it.

---

## Pass 5: Import and dependency sweep

**Question:** Is this import in the position and the form its checklist requires?

Collect every import in the file with its line, then check four properties. Position, so every import
sits at the top of the file and no deferred or function-local import appears. Form, so a local import
brings in the required names directly rather than the module holding them. Boundary, so an import
reaching another package goes through that package's public namespace rather than into a submodule.
Export surface, so an `__init__` declares `__all__` and orders its entries as the checklist requires.

This pass sees ONE file, so it decides only what that one file reveals. Whether an export list holds
the right NAMES is a question about the whole file set, because the answer turns on which packages
import which symbols, and pass 11 owns it. A deep import reaching past a missing export is one
construct together with that missing export, so Guard 13 of `false-positive-guards.md` reports the
pair once, under pass 11, with the import site carried as a citation.

Leave import sorting and grouping alone where the checklist delegates it to a formatter, and report
only the properties the checklist itself states.

---

## Pass 6: Idiom and error-handling sweep

**Question:** Does this statement use the construct its checklist prescribes for the job?

Sweep the file for the constructs the loaded checklist names, and for each one check whether the
prescribed form is the one in use. The recurring subjects are error reporting and its message format,
comparison forms for booleans and for null, guard clauses against nested conditionals, and resource
management through a scoped construct. They also cover path handling, string interpolation and
quoting, and the library helper the checklist prefers over a hand-rolled equivalent.

Judge each against the checklist text rather than against general good practice, and report only where
the checklist names the prescribed form. This pass is where invented conventions enter a report most
easily, because the constructs are familiar and the temptation to apply outside knowledge is strongest.

---

## Pass 7: Documentation form sweep

**Question:** Does this documentation block take the shape its checklist requires?

This pass judges the FORM of the prose. A claim inside it that disagrees with the code is a factual
finding and belongs to `/audit-facts`.

Walk every documentation block and check:

1. Presence on every member the checklist requires one for.
2. Section ordering within the block, and the absence of sections the checklist excludes.
3. Mood and person against the checklist's stated voice.
4. Prose form against the checklist's structural rules, which covers prose against bullet lists and
   the specifier forms permitted in each context.
5. Sentence length against the checklist's stated word limit, which is commonly 40 words.
6. Length proportionality, so the block's size tracks the difficulty of understanding the code rather
   than the length of the code.
7. Redundancy, so the block avoids restating the type signature and avoids padding the reader can
   infer from the code directly.
8. Behavioral scope, so the block describes the asset's own behavior rather than the pipeline stage or
   feature that consumes it.
9. Separator punctuation against the checklist's rule, so clauses are separated by full stops and
   commas rather than by a semicolon or an em-dash.
10. Positive description, so the text states present behavior rather than framing it by what it is not
    or what it used to be. Contrastive and historical framing ("does X, not Y" or "used to do Y") is a
    finding, with a load-bearing contrast that carries its reason exempt.
11. Spelling and grammar against the checklist's stated language variant, checked word by word rather
    than by impression.

Check each block against all eleven rather than stopping at the first, because these violations
co-occur and a block corrected for one frequently still breaks three others.

---

## Pass 8: Comment and suppression sweep

**Question:** Does this comment earn its place under the checklist's rules?

Walk every comment that sits outside a documentation block, and classify each one. A comment
explaining why a non-obvious decision was made is compliant. A comment narrating what the next
statement plainly does is a finding. A comment restating a type the annotation already carries is a
finding. A heavy separator block is a finding where the checklist prohibits it.

Then sweep the suppression comments separately, because they carry their own rules. Report an
editor-specific suppression where the checklist prohibits it, and leave the linter and type-checker
suppressions the checklist declares authoritative in place. Whether a suppression is still needed is a
factual question about a diagnostic and belongs to `/audit-facts`.

Apply the same mood, sentence-length, separator, and positive-description rules from Pass 7 to comment
prose, because the checklist states them for all documentation rather than for blocks alone.

---

## Pass 9: Cross-file consistency sweep

**Question:** Does this file follow the same convention its siblings follow?

This pass runs on the main agent after every per-file pass completes, because it needs the whole file
set in one view.

Build a convention table across the file set with one row per recurring decision and one column per
file, then read each row for disagreement. The rows worth building start with the name given to the
same concept in sibling classes, the ordering scheme within comparable files, and the declaration form
chosen for comparable types. The remaining rows are the error-reporting form, the documentation
section set on comparable members, and the import style for comparable dependencies.

Report a row where files disagree and the loaded checklist permits both forms only separately, which
is an INCONSISTENCY rather than a violation of any single rule. Where the checklist prescribes one
form outright, the deviating file is an ordinary finding under the pass that owns that rule, so report
it there instead and keep this pass for genuine drift.

---

## Pass 10: Project layout sweep

**Question:** Does the repository hold the tree its archetype requires, and nothing that tree forbids?

This pass runs once on the main agent at Step 4, before the per-file traversal opens. It is the only
pass that can report a path as ABSENT, because every other pass takes one existing file as its input
and no such pass can report the file nobody wrote.

Run it for a project-root target alone. A package directory and a single file carry no tree to judge,
and in change mode only a change set that creates or deletes files can alter one. Record the skip and
its reason rather than passing over it silently.

Work in four parts:

1. Resolve the archetype from the key-indicator table in `/project-layout`, recording the indicators
   that decided it and how confidently they matched. A repository that matches no archetype, such as an
   umbrella repository indexing siblings, carries no tree to diff, so record the skip with that reason
   and report no absent-path findings against it.
2. Read the archetype's section of `/project-layout`'s `archetype-trees.md`, which is the authority for
   the required and the forbidden paths.
3. Walk the required paths and report each one the repository lacks, then walk the repository and
   report each path the tree does not sanction.
4. Apply the presence and absence items of `/project-layout`'s checklist, which cover the `envs/`
   contents, the `.github/ISSUE_TEMPLATE/` contents, and the pairing of `.netlify-site` with the
   `deploy` tox environment.

Report each finding in the shape below, which replaces the skill's ordinary finding shape because a
`Location: <path>:<line>` citation cannot point at a file that does not exist:

```text
[Severity]: <BLOCKING | STANDARD | INCONSISTENCY | CONFLICT>
[Confidence]: <HIGH | MEDIUM | LOW>
Skill: /project-layout
Checklist point: "<verbatim archetype tree line, or verbatim layout checklist item>"
Expected path: <repository-relative path the archetype tree requires, for an absent path>
Offending path: <repository-relative path the tree does not sanction, for a stray path>
Current state: <ABSENT | PRESENT>
Required state: <PRESENT, or ABSENT for a stray or forbidden path>
Suggested fix: <concrete creation, removal, or relocation>
Approval: <REQUIRED when the fix deletes or relocates a tracked path, naming what breaks>
```

An absent-path finding carries `Expected path` alone and quotes the archetype tree line that requires
the path. A stray-path finding carries `Offending path` alone and quotes the checklist item or the tree
section that excludes it.

---

## Pass 11: Symbol usage sweep

**Question:** Does each symbol's declared visibility match the widest boundary its consumers actually
cross, and does each symbol have a consumer at all?

This pass runs on the main agent after every per-file pass completes, alongside pass 9, because a
symbol's tier is a property of the WHOLE file set rather than of the file that declares it. No per-file
pass can reach these findings. A pass reading one file sees the declaration or the reference, never
both, so it can no more report an export nobody imports than pass 10 could report a file nobody wrote.

Work in four parts:

**1. Build the declaration table.** One row per symbol the file set declares. Each row carries the
declaring path and line, the kind, the declared visibility, the defining module, and the defining
package, together with whether that package's `__init__.py` lists the symbol in its import block and
in `__all__`.

**2. Build the reference table.** One row per site that references a declared symbol, carrying the
referencing path and line, the referencing module, the referencing package, and whether the site sits
under `tests/`. Passes 2 through 8 collect both tables during their single traversal, and in a
Large-tier run each batch sub-agent returns its share alongside its findings.

**3. Resolve the actual tier of every symbol** by reading its reference rows: `none`, `module-local`,
`package-local`, or `cross-package`. Discount every row the guards exclude before resolving, which is
the work Guard 15 of `false-positive-guards.md` defines.

**4. Compare the actual tier against the declared one.** Every mismatch is one of five findings:

| Declared                              | Actual tier   | Finding            | Fix                                   |
|---------------------------------------|---------------|--------------------|---------------------------------------|
| Underscore prefix                     | package-local | UNDER_EXPOSED      | Rename the symbol public              |
| Public name                           | module-local  | OVER_EXPOSED       | Add the underscore prefix             |
| Absent from the package `__init__.py` | cross-package | MISSING_EXPORT     | Add to the import block and `__all__` |
| Listed in the package `__init__.py`   | package-local | UNWARRANTED_EXPORT | Remove both entries                   |
| Any                                   | none          | UNUSED_ASSET       | Remove the asset                      |

Report each finding in the skill's ordinary shape, carrying one added field directly below
`Current state`:

```text
Consumer set: <every referencing path and line, or NONE, with the search that established it>
```

`Location` and `Current state` cite the DECLARATION site and quote its line verbatim, because that is
what the fix edits. MISSING_EXPORT and UNWARRANTED_EXPORT instead cite the `__init__.py` and quote its
`__all__` block. Keeping `Current state` verbatim is what lets Check 1 verify these findings unchanged,
and `Consumer set` carries the one claim no reading of a single file can settle.

Every one of the five is a claim about ABSENCE, which is the shape of claim a partial reading gets
wrong most often, so `Consumer set` names the search that established it rather than asserting it.
Confirm each candidate with a repository-wide search for the symbol's name across `src/`, `tests/`, and
the configuration files carrying runtime registrations, then quote what the search returned. Where the
repository holds a `.codegraph/` index, `codegraph explore` answers the same question directly and is
the cheaper confirmation. A candidate whose search never ran, and a finding that cannot name its
consumer set, are both deleted rather than reported at low confidence.

Rate UNDER_EXPOSED, MISSING_EXPORT, and a deep import reaching past a missing export as BLOCKING,
because each names a rule the checklists state with MUST. Rate OVER_EXPOSED, UNWARRANTED_EXPORT, and
UNUSED_ASSET as STANDARD.

Run this pass for every language in scope, resolving the tier against the visibility construct the
bound checklist names. That construct is the underscore prefix and the `__init__.py` export list in
Python, the underscore prefix and the public header in C++, and the access modifier in C#.
