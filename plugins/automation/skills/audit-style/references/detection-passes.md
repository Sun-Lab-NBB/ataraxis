# Detection passes

The ordered sweep passes of `/audit-style`. Each pass asks one question of every line in scope, and
each names the mechanical procedure that answers it. Run them in order, because the passes that
resolve a construct's identity gate the passes that judge it.

The passes decompose the workflow steps in the skill. Pass 1 executes Step 2, passes 2 through 6
cover Dimension A of Step 3, passes 7 and 8 cover Dimension B, and pass 9 covers Dimension C.

Every pass draws its authority from the checklists Pass 1 loads. A convention absent from every loaded
checklist is not a violation, whatever a pass below appears to invite.

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
5. Blank-line counts between definitions and around import blocks.
6. Line length against the checklist's limit, and indentation width.

Record the position each construct occupies and the position the checklist requires. A finding names
both, because a reordering suggestion without the target position is not actionable.

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
Export surface, so an `__init__` declares the names the checklist requires and keeps them ordered.

Leave import sorting and grouping alone where the checklist delegates it to a formatter, and report
only the properties the checklist itself states.

---

## Pass 6: Idiom and error-handling sweep

**Question:** Does this statement use the construct its checklist prescribes for the job?

Sweep the file for the constructs the loaded checklist names, and for each one check whether the
prescribed form is the one in use. The recurring subjects are error reporting and its message format,
comparison forms for booleans and for null, guard clauses against nested conditionals, resource
management through a scoped construct, path handling, string interpolation and quoting, and the
library helper the checklist prefers over a hand-rolled equivalent.

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
5. Sentence length against the checklist's stated word limit.
6. Length proportionality, so the block's size tracks the difficulty of understanding the code rather
   than the length of the code.
7. Redundancy, so the block avoids restating the type signature and avoids padding the reader can
   infer from the code directly.
8. Behavioral scope, so the block describes the asset's own behavior rather than the pipeline stage or
   feature that consumes it.
9. Separator punctuation against the checklist's rule.
10. Positive description, so the text states present behavior rather than framing it by what it is not
    or what it formerly did.
11. Spelling and grammar.

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
file, then read each row for disagreement. The rows worth building are the name given to the same
concept in sibling classes, the ordering scheme within comparable files, the declaration form chosen
for comparable types, the error-reporting form, the documentation section set on comparable members,
and the import style for comparable dependencies.

Report a row where files disagree and the loaded checklist permits both forms only separately, which
is an INCONSISTENCY rather than a violation of any single rule. Where the checklist prescribes one
form outright, the deviating file is an ordinary finding under the pass that owns that rule, so report
it there instead and keep this pass for genuine drift.
