---
name: audit-project
description: >-
  Orchestrates the four ataraxis audits in the order and concurrency their dependencies require, and merges their
  findings into one deduplicated report. Runs in full mode over a repository, package, or directory, or in change mode
  over the files a feature, bugfix, or refactor touched, where it gates the work until the new code passes. Use when
  completing any implementation task, before committing, when auditing a project end to end, or when the user invokes
  /audit-project.
user-invocable: true
---

# Project audit orchestration

Runs `/audit-facts`, `/audit-correctness`, `/audit-performance`, and `/audit-style` as one pass, resolves the ownership
collisions between them, and produces a single merged report.

You MUST read this entire skill, and load each reference file at the step that names it, before acting on that step. The
verification checklist at the end is mandatory before presenting the report.

---

## Scope

**Covers:**
- Selecting which of the four audits a target actually needs, and sequencing them
- Asking the user whether wave 2 runs, rather than sweeping a whole project for bugs and cost unasked
- Running the shared discovery ONCE, so four audits stop re-deriving one inventory, one set of prerequisites, one
  coverage ranking, and one callgraph
- Full mode over a repository, a package, or a directory
- Change mode over the files a feature, bugfix, or refactor touched, together with the fix-and-recheck loop that gates
  the work until the new code passes
- Telling `/audit-style` whether its project-scope layout pass applies, which it does once per run for a project root
  and never for a package or a single-file target
- Merging, deduplicating, and adjudicating findings across the four audits into one report

**Does not cover:**
- The detection work itself, which each audit owns and this skill never duplicates
- Code, documentation, or configuration modifications. The audits produce findings only, and the change-mode fixes
  happen OUTSIDE them, between rounds
- Committing, pushing, or opening a pull request (see `/commit` and `/pr`)
- Codebase exploration (see `/explore-codebase`)

---

## The two modes

| Mode       | Target                                 | Trigger                                                |
|------------|----------------------------------------|--------------------------------------------------------|
| **Full**   | A repository, package, or directory    | The user asks for a project audit                      |
| **Change** | The files a unit of work touched       | Implementation work finished, or the user names a diff |

Change mode is the ONE sanctioned use of the explicit change-set scoping each audit defines. Every audit keeps
whole-target coverage as its own default, and only this skill's change mode narrows it.

Full mode never resolves a diff. A user who asks to audit the repository gets the repository.

---

## Sequence and concurrency

Four audits share one file set, so the question is which of them may read the same files at the same time and which must
wait for another's verdict.

**Real dependencies.** Only these three exist, and everything else is independent.

| Consumer             | Depends on          | What flows                                                  |
|----------------------|---------------------|-------------------------------------------------------------|
| Every audit          | Shared context      | File inventory, prerequisites, coverage ranking, callgraph  |
| `/audit-correctness` | `/audit-facts`      | The DRIFT and WRONG verdicts its ownership ladder routes on |
| `/audit-performance` | Shared context      | The callgraph its hot-path census turns into multiplicity   |

The order each audit states in its own Proactive behavior, which is facts, then correctness, then performance, then
style, is a FIX order. It exists because applying a fix from an earlier audit changes what a later audit would judge. No
audit modifies anything, so that order constrains the sequence in which the USER acts on the report rather than the
sequence in which the audits run.

**Waves.** Run the audits in two waves, and run the members of a wave concurrently.

| Wave | Audits                                     | Why they may share a wave                             |
|------|--------------------------------------------|-------------------------------------------------------|
| 1    | `/audit-facts`, `/audit-style`             | Neither reads the other's output                      |
| 2    | `/audit-correctness`, `/audit-performance` | Both read the shared context, not each other's output |

Facts owns the claim inside a documentation block and style owns the form of that block, so wave 1 splits cleanly. What
sits between the wave 2 members is a routing rule rather than a data dependency, so the merge step settles it after both
finish.

Wave 1 completes before wave 2 starts, because `/audit-correctness` adjudicates the ownership ladder against wave 1's
documentation verdicts. Wave 1 also runs the deterministic gates, whose diagnostics every later wave reads instead of
re-deriving.

**The project-scope layout pass.** Wave 1 also carries `/audit-style`'s sweep of the project directory tree against its
archetype. That sweep runs once per run, on the main agent and never inside a batch sub-agent, and ONLY where the
resolved target is a project root, because a package or a single file carries no tree to judge. In change mode it
additionally requires that the change set CREATES or DELETES a file. Tell `/audit-style` which case applies, so the pass
runs once rather than once per batch or not at all, and carry the status it reports into the merged coverage ledger.

**Parallelize at exactly ONE level.** Each audit already fans out internally over file batches, and nesting a fan-out
inside a fan-out multiplies the instruction payload by both factors. A target under 10 files parallelizes at the WAVE
level, and a target of 10 or more files parallelizes at the BATCH level.

The level table, the shared-context schema, and the sub-agent budget live in
[execution-plan.md](references/execution-plan.md).

---

## Workflow

You MUST follow these steps when this skill is invoked.

Copy this progress checklist into your response and check off items as you complete them:

```text
Project Audit Progress:
- [ ] Step 0: Mode resolved, target resolved, wave 2 elected, plan confirmed
- [ ] Step 1: Shared context built
- [ ] Step 2: Audits selected and the ones not running recorded
- [ ] Step 3: Wave 1 complete (facts, style)
- [ ] Step 4: Wave 2 complete, or skipped because the election kept neither
- [ ] Step 5: Findings merged and ownership collisions adjudicated
- [ ] Step 6: Report produced, and in change mode the gate applied
```

### Step 0: Resolve the mode and the target, then pause

Resolve the mode from the invocation. Implementation work that just finished resolves to change mode. A named
repository, package, or directory resolves to full mode. Ask when neither is clear.

Emit ONE plan covering the whole run:

- The mode, and for change mode the base revision and the resolved file list
- Files in scope, grouped by language and by kind, with counts
- Which of the four audits will run, and which are not running with the reason
- The wave 2 election below, stated as a question the user answers
- The parallel level the target size selects
- Whether the `/audit-style` project-scope layout pass runs, stated as `run`, `skipped-not-a-project-root`, or
  `skipped-no-created-or-deleted-files`
- Whether coverage artifacts are present, stale, or absent

Pause for user confirmation or a "proceed" signal.

This plan REPLACES the Step 0 plan of each individual audit. Tell every audit that its own Step 0 is already satisfied
and that it MUST NOT pause again, otherwise four separate confirmations interrupt one run.

### The wave 2 election

Wave 1 is not elective. `/audit-facts` and `/audit-style` run wherever they have files, because documentation drift and
convention drift accumulate across a whole target and are read that way.

Wave 2 IS elective, and you MUST ask for it explicitly rather than inferring it from the target. Put the question in the
plan with all four answers stated:

| Answer           | Wave 2 runs                                   |
|------------------|-----------------------------------------------|
| Both             | `/audit-correctness` and `/audit-performance` |
| Correctness only | `/audit-correctness`                          |
| Performance only | `/audit-performance`                          |
| Neither          | Wave 1 alone, and the run ends after it       |

Recommend NEITHER in full mode. A correctness or performance sweep over a whole project returns a report scoped to
nothing the user chose, and both are acted on one module at a time rather than one repository at a time. A user who
wants them names the module, and that naming is the scoping those two audits were built around.

Recommend what the routing table selects in change mode, because a change set is already that scoping.

Ask nothing where the target holds no `source` file. Step 2's bound-file-set rule already drops both wave 2 members
there, so the election has nothing to decide.

### Step 1: Build the shared context

Build it ONCE on the main agent, and write it to the session scratch directory. Every audit RECORDS it rather than
re-deriving it, and it stands as the output of the discovery steps that audit would otherwise run. Those steps are Step
1 for `/audit-facts` and `/audit-style`, Steps 1 through 3 for `/audit-correctness`, and Steps 1 and 2 for
`/audit-performance`.

The schema is in [execution-plan.md](references/execution-plan.md). It carries the file inventory with each file's
authority binding, the per-language prerequisites, the coverage ranking, and the CONTRACT, STATE, and CALLGRAPH ledgers.

Building this once is what stops a four-audit run from reading the same file set four times.

### Step 2: Select the audits

Both modes select from what the target actually CONTAINS rather than from file extensions alone, and wave 2 additionally
requires the Step 0 election. An audit that did not run is recorded in the report with its reason, so a thin run is
visible rather than silent.

Change mode routes with the routing table in [change-mode.md](references/change-mode.md), which reads the change set and
recommends the wave 2 election.

Full mode never narrows an audit, and it runs every audit that has something to read. Membership comes from the `kind`
field of the Step 1 inventory rather than from a change set:

| Audit                | Bound file set                        | Skipped when                  |
|----------------------|---------------------------------------|-------------------------------|
| `/audit-facts`       | Metadata and in-source documentation  | Neither class is present      |
| `/audit-style`       | Every file matching its binding table | No file matches a binding row |
| `/audit-correctness` | `source`, plus `test` for Pass 10     | Neither kind is present       |
| `/audit-performance` | `source`                              | No `source` file is present   |

A documentation-only, skill-only, or configuration-only target therefore runs wave 1 alone. Spawning an audit whose
bound file set is empty pays its whole instruction payload for a report that cannot hold a finding, and it spends
sub-agent budget the audits with work still need.

Two reductions apply, and they compose. The bound-file-set rule above drops an audit with nothing to read, and the Step
0 election drops a wave 2 audit the user declined. Record each under its own reason, because a DECLINED audit was
offered and an EMPTY one could never have run.

Neither reduction narrows an audit that does run. It covers the whole target, and no finding is dropped because the
target is small or unusual.

### Step 3: Run wave 1

Run `/audit-facts` and `/audit-style` at the parallel level Step 0 selected. Hand each one the shared context, the
change-set narrowing where change mode applies, and the instruction that its Step 0 is satisfied.

Tell `/audit-style` whether the target is a project root, and in change mode whether the change set creates or deletes a
file, because those two facts decide whether its project-scope layout pass runs. Collect the status it reports, in the
vocabulary Step 0 states, for the merged coverage ledger.

Collect the deterministic-gate diagnostics `/audit-style` produced into the shared context, so wave 2 reads them rather
than re-deriving them.

### Step 4: Run wave 2

Run the members the Step 0 election kept, at the same parallel level. Hand `/audit-correctness` wave 1's DRIFT and WRONG
verdicts alongside the shared context, so its ownership ladder adjudicates against findings that already exist rather
than re-deriving them.

Where the election kept neither, skip wave 2 and go to Step 5, which merges wave 1 alone. Where it kept one, that audit
runs by itself and the wave costs one sub-agent rather than two.

### Step 5: Merge and adjudicate

Apply the rules in [report-merge.md](references/report-merge.md), which deduplicate one construct reported by two
audits, resolve the four ownership collisions the audits leave to a caller, and collapse one root cause reported at
several sites.

Every finding keeps the severity, impact, confidence, and evidence its owning audit assigned. This step removes
duplicates and settles ownership, and it never re-rates a finding.

### Step 6: Produce the report and apply the gate

Use the output format below.

In full mode the report is the deliverable, and the run ends there.

In change mode, apply the gate in [change-mode.md](references/change-mode.md). A BLOCKED verdict names the findings that
must be resolved, hands them back for fixing OUTSIDE the audits, and re-runs the audits over the union of the original
change set and the files the fixes touched. Three rounds cap the loop, after which the report states what remains and
the decision passes to the user.

---

## Output format

Open with the merged triage header from [report-merge.md](references/report-merge.md), then the combined coverage
ledger, then the findings. Both layouts are given there verbatim, and this skill emits them unchanged.

Group the findings by AUDIT, then follow each audit's own ordering inside its section, including its trailing LOW
confidence appendix. A reader who wants one audit's report finds it whole rather than interleaved with three others.

Every finding keeps the shared shape its own audit defines, which is a stable ID, a rank, a location line, and the
Wrong, Fix, and Impact bullets, with a Choice bullet where the audit cannot settle the question. The four audits use
distinct ID letters, `C` for correctness, `P` for performance, `S` for style, and `F` for facts, so identifiers stay
unique across the merged report and a reader answers with one identifier rather than with a file and a line.

A finding that survived adjudication against another audit names the audit that yielded and the rule that decided it at
the end of its Wrong bullet.

---

## Discipline

You MUST adhere to the following discipline during every run.

- Never perform detection work yourself. This skill selects, sequences, and merges, and every finding comes from an
  audit that produced it under its own evidence floor.
- Never re-rate a finding. Severity, impact, confidence, and evidence belong to the owning audit.
- Never let an audit pause for its own Step 0 confirmation once Step 0 here is confirmed.
- Never narrow an audit to a change set in full mode.
- Never run a wave 2 audit the user did not elect, and never infer the election from the target alone. A wave 2 audit is
  expensive and its findings are acted on per module, so the user chooses it.
- Never apply a fix inside an audit. Fixes happen between rounds, outside the audits, and the next round re-audits the
  files they touched.
- Never resolve a finding by weakening a test, a contract, a docstring, or a coverage setting. A gate satisfied that way
  is a gate defeated.
- Never apply a fix that breaks the public API or alters public behavior without the user's explicit approval, whatever
  the severity and whatever the gate. Present it, name what breaks and who calls it, and WAIT. See the approval section
  in [change-mode.md](references/change-mode.md).
- Record every audit that did not run under its own reason, DECLINED or EMPTY or a routing row, and every capped loop
  with what remained.
- Fill each line the plan, the gate verdict, and the merged header write to 120 characters before breaking it, under
  the wrap width rule `/python-style` defines. A line ending before column 100 while its next word would still fit is
  re-flowed.

---

## Related skills

| Skill                   | Relationship                                                               |
|-------------------------|----------------------------------------------------------------------------|
| `/audit-facts`          | Wave 1 member, owns documentation claims that disagree with the code       |
| `/audit-style`          | Wave 1 member, owns style, formatting, and documentation quality           |
| `/audit-correctness`    | Wave 2 member, owns active and latent bugs and broken stated contracts     |
| `/audit-performance`    | Wave 2 member, owns cost, speed, memory use, and dtype predictability      |
| `/explore-codebase`     | Provides project structure context, invoke first on an unfamiliar codebase |
| `/explore-dependencies` | Provides ataraxis API snapshots the audits verify library calls against    |
| `/commit`               | Runs after a change-mode gate passes, and never before it                  |
| `/pr`                   | Drafts the pull request summary after a change-mode gate passes            |

---

## Proactive behavior

Invoke this skill in change mode after completing any unit of implementation work, which covers a feature, a bugfix, a
refactor, and a documentation change, and BEFORE offering to commit. The point is that new code passes the audits it is
subject to while the work is still in context.

Invoke it in full mode when the user asks to audit a project, a package, or a repository, rather than running the four
audits by hand in sequence.

Prefer this skill over invoking a single audit whenever more than one audit applies to the target, because a
single-audit run pays the shared discovery again and produces a report a reader must merge by hand.

Invoke `/audit-correctness` or `/audit-performance` DIRECTLY when the user names a module and wants one of them over it.
That narrow, deliberate run is what those two audits are built for, and routing it through this skill adds an
orchestration a single-audit target does not need.

Do NOT make code changes during an audit round. Present findings and, in change mode, fix between rounds rather than
inside them.

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
- [ ] Audits selected by what the target contains, with every audit that did not run recorded
- [ ] Wave 2 election asked explicitly at Step 0, with all four answers offered
- [ ] Neither wave 2 audit run without the user electing it, and any decline recorded as DECLINED
- [ ] Wave 1 completed before wave 2 started
- [ ] Members of each wave run concurrently at the parallel level the target size selected
- [ ] Parallelism applied at exactly one level, never wave and batch together
- [ ] Deterministic-gate diagnostics collected in wave 1 and reused by wave 2
- [ ] Project-scope layout pass status collected from /audit-style and carried into the merged ledger
- [ ] /audit-correctness received wave 1's DRIFT and WRONG verdicts for its ownership ladder
- [ ] Findings merged, with one construct reported by two audits resolved to one owner
- [ ] Every adjudicated finding carries the audit that yielded and the rule that decided it
- [ ] No finding re-rated, and every severity, impact, confidence, and evidence left as its audit assigned
- [ ] Merged triage header present, carrying per-audit counts and every discard count
- [ ] Each audit's section preserves its own output format, ordering, and LOW confidence appendix
- [ ] In change mode, the gate verdict stated and every blocking finding named
- [ ] In change mode, fixes applied between rounds rather than inside an audit
- [ ] Every public API break and public behavior change presented for approval and waited on
- [ ] No fix needing approval applied to reach a PASSED verdict
- [ ] In change mode, each re-run covered the original change set plus the files the fixes touched
- [ ] Loop capped at 3 rounds, with anything remaining stated in the report
- [ ] No finding resolved by weakening a test, a contract, a docstring, or a coverage setting
- [ ] Plan, gate verdict, and merged header prose fill each line to 120 characters, with no line ending before column
      100 while its next word would still fit
- [ ] No file modifications made during any audit round
```
