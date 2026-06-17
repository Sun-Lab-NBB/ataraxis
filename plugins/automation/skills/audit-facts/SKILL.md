---
name: audit-facts
description: >-
  Performs a thorough fact-check audit of documentation files against their authoritative
  source code. Verifies every concrete claim, surfaces drift, contradictions, and substantive
  omissions, and produces a structured findings report with verbatim source citations. Use when
  auditing README files, CLAUDE.md, SKILL.md, Sphinx documentation, or any project
  documentation for factual accuracy against the codebase. Use when the user invokes
  /audit-facts.
user-invocable: true
---

# Documentation fact audit

Audits documentation files against the authoritative source code, reporting only factual
mismatches and substantive omissions with verbatim source citations.

You MUST read this entire skill before starting an audit. The verification checklist at the end
is mandatory before submitting findings.

---

## Scope

**Covers:**
- Verifying claims in documentation against source code (API names, signatures, file paths,
  behaviors, configuration values, workflow steps, cross-references, version pins)
- Detecting drift between documentation and current code state
- Identifying substantive omissions in documentation that partially covers a source area
- Verifying cross-file references for accuracy
- Surfacing internal contradictions within a documentation file

**Does not cover:**
- Style, formatting, section ordering, or convention compliance (see `/audit-style`)
- Code modifications or fact corrections (this skill produces findings only)
- Codebase exploration (see `/explore-codebase`)
- Verifying external library API claims requires reading the installed library; for ataraxis
  dependencies, invoke `/explore-dependencies` first to obtain a current API snapshot

---

## Workflow

You MUST follow these steps when this skill is invoked.

Copy this progress checklist into your response and check off items as you complete them:

```text
Audit Progress:
- [ ] Step 0: Plan produced and confirmed
- [ ] Step 1: Scope resolved and tier selected
- [ ] Step 2: Claims extracted
- [ ] Step 3: Claims verified
- [ ] Step 4: Omission pass complete
- [ ] Step 5: Cross-file references checked
- [ ] Step 6: Contradictions surfaced
- [ ] Step 7: Sample verification complete
- [ ] Step 8: Report produced
```

### Step 0: Produce audit plan and pause

Emit a plan before any verification work fires. The plan must list:

- Target file(s) in scope (resolved absolute paths)
- Source code area authoritative for each file
- Tier classification (small, medium, or large)
- Expected finding categories

Pause for user confirmation or a "proceed" signal. This catches misidentified targets before
tokens burn on the wrong scope.

### Step 1: Resolve target and scope

Resolve the target into the set of documentation files in scope. The target may be a single
file, a directory, or a project root. For directories or project roots, the scope is every
documentation file under the target (README, RST, CLAUDE.md, SKILL.md, AGENTS.md, etc.).

For each file in scope, identify the source code area authoritative for that file: directories,
modules, tools, registries, and public APIs that the file explicitly covers. Each file's source
scope is independent. Subsequent steps are bounded by the per-file scope.

For files that reference external libraries, include the installed package location in the
authoritative source list for that file. For ataraxis dependencies, invoke
`/explore-dependencies` to obtain a current API snapshot before proceeding.

Classify the audit tier:

| Tier   | Indicators                     | Execution                                                                   |
|--------|--------------------------------|-----------------------------------------------------------------------------|
| Small  | 1 file, under 500 lines        | Main agent, sequential                                                      |
| Medium | 2–10 files                     | Main agent, file-by-file                                                    |
| Large  | 10+ files or full project root | Parallel `general-purpose` sub-agents, one per file or per directory group  |

Specify agent type per subsequent step:

| Step                                 | Agent type                   | Why                                     |
|--------------------------------------|------------------------------|-----------------------------------------|
| Per-file claim verification (Small)  | main                         | Sequential preserves citation precision |
| Per-file claim verification (Medium) | main                         | Sequential preserves citation precision |
| Per-file claim verification (Large)  | `general-purpose` (parallel) | Parallelizes across files               |
| Omission pass                        | main                         | Needs synthesis across claims           |
| Cross-file consistency               | main                         | Single-agent view required              |
| Contradiction pass                   | main                         | Single-agent view required              |
| Sample verification                  | main                         | Trust boundary                          |
| Final report                         | main                         | Format and ordering live here           |

Do NOT use the `Explore` agent type for verification work. Explore returns summaries rather
than verbatim citations and breaks the "verbatim quote" discipline.

### Step 2: Extract verifiable claims

Walk each target file top to bottom and extract every verifiable claim.

A claim is any verifiable assertion: API names and signatures, file paths, directory layouts,
canonical filenames, behaviors, configuration values, defaults, workflow step orderings,
cross-references, version pins, external-library API references, numerical facts, existence
assertions, and date or version markers.

The following are NOT claims and must be skipped: subjective quality language, aspirational or
future-tense statements, motivational prose. Pedagogical "why" prose is a claim only when it
contains a verifiable factual statement.

### Step 3: Verify each claim

For each claim, locate the authoritative source and compare. You MUST open the source file and
verify directly. Do NOT verify from memory or training data.

For external-library claims, read the installed library under `.venv`, conda env, or
`site-packages`. For ataraxis dependencies, use the API snapshot from `/explore-dependencies`.

For Large-tier audits, spawn one `general-purpose` sub-agent per file (or per directory group
of related files). Each sub-agent receives the file path, the source scope list, and the
verdict categorization rules. Sub-agents return findings in the output format defined below.
The main agent synthesizes after all sub-agents complete.

For Small and Medium tiers, the main agent performs all verification sequentially.

Cite each source as `<path>:<line>` or `<path>:<line>-<line>`.

Categorize every claim using one of:

| Verdict      | Meaning                                                                    |
|--------------|----------------------------------------------------------------------------|
| EXACT        | Claim matches source verbatim                                              |
| SEMANTIC     | Claim is correct in meaning but uses different wording                     |
| DRIFT        | Claim was true at one point but has changed in source                      |
| WRONG        | Claim is factually incorrect                                               |
| UNVERIFIABLE | Source is missing, or claim is too vague to verify after reasonable search |

Also assign a confidence tier to every finding:

| Confidence | Meaning                                                            |
|------------|--------------------------------------------------------------------|
| HIGH       | Verbatim source quote present; mismatch is mechanical              |
| MEDIUM     | Source quote present, mismatch requires interpretation             |
| LOW        | Pattern detected but source/claim mapping is inferred, not literal |

For UNVERIFIABLE findings, state what you searched for and where you looked.

### Step 4: Omission pass

Within the per-file source scope from Step 1 only, walk the source code and check whether the
target file mentions every relevant public API, behavior, and constraint.

An omission is substantive when the file already partially covers a surface but misses pieces
of it. Examples include documenting 3 of 5 public functions in a module the file walks through,
describing a workflow but skipping a required step, or listing registries but missing one.

Do NOT flag the file for lacking an entire section, format, or convention. That is a style
concern and belongs to `/audit-style`.

### Step 5: Cross-file consistency pass

For every "see X" or "documented in Y" reference in the target file, verify that X exists and
contains what the file claims it contains. Flag only broken or wrong cross-references.

### Step 6: Internal contradiction pass

Identify cases where the target file makes incompatible claims about the same thing.

### Step 7: Sample verification

Before emitting the report:

1. Sample 3 random findings from the candidate list (if fewer than 3, sample all).
2. Re-read the cited source line(s) without looking at the original finding.
3. Re-derive whether the finding holds from scratch.
4. Discard any finding that does not survive re-verification.

This step catches the most common audit failure mode: confidently misquoting source.

### Step 8: Produce the findings report

Use the output format below. Skip EXACT and SEMANTIC findings entirely. Report HIGH and MEDIUM
confidence findings by default. Report LOW confidence findings only when the user explicitly
requests them via `--include-low` or equivalent invocation.

---

## Output format

Report only WRONG, DRIFT, CONTRADICTION, OMISSION, and UNVERIFIABLE findings, in that order.

When the audit spans multiple files, group findings hierarchically: file -> finding type ->
findings.

Each finding uses this structure:

```text
[Type]: <WRONG | DRIFT | CONTRADICTION | OMISSION | UNVERIFIABLE>
[Confidence]: <HIGH | MEDIUM | LOW>
Location in file: <path>:<line>-<line>
Location in source: <path>:<line> or N/A
Claim: "<verbatim quote from file>"
Source reality: "<verbatim quote or factual summary with citation>"
Suggested fix: <concrete textual edit>
```

For UNVERIFIABLE findings, replace the source reality with a description of what was searched
and where.

---

## Discipline

You MUST adhere to the following discipline during every audit.

- Never invent source. If you cannot open the source after reasonable search, mark UNVERIFIABLE.
- Facts not derivable from the audited repo's own source (external toolchain version floors,
  installer requirements, cross-repo version pins, environment prerequisites) are authoritative
  by default. Report them as UNVERIFIABLE only when they appear internally contradicted or stale;
  otherwise omit them, or set the Suggested fix to "leave as-is — authoritative external
  requirement" rather than a removal or change. `/readme-style` is the source of the canonical
  install-section requirements (e.g. the mamba 2.3.2+ / miniforge3 floor).
- Never paraphrase source and present it as a verbatim quote. Use the Read tool and copy.
- Never expand scope to restructure, restyle, or refactor. This skill produces findings only.
- Never flag style, formatting, structural, or convention issues. Those belong to
  `/audit-style`.
- Re-exports count: if the file says X is in module Y, and X is re-exported by Y but defined
  elsewhere, that is EXACT, not WRONG.
- Do not flag subjective preferences (tone, ordering, terminology).

---

## Related skills

| Skill                   | Relationship                                                                            |
|-------------------------|-----------------------------------------------------------------------------------------|
| `/audit-style`          | Sibling audit for style, formatting, and convention compliance                          |
| `/explore-codebase`     | Provides project structure context; invoke first when auditing an unfamiliar codebase   |
| `/explore-dependencies` | Provides ataraxis API snapshots; invoke before verifying external API claims            |
| `/python-style`         | Provides Python conventions referenced when source claims involve Python code           |
| `/readme-style`         | Provides README conventions for context (compliance is handled by `/audit-style`)       |
| `/skill-design`         | Provides SKILL.md and CLAUDE.md conventions for context (compliance via `/audit-style`) |

---

## Proactive behavior

Invoke this skill when the user asks to fact-check, verify, or audit any documentation file
against the source code. Run before `/audit-style` when auditing the same file end to end.
Factual corrections may rewrite prose that style would otherwise restyle redundantly.

Do NOT make code or documentation changes during the audit. Present findings and wait for user
direction.

---

## Verification checklist

You MUST verify the audit output against this checklist before presenting it to the user.

```text
Documentation Fact Audit Compliance:
- [ ] Step 0 plan produced and confirmed by user before verification began
- [ ] Tier classified (small/medium/large) and agent allocation matched the table
- [ ] For Large tier, parallel `general-purpose` sub-agents used and findings synthesized
- [ ] Step 1 scope explicitly enumerated for each file (directories, modules, registries, APIs)
- [ ] Every claim categorized (EXACT, SEMANTIC, DRIFT, WRONG, UNVERIFIABLE)
- [ ] Every claim assigned a confidence tier (HIGH, MEDIUM, LOW)
- [ ] Every non-EXACT and non-SEMANTIC finding cites a source location <path>:<line>
- [ ] Every claim verified by reading the source file (no memory-based verification)
- [ ] External library claims verified against installed library, not training data
- [ ] Omission pass executed and bounded by Step 1 scope (no out-of-scope omission claims)
- [ ] Cross-file references verified for accuracy
- [ ] Internal contradictions surfaced
- [ ] Sample verification (3 random findings re-derived) complete
- [ ] LOW-confidence findings excluded unless explicitly requested
- [ ] No EXACT or SEMANTIC findings appear in the report
- [ ] No style, formatting, or convention findings appear in the report
- [ ] No file modifications made during the audit
- [ ] Findings ordered: WRONG -> DRIFT -> CONTRADICTION -> OMISSION -> UNVERIFIABLE
- [ ] Suggested fixes are concrete textual edits, not abstract recommendations
```
