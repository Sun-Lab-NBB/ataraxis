# Detection passes

The ordered sweep passes of `/audit-facts`. Each pass asks one question of every claim in scope, and
each names the mechanical procedure that answers it. Run them in order, because the cheapest and most
mechanical passes gate the ones that require reading a full implementation.

The passes decompose the workflow steps in the skill. Pass 1 executes Step 2, passes 2 through 6
execute Step 3, and passes 7, 8, and 9 execute Steps 4, 5, and 6.

Every pass applies to both documentation classes. A metadata file's claims resolve against the
packages and tools it names, and an in-source block's claims resolve against the implementation it
sits on together with the symbols that implementation calls.

## Contents

- Pass 1: Claim harvest
- Pass 2: Existence sweep
- Pass 3: Signature sweep
- Pass 4: Behavior sweep
- Pass 5: Failure sweep
- Pass 6: Quantity sweep
- Pass 7: Omission sweep
- Pass 8: Reference sweep
- Pass 9: Contradiction sweep

## One traversal, nine questions

Passes 2 through 6 are a CHECKLIST OF QUESTIONS asked of the CLAIM LEDGER rather than a schedule of
re-reads. Each claim carries the kind that names its pass, so walk the ledger once and answer each
claim under its own pass, opening each authoritative source once and settling every claim that
resolves against it while it is open. Re-reading the file set once per pass costs four extra
traversals and surfaces nothing the single walk misses.

Pass 1 runs to completion first, because it builds the ledger. Passes 7, 8, and 9 run after the ledger
is fully verified, because each needs the whole ledger in one view.

---

## Pass 1: Claim harvest

**Question:** Is this sentence asserting something the source can confirm or deny?

Walk each file in scope top to bottom and extract every verifiable claim into a ledger with five
columns:

```text
| line | claim text | kind | authoritative source | verdict |
```

Tag every claim with its kind, because the kind decides which later pass verifies it:

| Kind      | What it asserts                                                      | Verified by |
|-----------|----------------------------------------------------------------------|-------------|
| EXISTENCE | A symbol, file, path, command, environment, or version exists        | Pass 2      |
| SIGNATURE | A parameter, its default, or a return value and its type             | Pass 3      |
| BEHAVIOR  | What a callable, module, workflow, or command does                   | Pass 4      |
| FAILURE   | An exception raised, and the condition documented as triggering it   | Pass 5      |
| QUANTITY  | A unit, range, shape, dtype, count, ordering, or configuration value | Pass 6      |
| POINTER   | A cross-reference to another document, symbol, or section            | Pass 8      |

Leave out subjective quality language, aspirational and future-tense statements, and motivational
prose. Pedagogical prose enters the ledger only where it contains a verifiable factual statement.

The ledger is the unit of work for every later pass. A claim that never reaches it is never verified,
so err toward harvesting a borderline sentence and discarding it later.

---

## Pass 2: Existence sweep

**Question:** Does the thing this claim names still exist under that name?

This is the cheapest pass and the highest yield, because renames and deletions are the most common
source of documentation drift. Run it before any pass that reads an implementation.

Take every EXISTENCE claim and resolve the name it states:

| Named thing                  | Resolution                                                       |
|------------------------------|------------------------------------------------------------------|
| A symbol                     | `codegraph explore` where indexed, otherwise grep the definition |
| A file or directory path     | Test the path directly against the repository                    |
| A CLI command or subcommand  | The entry point declaration and the command registration         |
| A tox environment            | The `envlist` and the environment definition                     |
| A dependency or version pin  | The dependency declaration in the project metadata               |
| An environment or config key | The code that reads the key                                      |

Record the verdict per claim. A name that resolves nowhere is DRIFT when the documentation once
matched a symbol that has since been renamed, and WRONG when no such symbol ever existed. Where the
symbol resolves through a re-export rather than a definition, the claim is EXACT, because a
re-exported name genuinely lives in the module the documentation names.

---

## Pass 3: Signature sweep

**Question:** Does the documented interface match the declared one, parameter by parameter?

Take every SIGNATURE claim, open the declaration, and compare mechanically rather than by reading for
sense. Build a two-column diff per callable, the documented parameters against the declared ones, and
walk it to the end.

Check each of the following in order, because an early mismatch often explains a later one:

1. Every declared parameter appears in the documentation, and every documented parameter is declared.
2. Parameter names match exactly, including a leading underscore and a trailing type suffix.
3. Every documented default matches the declared default. A documented default on a parameter that
   has none, and a declared default the documentation omits, are both findings.
4. A parameter the declaration makes keyword-only is documented as such where the convention requires
   it.
5. The documented return matches the return annotation, and matches what every `return` statement
   actually produces.

For C++ read the signature together with its `const` and reference qualifiers, and for C# read the
declared type together with its nullability annotation. A documented parameter that omits a qualifier
carrying observable meaning is a finding on the same footing as a wrong name.

---

## Pass 4: Behavior sweep

**Question:** Does the implementation do what this claim says it does?

Take every BEHAVIOR claim. Read the FULL body of the documented callable, module, or workflow before
judging it, then trace the claim to the statements that satisfy it.

Follow the call before flagging. Behavior a summary describes is frequently delegated to a helper, and
delegated behavior still belongs to the documented callable. A claim is a finding only after the
delegation chain is walked and no statement anywhere in it satisfies the claim.

For a workflow or command claim, the authority is the ordered steps the code actually executes, so
compare the documented ordering against the execution order rather than against a list of the steps.

Record the verdict as SEMANTIC where the implementation satisfies the claim through different wording,
which is a match and stays out of the report. Reserve DRIFT and WRONG for a claim no statement
satisfies.

---

## Pass 5: Failure sweep

**Question:** Does this block document the failures the implementation actually produces?

Take every FAILURE claim and build two lists per callable. The first holds every exception the
documentation names together with the condition it attributes to each. The second holds every
exception the body raises, including the ones raised by the helpers the body calls where the
documentation attributes their failures to this callable.

Diff the two lists in both directions. A documented exception the body cannot raise is a finding. An
exception the body raises that a partially populated failure section omits is an omission finding, and
it belongs to Pass 7 rather than here.

Then verify each documented triggering condition against the guard that raises it, substituting one
concrete value that satisfies the documented condition and confirming it reaches the raise. A
documented condition that the guard states differently, such as an inclusive bound documented as
exclusive, is a finding even where the exception type is right.

---

## Pass 6: Quantity sweep

**Question:** Is this stated value the value the source actually carries?

Take every QUANTITY claim and resolve it against the declaration that fixes it. This pass catches the
mismatches a reader glides over, so check the value character by character rather than by impression.

| Quantity              | Authority                                                              |
|-----------------------|------------------------------------------------------------------------|
| A unit                | The variable's name suffix, its conversion site, and its consumer      |
| A range or bound      | The validation guard that enforces it                                  |
| A shape or dtype      | The array construction and the annotation                              |
| A default             | The declared default in the signature or the configuration schema      |
| A magic constant      | The constant's definition, and every other site declaring the same one |
| A count or ordering   | The collection the claim counts, enumerated                            |
| A version pin         | The dependency declaration                                             |
| A thread-safety claim | The synchronization primitive the code actually uses                   |

Where a claim states a derived quantity, such as a duration computed from a rate, recompute it and
compare. A documented figure that was correct before a constant changed is DRIFT rather than WRONG.

---

## Pass 7: Omission sweep

**Question:** Does this partially populated section cover everything it undertook to cover?

Bound this pass by the per-file source scope from Step 1, so no out-of-scope omission is claimed.

For each documentation section that already covers part of a surface, enumerate the full surface from
the source and diff it against the section. The surfaces worth enumerating are the parameters of a
documented callable, the exceptions its body raises, the attributes its class assigns, the public
functions of a module the file walks through, the steps of a workflow it describes, and the members of
a registry it lists.

A section that covers three of five members is a finding naming the two it misses. A section that is
absent entirely, and a callable, class, module, or file carrying no documentation at all, are style
concerns and belong to `/audit-style`.

---

## Pass 8: Reference sweep

**Question:** Does the thing this pointer names exist, and does it contain what the pointer claims?

Take every POINTER claim and resolve it in two stages, because a pointer fails in two distinct ways.
First confirm the target exists, which covers a document path, a section heading, a symbol, an issue,
and a version. Then open the target and confirm it contains what the pointer says it contains, because
a reference that resolves to a document no longer covering the subject is as wrong as a broken one.

Apply the same check to symbol references inside in-source documentation, which covers Sphinx
cross-reference specifiers in Python docstrings, `@see` and `@ref` tags in Doxygen blocks, and
`<see cref="...">` elements in XML doc comments.

Report only a broken or wrong reference. A working reference produces nothing.

---

## Pass 9: Contradiction sweep

**Question:** Does this claim disagree with another claim about the same thing?

Group the ledger by subject rather than by file location, so claims about one symbol sit together
however far apart they were written. Then compare every pair within a group for compatibility.

The productive groupings are a value stated in two places, a behavior described in both a module
docstring and the docstring of the member implementing it, a default stated in prose and in a table, a
parameter described differently in a class docstring and in its property, and an ordering given twice.

Report a contradiction whenever two claims cannot both be true, and name both sides with their
locations. Where the source settles which side is right, say so, and where it does not, report the
contradiction itself as the finding.
