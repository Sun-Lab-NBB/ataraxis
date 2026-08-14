# Change mode

How `/audit-project` resolves a change set, decides which audits that change needs, gates the work on the result, and
loops until the new code passes.

Change mode exists so a feature, a bugfix, or a refactor is audited while it is still in context.

---

## Contents

- Resolving the change set
- The routing table
- What a narrowed audit still reads
- The gate
- The fix-and-recheck loop
- Ending the loop

---

## Resolving the change set

Resolve the base revision first, then the file list.

```bash
git branch --show-current                                  # the working branch
git symbolic-ref --short refs/remotes/origin/HEAD          # the default branch, strip origin/
git merge-base HEAD <default-branch>                       # the base revision
git diff --name-only <base>...HEAD                         # committed work on this branch
git status --porcelain                                     # uncommitted work
```

The change set is the UNION of the committed and uncommitted lists, because work in progress is exactly what this mode
exists to audit.

Three narrower forms exist when the user names one:

| The user says              | Base                                      |
|----------------------------|-------------------------------------------|
| "audit my changes"         | The merge base against the default branch |
| "audit this commit"        | That commit's parent                      |
| "audit what I have staged" | `HEAD`, over `git diff --cached`          |

State the resolved base revision in the plan and in the report. A change-mode report that does not name its base cannot
be reproduced.

Drop generated files, vendored trees, and anything under a virtual environment, a build directory, or a tox working
directory from the list before routing, and record the count dropped.

---

## The routing table

Route from what the change CONTAINS rather than from file extensions alone. A docstring-only edit to a Python file needs
no numeric width trace, and running one wastes a whole audit.

| The change set contains                                                 | Audits that run                       |
|-------------------------------------------------------------------------|---------------------------------------|
| Any executable source statement, added or modified                      | facts, style, correctness             |
| Numeric work, a loop, an allocation, an I/O call, or a per-frame method | adds performance                      |
| A docstring, a comment, or a Doxygen or XML doc block                   | facts, style                          |
| A metadata documentation file (`*.md`, `*.rst`)                         | facts, style                          |
| A build or configuration file                                           | facts, style                          |
| A test file                                                             | style, plus correctness Pass 10 alone |
| Generated or vendored files alone                                       | none, and the run reports that        |
| A created or deleted file, whatever else the change set holds           | adds the `/audit-style` layout pass   |

Apply the table to the WHOLE change set rather than per file. An audit runs when any file in the set triggers it, and it
then covers every file in the set that binds to it.

This table decides what the wave 2 members are RECOMMENDED for. The Step 0 election decides whether they run, and a user
who declines one gets it recorded as DECLINED rather than as a routing skip.

Two refinements matter.

**Performance is recommended, not merely permitted, for source changes that touch data.** Read the hunks rather than
guessing. A change adding a loop over an array, a `np.` constructor, a file read, a serialization call, or a method
inside a Unity per-frame set triggers it. A change renaming a variable does not.

**Correctness is recommended whenever an executable statement changed.** There is no cheap way to know that a statement
is safe without asking, and that asking is the audit. This is the case where the plan should argue for the election
rather than present it neutrally.

**The layout pass follows the shape of the file set rather than its contents.** `/audit-style` sweeps the project
directory tree once per run, on its own main agent, and only where the target is a project root. In change mode it
additionally requires that the change set CREATED or DELETED a file, because a layout finding follows from where a file
sits rather than from what it holds. Tell `/audit-style` both facts, and record the status it returns, which is `run`,
`skipped-not-a-project-root`, or `skipped-no-created-or-deleted-files`. A change set that edits files in place skips the
pass, and the report states that rather than reporting a tree nothing examined as clean.

Record every audit that did not run with the routing row or the election that stopped it.

---

## What a narrowed audit still reads

A narrowed audit reads every file in the change set IN FULL. It never reads only the hunk.

Every audit depends on whole-file context that a hunk destroys. Correctness needs the full body of a callable and the
guards that dominate a line. Performance needs the call sites that establish multiplicity, which usually sit in other
files. Facts needs the whole implementation a docstring documents. Style needs the whole file to judge ordering,
visibility grouping, and length proportionality.

The saving in change mode comes from auditing FEWER FILES, never from reading less of each one.

Where a changed file's caller lives outside the change set, that caller is read as AUTHORITY rather than audited,
exactly as the audits already read tests and build files.

---

## The gate

Findings split into two classes, and the split decides whether the work is done.

| Verdict      | Findings in the class                                                                |
|--------------|--------------------------------------------------------------------------------------|
| **BLOCKING** | Correctness CRITICAL and HIGH, every CONTRACT_VIOLATION, facts WRONG, style BLOCKING |
| **ADVISORY** | Everything else: correctness MEDIUM and LOW, style STANDARD, INCONSISTENCY, and    |
|              | CONFLICT, every facts verdict, and every performance finding                      |

Every performance finding is advisory. Cost is a judgment call the user makes with the numbers in front of them, and a
slow implementation is finished work while a wrong one is not.

A gate covers the audits that RAN. Where the election declined `/audit-correctness`, the largest blocking class was
never computed, so the verdict states that it covers wave 1 alone and names the declined audit beside it. A PASSED
verdict that hides a declined correctness audit reads as an assurance nobody produced.

Report the verdict as one of four states:

| State                 | Meaning                                                  |
|-----------------------|----------------------------------------------------------|
| PASSED                | No blocking findings survived                            |
| BLOCKED               | Blocking findings remain, and a round is available       |
| ADVISORY ONLY         | No blocking findings, and advisory findings are reported |
| CAPPED after 3 rounds | Blocking findings remain and the loop budget is spent    |

---

## The fix-and-recheck loop

A BLOCKED verdict starts a round. Each round has three parts, and the middle part happens OUTSIDE the audits.

1. **Report.** Hand back every blocking finding with its full evidence, grouped by file so the fixes can be made in one
   pass per file.
2. **Fix, outside the audit.** The audits produce findings only and modify nothing. Fixes are ordinary implementation
   work performed between rounds, in the audits' declared fix order, which is facts, then correctness, then performance,
   then style. A fix whose Impact names a break STOPS here for the user, under the section below.
3. **Re-audit.** Run the audits again over the UNION of the original change set and every file the fixes touched. A fix
   frequently edits a file the original change did not, and that file is now part of the work.

Re-run only the audits that produced the blocking findings, plus any audit the newly touched files newly trigger under
the routing table. An audit that passed on unchanged files does not run again.

### Fixes that require explicit user approval

A fix that breaks the public API or alters public behavior is PRESENTED and waited on rather than applied. The agent
never makes that call alone, in any mode, and no gate verdict authorizes it.

| The fix would                                                        | Example                          |
|----------------------------------------------------------------------|----------------------------------|
| Remove, rename, or re-type a public symbol                           | `read_frame` becomes `read`      |
| Change a public signature, its defaults, or its return type          | Adding a required parameter      |
| Change what a public callable returns or raises for accepted input   | Raising where it returned `None` |
| Change a wire format, on-disk schema, packed layout, or CLI contract | Widening a packed field          |
| Change a documented contract that callers rely on                    | Narrowing an accepted range      |

Public carries the ataraxis visibility meaning, which is a symbol with no leading underscore, or one its package
re-exports from `__init__`. A symbol private to its own module is not in this class, and neither is a test.

Present such a fix with what breaks, the callers the shared context's CALLGRAPH ledger shows reaching it, and the
alternative that preserves the contract where one exists. Then WAIT.

Approval is per fix rather than per round or per run. A user who approved one break has not approved the next, and a
blocking finding whose only fix needs approval stays unresolved until they answer, which holds the verdict at BLOCKED
rather than advancing it.

Three rules protect the loop from defeating itself:

- Never resolve a finding by weakening a test, a contract, a docstring, a coverage setting, or a lint suppression. A
  gate satisfied that way is a gate defeated, and the finding is recorded as unresolved instead.
- Never resolve a facts finding by editing the code when the ownership ladder assigned the fix to the documentation, or
  the reverse.
- Never suppress a finding to reach PASSED. An accepted finding is stated as accepted, with its reason, and the verdict
  stays ADVISORY ONLY rather than becoming PASSED.

---

## Ending the loop

Three rounds cap the loop. A round is one report, one fix pass, and one re-audit.

The cap exists because a change set that still blocks after three rounds is telling you something the next round will
not fix, which is usually that the design is wrong rather than the code.

On the cap, produce the report with the CAPPED verdict, list every remaining blocking finding with the rounds it
survived, and hand the decision to the user. Do not start a fourth round, and do not lower a severity to reach PASSED.

A PASSED or ADVISORY ONLY verdict ends the loop and is the point at which offering to commit is appropriate. A BLOCKED
or CAPPED verdict is not.
