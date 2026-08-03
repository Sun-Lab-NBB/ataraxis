# Change mode

How `/audit-project` resolves a change set, decides which audits that change needs, gates the work on
the result, and loops until the new code passes.

Change mode exists so a feature, a bugfix, or a refactor is audited while it is still in context, and
it is the ONE sanctioned use of the explicit change-set scoping each audit defines.

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

The change set is the UNION of the committed and uncommitted lists, because work in progress is
exactly what this mode exists to audit.

Three narrower forms exist when the user names one:

| The user says              | Base                                      |
|----------------------------|-------------------------------------------|
| "audit my changes"         | The merge base against the default branch |
| "audit this commit"        | That commit's parent                      |
| "audit what I have staged" | `HEAD`, over `git diff --cached`          |

State the resolved base revision in the plan and in the report. A change-mode report that does not
name its base cannot be reproduced.

Drop generated files, vendored trees, and anything under a virtual environment, a build directory, or
a tox working directory from the list before routing, and record the count dropped.

---

## The routing table

Route from what the change CONTAINS rather than from file extensions alone. A docstring-only edit to a
Python file needs no numeric width trace, and running one wastes a whole audit.

| The change set contains                                                 | Audits that run                       |
|-------------------------------------------------------------------------|---------------------------------------|
| Any executable source statement, added or modified                      | facts, style, correctness             |
| Numeric work, a loop, an allocation, an I/O call, or a per-frame method | adds performance                      |
| A docstring, a comment, or a Doxygen or XML doc block                   | facts, style                          |
| A metadata documentation file (`*.md`, `*.rst`)                         | facts, style                          |
| A build or configuration file                                           | facts, style                          |
| A test file                                                             | style, plus correctness Pass 10 alone |
| Generated or vendored files alone                                       | none, and the run reports that        |

Apply the table to the WHOLE change set rather than per file. An audit runs when any file in the set
triggers it, and it then covers every file in the set that binds to it.

Two refinements matter.

**Performance is opt-out, not opt-in, for source changes that touch data.** Read the hunks rather than
guessing. A change adding a loop over an array, a `np.` constructor, a file read, a serialization
call, or a method inside a Unity per-frame set triggers it. A change renaming a variable does not.

**Correctness always runs when an executable statement changed.** There is no cheap way to know that a
statement is safe without asking, and that asking is the audit.

Record every skipped audit with the routing row that skipped it.

---

## What a narrowed audit still reads

A narrowed audit reads every file in the change set IN FULL. It never reads only the hunk.

Every audit depends on whole-file context that a hunk destroys. Correctness needs the full body of a
callable and the guards that dominate a line. Performance needs the call sites that establish
multiplicity, which usually sit in other files. Facts needs the whole implementation a docstring
documents. Style needs the whole file to judge ordering, visibility grouping, and length
proportionality.

The saving in change mode comes from auditing FEWER FILES, never from reading less of each one.

Where a changed file's caller lives outside the change set, that caller is read as AUTHORITY rather
than audited, exactly as the audits already read tests and build files.

---

## The gate

Findings split into two classes, and the split decides whether the work is done.

| Verdict      | Findings in the class                                                                |
|--------------|--------------------------------------------------------------------------------------|
| **BLOCKING** | Correctness CRITICAL and HIGH, every CONTRACT_VIOLATION, facts WRONG, style BLOCKING |
| **ADVISORY** | Everything else, meaning every MEDIUM and LOW finding and every performance one      |

Every performance finding is advisory. Cost is a judgment call the user makes with the numbers in
front of them, and a slow implementation is finished work while a wrong one is not.

Report the verdict as one of four states:

| State                 | Meaning                                                  |
|-----------------------|----------------------------------------------------------|
| PASSED                | No blocking findings survived                            |
| BLOCKED               | Blocking findings remain, and a round is available       |
| ADVISORY ONLY         | No blocking findings, and advisory findings are reported |
| CAPPED after 3 rounds | Blocking findings remain and the loop budget is spent    |

---

## The fix-and-recheck loop

A BLOCKED verdict starts a round. Each round has three parts, and the middle part happens OUTSIDE the
audits.

1. **Report.** Hand back every blocking finding with its full evidence, grouped by file so the fixes
   can be made in one pass per file.
2. **Fix, outside the audit.** The audits produce findings only and modify nothing. Fixes are ordinary
   implementation work performed between rounds, in the audits' declared fix order, which is facts,
   then correctness, then performance, then style.
3. **Re-audit.** Run the audits again over the UNION of the original change set and every file the
   fixes touched. A fix frequently edits a file the original change did not, and that file is now part
   of the work.

Re-run only the audits that produced the blocking findings, plus any audit the newly touched files
newly trigger under the routing table. An audit that passed on unchanged files does not run again.

Three rules protect the loop from defeating itself:

- Never resolve a finding by weakening a test, a contract, a docstring, a coverage setting, or a lint
  suppression. A gate satisfied that way is a gate defeated, and the finding is recorded as unresolved
  instead.
- Never resolve a facts finding by editing the code when the ownership ladder assigned the fix to the
  documentation, or the reverse.
- Never suppress a finding to reach PASSED. An accepted finding is stated as accepted, with its
  reason, and the verdict stays ADVISORY ONLY rather than becoming PASSED.

---

## Ending the loop

Three rounds cap the loop. A round is one report, one fix pass, and one re-audit.

The cap exists because a change set that still blocks after three rounds is telling you something the
next round will not fix, which is usually that the design is wrong rather than the code.

On the cap, produce the report with the CAPPED verdict, list every remaining blocking finding with the
rounds it survived, and hand the decision to the user. Do not start a fourth round, and do not lower a
severity to reach PASSED.

A PASSED or ADVISORY ONLY verdict ends the loop and is the point at which offering to commit is
appropriate. A BLOCKED or CAPPED verdict is not.
