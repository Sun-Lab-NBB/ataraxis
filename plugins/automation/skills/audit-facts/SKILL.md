---
name: audit-facts
description: >-
  Performs a thorough fact-check audit of documentation against its authoritative source code,
  covering standalone documentation files and the docstrings and comments embedded in source code.
  Verifies every concrete claim and surfaces drift, contradictions, and substantive omissions with
  verbatim source citations. Use when auditing a README, CLAUDE.md, SKILL.md, Sphinx page, Python
  docstring, Doxygen block, or XML doc comment for factual accuracy, or when the user invokes /audit-facts.
user-invocable: true
---

# Documentation fact audit

Audits documentation against the authoritative source code, reporting only factual mismatches and
substantive omissions with verbatim source citations.

You MUST read this entire skill before starting an audit. The verification checklist at the end
is mandatory before submitting findings.

---

## Scope

**Covers:**
- Verifying claims in metadata documentation against source code (API names, signatures, file
  paths, behaviors, configuration values, workflow steps, cross-references, version pins)
- Verifying claims in in-source documentation against the implementation each block documents
  (parameters, return values, raised exceptions, attributes, units, ranges, invariants, defaults)
- Detecting drift between either documentation class and the current code state
- Identifying substantive omissions in documentation that partially covers a source area
- Verifying cross-file and cross-symbol references for accuracy
- Surfacing internal contradictions within a documentation file

**Does not cover:**
- Style, formatting, section ordering, or convention compliance (see `/audit-style`)
- Documentation quality such as density, length proportionality, typos, imperative mood,
  separator punctuation, type-signature restating, and narrate-the-code comments (see
  `/audit-style`)
- A callable, class, module, or file that carries no documentation at all, which is a style
  finding (see `/audit-style`)
- Code modifications or fact corrections (this skill produces findings only)
- Codebase exploration (see `/explore-codebase`)
- Verifying external library API claims requires reading the installed library. For ataraxis
  dependencies, invoke `/explore-dependencies` first to obtain a current API snapshot

---

## Documentation classes

Every audited artifact belongs to exactly one of two documentation classes. These are the
canonical names used throughout this skill.

**Metadata documentation** is a standalone file whose entire content is prose about the project.
Its authoritative source lives in other files.

**In-source documentation** is prose embedded inside a source file. It covers Python module,
class, function, and property docstrings and `#` comments, C++ Doxygen blocks (`@brief`,
`@param`, `@returns`, `@note`, `@warning`, `@tparam`) and `//` comments, and C# XML doc comments
(`<summary>`, `<param>`, `<returns>`, `<remarks>`, `<exception>`) and `//` comments. Its
authoritative source is the implementation the block sits on, together with the symbols that
implementation calls.

Both classes drift, and both are in scope. A repository-wide audit that inspects only metadata
documentation has covered a fraction of the project's factual surface.

---

## Workflow

You MUST follow these steps when this skill is invoked.

Copy this progress checklist into your response and check off items as you complete them:

```text
Audit Progress:
- [ ] Step 0: Plan produced and confirmed
- [ ] Step 1: Scope resolved, documentation classes bound, tier selected
- [ ] Step 2: Claims extracted
- [ ] Step 3: Claims verified
- [ ] Step 4: Omission pass complete
- [ ] Step 5: Reference pass complete
- [ ] Step 6: Contradictions surfaced
- [ ] Step 7: Sample verification complete
- [ ] Step 8: Coverage ledger assembled
- [ ] Step 9: Report produced
```

### Step 0: Produce audit plan and pause

Emit a plan before any verification work fires. The plan must list:

- Target file(s) in scope (resolved absolute paths), split by documentation class
- File counts per class, so the user sees the in-source volume before work starts
- Source code area authoritative for each metadata file
- Tier classification (small, medium, or large)
- Expected finding categories

Pause for user confirmation or a "proceed" signal. This catches misidentified targets before
tokens burn on the wrong scope. A user who wants a narrower run says so here, and the narrowing
is recorded in the Step 8 coverage ledger.

### Step 1: Resolve target and bind documentation classes

Resolve the target into the set of files in scope. A target that names a single file binds that
file to its own class and stops there.

When the target is a directory or a repository root, **both documentation classes are in scope**.
You MUST enumerate the in-source documentation files and carry them through every subsequent
step. Reducing a repository target to its metadata documentation is a scope error, and it is the
most frequent failure of this skill.

Enumerate with the project's version control index, which reports the files that actually ship:

```bash
# Metadata documentation in scope
git ls-files '*.md' '*.rst' '*.toml' '*.ini' '*.json' '*.yml' '*.yaml'

# In-source documentation in scope
git ls-files '*.py' '*.pyi' '*.h' '*.hpp' '*.cpp' '*.cs'
```

For a target outside version control, run the equivalent `find` over the same patterns. Add
untracked files reported by `git status --porcelain` when the audit covers work in progress.

Bind every enumerated file to its class and its authoritative source:

| File pattern                                                  | Class     | Authoritative source                                                     |
|---------------------------------------------------------------|-----------|--------------------------------------------------------------------------|
| `README.md`, `CLAUDE.md`, `AGENTS.md`                         | Metadata  | The packages, modules, CLI entry points, and registries the file names   |
| `SKILL.md`, skill `references/*.md`                           | Metadata  | The tools, workflows, and conventions the skill describes                |
| `docs/**/*.rst`, `docs/**/conf.py`                            | Metadata  | The documented public API and the project build configuration            |
| `pyproject.toml`, `tox.ini`, `platformio.ini`, `library.json` | Metadata  | The declared dependencies and environments, and the code that reads them |
| `.github/**/*.yml`, `.github/**/*.md`                         | Metadata  | The workflows, environments, and project layout the template references  |
| `*.py`, `*.pyi`                                               | In-source | The documented module, class, function, and the symbols they call        |
| `*.h`, `*.hpp`, `*.cpp`                                       | In-source | The documented file, class, method, and the symbols they call            |
| `*.cs`                                                        | In-source | The documented file, class, member, and the symbols they call            |

A file matching no row is out of scope. Record it in the coverage ledger as UNBOUND.

For files that reference external libraries, include the installed package location in the
authoritative source list for that file. For ataraxis dependencies, invoke
`/explore-dependencies` to obtain a current API snapshot before proceeding.

Classify the audit tier from the union of both classes:

| Tier   | Indicators                     | Execution                                                                  |
|--------|--------------------------------|----------------------------------------------------------------------------|
| Small  | 1 file, under 500 lines        | Main agent, sequential                                                     |
| Medium | 2–10 files                     | Main agent, file-by-file                                                   |
| Large  | 10+ files or full project root | Parallel `general-purpose` sub-agents, one per file or per directory group |

A repository-root target is always Large. Group Large-tier work by class: one sub-agent per
metadata file, and one sub-agent per package or per source directory for in-source
documentation. A sub-agent that receives a source directory audits every file in it.

Specify agent type per subsequent step:

| Step                                 | Agent type                   | Why                                     |
|--------------------------------------|------------------------------|-----------------------------------------|
| Per-file claim verification (Small)  | main                         | Sequential preserves citation precision |
| Per-file claim verification (Medium) | main                         | Sequential preserves citation precision |
| Per-file claim verification (Large)  | `general-purpose` (parallel) | Parallelizes across files               |
| Omission pass                        | main                         | Needs synthesis across claims           |
| Reference pass                       | main                         | Single-agent view required              |
| Contradiction pass                   | main                         | Single-agent view required              |
| Sample verification                  | main                         | Trust boundary                          |
| Coverage ledger                      | main                         | Owns the record of what was audited     |
| Final report                         | main                         | Format and ordering live here           |

Do NOT use the `Explore` agent type for verification work. Explore returns summaries rather
than verbatim citations and breaks the "verbatim quote" discipline.

### Step 2: Extract verifiable claims

Walk each target file top to bottom and extract every verifiable claim.

In metadata documentation, a claim is any verifiable assertion: API names and signatures, file
paths, directory layouts, canonical filenames, behaviors, configuration values, defaults,
workflow step orderings, cross-references, version pins, external-library API references,
numerical facts, existence assertions, and date or version markers.

In in-source documentation, a claim is any assertion the implementation can confirm:

- A summary line stating what the callable, class, module, or file does
- A parameter description, including the parameter's name, its presence, and any default it states
- A return description, including the value's meaning, its units, and its range
- A documented raised exception, and the condition documented as triggering it
- A documented class or dataclass attribute, and the value it is documented to hold
- A stated unit, range, format, shape, dtype, invariant, ordering, or thread-safety guarantee
- An inline comment asserting a value, a magic constant's derivation, an algorithm, or a reason
- A reference from a docstring or comment to another symbol, module, file, issue, or version

The following are NOT claims and must be skipped: subjective quality language, aspirational or
future-tense statements, motivational prose, and every wording concern that `/audit-style` owns.
Pedagogical "why" prose is a claim only when it contains a verifiable factual statement.

### Step 3: Verify each claim

For each claim, locate the authoritative source and compare. You MUST open the source file and
verify directly. Do NOT verify from memory or training data.

For metadata claims, read the module, tool, or configuration file the claim names. For
external-library claims, read the installed library under `.venv`, conda env, or
`site-packages`. For ataraxis dependencies, use the API snapshot from `/explore-dependencies`.

For in-source claims, the implementation is the authority. Apply these rules:

- Read the full body of the documented callable, class, or module before judging its
  documentation. A summary that looks wrong at the signature is often satisfied deeper in the body
- Compare documented parameters against the signature for name, presence, and stated default
- Compare the documented return against every `return` statement and the return annotation
- Compare documented exceptions against every exception the body raises, following helpers the
  body calls whenever the documentation attributes their failures to this callable
- Compare documented attributes against the attributes the class assigns
- Confirm every symbol, module, file, or version a comment names still exists under that name
- Follow the call before flagging. Behavior a docstring describes is often delegated to a helper,
  and delegated behavior still belongs to the documented callable
- A `# noqa` or `# type: ignore` code is a claim about a diagnostic. Report it as DRIFT only when
  the project's linter is available and confirms the line no longer produces that diagnostic.
  Without the linter, leave the suppression unreported

For Large-tier audits, spawn one `general-purpose` sub-agent per file (or per directory group
of related files). Each sub-agent receives the file paths, the documentation class, the
authoritative source scope, and the verdict categorization rules. Sub-agents return findings in
the output format defined below. The main agent synthesizes after all sub-agents complete.

For Small and Medium tiers, the main agent performs all verification sequentially.

Cite each source as `<path>:<line>` or `<path>:<line>-<line>`. For an in-source finding, the file
location and the source location commonly sit in the same file a few lines apart, and both are
still required.

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
| HIGH       | Verbatim source quote present, mismatch is mechanical              |
| MEDIUM     | Source quote present, mismatch requires interpretation             |
| LOW        | Pattern detected but source/claim mapping is inferred, not literal |

For UNVERIFIABLE findings, state what you searched for and where you looked.

### Step 4: Omission pass

Within the per-file source scope from Step 1 only, walk the source code and check whether the
documentation mentions every relevant public API, behavior, and constraint.

An omission is substantive when the documentation already partially covers a surface but misses
pieces of it. In metadata documentation, examples include documenting 3 of 5 public functions in
a module the file walks through, describing a workflow but skipping a required step, or listing
registries but missing one. In in-source documentation, examples include an `Args:` section that
documents 3 of 5 parameters, a `Raises:` section that omits an exception the body raises, an
`Attributes:` section missing an attribute the class assigns, and a Doxygen block carrying
`@param` tags for some parameters of a method.

Do NOT flag documentation for lacking an entire section, format, or convention, and do NOT flag a
callable, class, module, or file that carries no documentation at all. Both are style concerns
and belong to `/audit-style`.

### Step 5: Reference pass

For every "see X" or "documented in Y" reference in metadata documentation, verify that X exists
and contains what the file claims it contains.

For in-source documentation, apply the same check to symbol references: Sphinx cross-reference
specifiers in Python docstrings, `@see` and `@ref` tags in Doxygen blocks, `<see cref="...">`
elements in XML doc comments, and any module, file, or symbol a comment names.

Flag only broken or wrong references.

### Step 6: Internal contradiction pass

Identify cases where the documentation makes incompatible claims about the same thing. This
includes a metadata file contradicting itself, and a docstring contradicting a comment or another
docstring inside the same module about the same symbol.

### Step 7: Sample verification

Before emitting the report:

1. Sample 3 random findings from the candidate list (if fewer than 3, sample all). When the
   candidate list contains findings from both documentation classes, the sample must include at
   least one of each.
2. Re-read the cited source line(s) without looking at the original finding.
3. Re-derive whether the finding holds from scratch.
4. Discard any finding that does not survive re-verification.

This step catches the most common audit failure mode: confidently misquoting source.

### Step 8: Assemble the coverage ledger

Build the ledger that opens the report. It records what was audited, so a thin in-source pass is
visible rather than silent:

```text
| Documentation class | Files in scope | Files audited | Files skipped |
|---------------------|----------------|---------------|---------------|
| Metadata            | 12             | 12            | 0             |
| In-source           | 47             | 47            | 0             |
```

List every skipped and UNBOUND file by path with its reason. Skipping is allowed only when the
user narrowed the scope in Step 0, when a file is generated, or when a file is unreadable. A
Large-tier audit that produced no in-source findings still reports a non-zero audited count in
the in-source row, which distinguishes clean documentation from an unrun pass.

### Step 9: Produce the findings report

Use the output format below. Skip EXACT and SEMANTIC findings entirely. Report HIGH and MEDIUM
confidence findings by default. Report LOW confidence findings only when the user explicitly
requests them via `--include-low` or equivalent invocation.

---

## Output format

Open the report with the Step 8 coverage ledger. Then report only WRONG, DRIFT, CONTRADICTION,
OMISSION, and UNVERIFIABLE findings, in that order.

When the audit spans multiple files, group findings hierarchically: documentation class -> file
-> finding type -> findings.

Each finding uses this structure:

```text
[Type]: <WRONG | DRIFT | CONTRADICTION | OMISSION | UNVERIFIABLE>
[Class]: <METADATA | IN-SOURCE>
[Confidence]: <HIGH | MEDIUM | LOW>
Location in file: <path>:<line>-<line>
Location in source: <path>:<line> or N/A
Claim: "<verbatim quote from the documentation>"
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
  by default. Report them as UNVERIFIABLE only when they appear internally contradicted or stale,
  and otherwise omit them, or set the Suggested fix to "leave as-is — authoritative external
  requirement" rather than a removal or change. `/readme-style` is the source of the canonical
  install-section requirements (e.g. the mamba 2.3.2+ / miniforge3 floor).
- Never paraphrase source and present it as a verbatim quote. Use the Read tool and copy.
- Never expand scope to restructure, restyle, or refactor. This skill produces findings only.
- Never flag style, formatting, structural, or convention issues. Those belong to
  `/audit-style`. A docstring or comment enters this report only when a fact inside it disagrees
  with the code.
- The implementation is authoritative over the documentation, so the Suggested fix edits the
  documentation by default. When the code is the side that looks wrong, say so in the Suggested
  fix and leave the choice to the user.
- Re-exports count: if the documentation says X is in module Y, and X is re-exported by Y but
  defined elsewhere, that is EXACT, not WRONG.
- Do not flag subjective preferences (tone, ordering, terminology).

---

## Related skills

| Skill                   | Relationship                                                                            |
|-------------------------|-----------------------------------------------------------------------------------------|
| `/audit-style`          | Sibling audit for style, formatting, documentation quality, and convention compliance   |
| `/explore-codebase`     | Provides project structure context; invoke first when auditing an unfamiliar codebase   |
| `/explore-dependencies` | Provides ataraxis API snapshots; invoke before verifying external API claims            |
| `/python-style`         | Defines the docstring sections whose contents this skill verifies against Python code   |
| `/cpp-style`            | Defines the Doxygen tags whose contents this skill verifies against C++ code            |
| `/csharp-style`         | Defines the XML doc tags whose contents this skill verifies against C# code             |
| `/readme-style`         | Provides README conventions for context (compliance is handled by `/audit-style`)       |
| `/api-docs`             | Provides Sphinx and Doxygen build conventions for context when auditing `docs/` pages   |
| `/skill-design`         | Provides SKILL.md and CLAUDE.md conventions for context (compliance via `/audit-style`) |

---

## Proactive behavior

Invoke this skill when the user asks to fact-check, verify, or audit documentation against the
source code. A request that names a directory or a repository covers both documentation classes
by default, and the Step 0 plan is where the user narrows that scope.

Run before `/audit-style` when auditing the same file end to end. Factual corrections may rewrite
prose that style would otherwise restyle redundantly.

Do NOT make code or documentation changes during the audit. Present findings and wait for user
direction.

---

## Verification checklist

You MUST verify the audit output against this checklist before presenting it to the user.

```text
Documentation Fact Audit Compliance:
- [ ] Step 0 plan produced and confirmed by user before verification began
- [ ] Plan listed per-class file counts, including the in-source count
- [ ] Both documentation classes enumerated for every directory or repository target
- [ ] Every file in scope bound to a class and an authoritative source per the binding table
- [ ] Tier classified (small/medium/large) and agent allocation matched the table
- [ ] For Large tier, parallel `general-purpose` sub-agents used and findings synthesized
- [ ] Every claim categorized (EXACT, SEMANTIC, DRIFT, WRONG, UNVERIFIABLE)
- [ ] Every claim assigned a confidence tier (HIGH, MEDIUM, LOW)
- [ ] Every non-EXACT and non-SEMANTIC finding cites a source location <path>:<line>
- [ ] Every claim verified by reading the source file (no memory-based verification)
- [ ] Full body of every documented callable read before its documentation was judged
- [ ] External library claims verified against installed library, not training data
- [ ] Omission pass executed and bounded by Step 1 scope (no out-of-scope omission claims)
- [ ] Cross-file and cross-symbol references verified for accuracy
- [ ] Internal contradictions surfaced
- [ ] Sample verification complete, covering both classes when both produced findings
- [ ] Coverage ledger present, with every skipped and UNBOUND file listed by path and reason
- [ ] In-source row of the ledger shows a non-zero audited count for repository targets
- [ ] LOW-confidence findings excluded unless explicitly requested
- [ ] No EXACT or SEMANTIC findings appear in the report
- [ ] No style, formatting, convention, or documentation-quality findings appear in the report
- [ ] No wholly undocumented callable, class, module, or file reported as an omission
- [ ] No file modifications made during the audit
- [ ] Findings ordered: WRONG -> DRIFT -> CONTRADICTION -> OMISSION -> UNVERIFIABLE
- [ ] Suggested fixes are concrete textual edits, not abstract recommendations
```
