---
name: audit-project
description: >-
  Orchestrates the four ataraxis audits in the order and concurrency their dependencies require, and
  merges their findings into one deduplicated report. Runs in full mode over a repository, package, or
  directory, or in change mode over the files a feature, bugfix, or refactor touched, where it gates
  the work until the new code passes. Use after completing any implementation task, before committing,
  when auditing a project end to end, or when the user invokes /audit-project.
user-invocable: true
---

# Project audit orchestration

Runs `/audit-facts`, `/audit-correctness`, `/audit-performance`, and `/audit-style` as one pass,
resolves the ownership collisions between them, and produces a single merged report.

You MUST read this entire skill, and load each reference file at the step that names it, before acting
on that step. The verification checklist at the end is mandatory before presenting the report.

---

## Scope

**Covers:**
- Selecting which of the four audits a target actually needs, and sequencing them
- Running the shared discovery ONCE, so four audits stop re-deriving one inventory, one set of
  prerequisites, one coverage ranking, and one callgraph
- Full mode over a repository, a package, or a directory
- Change mode over the files a feature, bugfix, or refactor touched, together with the fix-and-recheck
  loop that gates the work until the new code passes
- Merging, deduplicating, and adjudicating findings across the four audits into one report

**Does not cover:**
- The detection work itself, which each audit owns and this skill never duplicates
- Code, documentation, or configuration modifications. The audits produce findings only, and the
  change-mode fixes happen OUTSIDE them, between rounds
- Committing, pushing, or opening a pull request (see `/commit` and `/pr`)
- Codebase exploration (see `/explore-codebase`)

---

## The two modes

| Mode       | Target                                 | Trigger                                                |
|------------|----------------------------------------|--------------------------------------------------------|
| **Full**   | A repository, package, or directory    | The user asks for a project audit                      |
| **Change** | The files a unit of work touched       | Implementation work finished, or the user names a diff |

Change mode is the ONE sanctioned use of the explicit change-set scoping each audit defines. Every
audit keeps whole-target coverage as its own default, and only this skill's change mode narrows it.

Full mode never resolves a diff. A user who asks to audit the repository gets the repository.

---

## Sequence and concurrency

Four audits share one file set, so the question is which of them may read the same files at the same
time and which must wait for another's verdict.

**Real dependencies.** Only these three exist, and everything else is independent.

| Consumer             | Depends on          | What flows                                                  |
|----------------------|---------------------|-------------------------------------------------------------|
| Every audit          | Shared context      | File inventory, prerequisites, coverage ranking, callgraph  |
| `/audit-correctness` | `/audit-facts`      | The DRIFT and WRONG verdicts its ownership ladder routes on |
| `/audit-performance` | Shared context      | The callgraph its hot-path census turns into multiplicity   |

The order each audit states in its own Proactive behavior, which is facts, then correctness, then
performance, then style, is a FIX order. It exists because applying a fix from an earlier audit
changes what a later audit would judge. No audit modifies anything, so that order constrains the
sequence in which the USER acts on the report rather than the sequence in which the audits run.

**Waves.** Run the audits in two waves, and run the members of a wave concurrently.

| Wave | Audits                                     | Why they may share a wave                             |
|------|--------------------------------------------|-------------------------------------------------------|
| 1    | `/audit-facts`, `/audit-style`             | Neither reads the other's output                      |
| 2    | `/audit-correctness`, `/audit-performance` | Both read the shared context, not each other's output |

Facts owns the claim inside a documentation block and style owns the form of that block, so wave 1
splits cleanly. What sits between the wave 2 members is a routing rule rather than a data dependency,
so the merge step settles it after both finish.

Wave 1 completes before wave 2 starts, because `/audit-correctness` adjudicates the ownership ladder
against wave 1's documentation verdicts. Wave 1 also runs the deterministic gates, whose diagnostics
every later wave reads instead of re-deriving.

**Parallelize at exactly ONE level.** Each audit already fans out internally over file batches, and
nesting a fan-out inside a fan-out multiplies the instruction payload by both factors.

| Target size      | Parallel level                                                               |
|------------------|------------------------------------------------------------------------------|
| Under 10 files   | WAVE. One sub-agent per audit, and each audit works sequentially inside it   |
| 10 or more files | BATCH. Audits run one at a time in wave order, each fanning out over batches |

Full detail, including the shared-context schema and the sub-agent budget, lives in
[execution-plan.md](references/execution-plan.md).

---

## Workflow

You MUST follow these steps when this skill is invoked.

Copy this progress checklist into your response and check off items as you complete them:

```text
Project Audit Progress:
- [ ] Step 0: Mode resolved, target resolved, plan produced and confirmed
- [ ] Step 1: Shared context built
- [ ] Step 2: Audits selected and the skipped ones recorded
- [ ] Step 3: Wave 1 complete (facts, style)
- [ ] Step 4: Wave 2 complete (correctness, performance)
- [ ] Step 5: Findings merged and ownership collisions adjudicated
- [ ] Step 6: Report produced, and in change mode the gate applied
```

### Step 0: Resolve the mode and the target, then pause

Resolve the mode from the invocation. Implementation work that just finished resolves to change mode.
A named repository, package, or directory resolves to full mode. Ask when neither is clear.

Emit ONE plan covering the whole run:

- The mode, and for change mode the base revision and the resolved file list
- Files in scope, grouped by language and by kind, with counts
- Which of the four audits will run, and which are skipped with the reason
- The parallel level the target size selects
- Whether coverage artifacts are present, stale, or absent

Pause for user confirmation or a "proceed" signal.

This plan REPLACES the Step 0 plan of each individual audit. Tell every audit that its own Step 0 is
already satisfied and that it MUST NOT pause again, otherwise four separate confirmations interrupt
one run.

### Step 1: Build the shared context

Build it ONCE on the main agent, and write it to the session scratch directory. Every audit receives
it as its Step 1 output and RECORDS it rather than re-deriving it.

The schema is in [execution-plan.md](references/execution-plan.md). It carries the file inventory with
each file's authority binding, the per-language prerequisites, the coverage ranking, and the
CONTRACT, STATE, and CALLGRAPH ledgers.

Building this once is what stops a four-audit run from reading the same file set four times.

### Step 2: Select the audits

Route from what the change actually contains rather than from file extensions alone, using the routing
table in [change-mode.md](references/change-mode.md). A skipped audit is recorded in the report with
its reason, so a thin run is visible rather than silent.

Full mode runs all four audits over the whole target and skips nothing.

### Step 3: Run wave 1

Run `/audit-facts` and `/audit-style` at the parallel level Step 0 selected. Hand each one the shared
context, the change-set narrowing where change mode applies, and the instruction that its Step 0 is
satisfied.

Collect the deterministic-gate diagnostics `/audit-style` produced into the shared context, so wave 2
reads them rather than re-deriving them.

### Step 4: Run wave 2

Run `/audit-correctness` and `/audit-performance` at the same parallel level. Hand `/audit-correctness`
wave 1's DRIFT and WRONG verdicts alongside the shared context, so its ownership ladder adjudicates
against findings that already exist rather than re-deriving them.

### Step 5: Merge and adjudicate

Apply the rules in [report-merge.md](references/report-merge.md), which deduplicate one construct
reported by two audits, resolve the three ownership collisions the audits leave to a caller, and
collapse one root cause reported at several sites.

Every finding keeps the severity, confidence, and evidence its owning audit assigned. This step
removes duplicates and settles ownership, and it never re-rates a finding.

### Step 6: Produce the report and apply the gate

Use the output format below.

In full mode the report is the deliverable, and the run ends there.

In change mode, apply the gate in [change-mode.md](references/change-mode.md). A BLOCKED verdict names
the findings that must be resolved, hands them back for fixing OUTSIDE the audits, and re-runs the
audits over the union of the original change set and the files the fixes touched. Three rounds cap the
loop, after which the report states what remains and the decision passes to the user.

---

## Output format

Open with the merged triage header from [report-merge.md](references/report-merge.md), then the
combined coverage ledger, then the findings.

```text
Mode: <FULL | CHANGE, base <revision>>
Gate: <PASSED | BLOCKED | ADVISORY ONLY | CAPPED after 3 rounds>   (change mode only)

| Audit               | Ran | Findings | Skipped because           |
|---------------------|-----|----------|---------------------------|
| /audit-facts        | yes | <n>      |                           |
| /audit-style        | yes | <n>      |                           |
| /audit-correctness  | yes | <n>      |                           |
| /audit-performance  | no  | 0        | <reason from the routing> |
```

Group the findings by AUDIT, then follow each audit's own output format unchanged inside its section,
including its ordering and its trailing LOW confidence appendix. A reader who wants one audit's report
finds it whole rather than interleaved with three others.

Findings that survived adjudication against another audit carry an `Adjudicated:` line naming the
audit that yielded and the rule that decided it.

---

## Discipline

You MUST adhere to the following discipline during every run.

- Never perform detection work yourself. This skill selects, sequences, and merges, and every finding
  comes from an audit that produced it under its own evidence floor.
- Never re-rate a finding. Severity, impact, confidence, and evidence belong to the owning audit.
- Never let an audit pause for its own Step 0 confirmation once Step 0 here is confirmed.
- Never narrow an audit to a change set in full mode.
- Never apply a fix inside an audit. Fixes happen between rounds, outside the audits, and the next
  round re-audits the files they touched.
- Never resolve a finding by weakening a test, a contract, a docstring, or a coverage setting. A gate
  satisfied that way is a gate defeated.
- Record every skipped audit with its reason, and every capped loop with what remained.

---

## Related skills

| Skill                   | Relationship                                                                  |
|-------------------------|-------------------------------------------------------------------------------|
| `/audit-facts`          | Wave 1 member, owns documentation claims that disagree with the code          |
| `/audit-style`          | Wave 1 member, owns style, formatting, and documentation quality              |
| `/audit-correctness`    | Wave 2 member, owns active and latent bugs and broken stated contracts        |
| `/audit-performance`    | Wave 2 member, owns cost, speed, memory use, and numeric width predictability |
| `/explore-codebase`     | Provides project structure context, invoke first on an unfamiliar codebase    |
| `/explore-dependencies` | Provides ataraxis API snapshots the audits verify library calls against       |
| `/commit`               | Runs after a change-mode gate passes, and never before it                     |

---

## Proactive behavior

Invoke this skill in change mode after completing any unit of implementation work, which covers a
feature, a bugfix, a refactor, and a documentation change, and BEFORE offering to commit. The point is
that new code passes the audits it is subject to while the work is still in context.

Invoke it in full mode when the user asks to audit a project, a package, or a repository, rather than
running the four audits by hand in sequence.

Prefer this skill over invoking a single audit whenever more than one audit applies to the target,
because a single-audit run pays the shared discovery again and produces a report a reader must merge
by hand.

Do NOT make code changes during an audit round. Present findings and, in change mode, fix between
rounds rather than inside them.

---

## Verification checklist

You MUST verify the run against this checklist before presenting the report.

```text
Project Audit Orchestration Compliance:
- [ ] Mode resolved explicitly, and change mode used only for a change set
- [ ] One plan produced and confirmed before any audit started
- [ ] Every audit told its Step 0 is satisfied, and none paused again
- [ ] Shared context built once on the main agent and handed to every audit
- [ ] No audit re-derived the inventory, prerequisites, coverage ranking, or callgraph
- [ ] Audits selected by what the change contains, with every skipped audit recorded with its reason
- [ ] Wave 1 completed before wave 2 started
- [ ] Members of each wave run concurrently at the parallel level the target size selected
- [ ] Parallelism applied at exactly one level, never wave and batch together
- [ ] Deterministic-gate diagnostics collected in wave 1 and reused by wave 2
- [ ] /audit-correctness received wave 1's DRIFT and WRONG verdicts for its ownership ladder
- [ ] Findings merged, with one construct reported by two audits resolved to one owner
- [ ] Every adjudicated finding carries the audit that yielded and the rule that decided it
- [ ] No finding re-rated, and every severity, confidence, and evidence left as its audit assigned
- [ ] Merged triage header present, carrying per-audit counts and every discard count
- [ ] Each audit's section preserves its own output format, ordering, and LOW confidence appendix
- [ ] In change mode, the gate verdict stated and every blocking finding named
- [ ] In change mode, fixes applied between rounds rather than inside an audit
- [ ] In change mode, each re-run covered the original change set plus the files the fixes touched
- [ ] Loop capped at 3 rounds, with anything remaining stated in the report
- [ ] No finding resolved by weakening a test, a contract, a docstring, or a coverage setting
- [ ] No file modifications made during any audit round
```
