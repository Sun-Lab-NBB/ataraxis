---
name: audit-style
description: >-
  Performs a thorough style-compliance audit of source code, configuration, or documentation files
  against the applicable ataraxis framework style skill checklists. Walks every line in scope and
  reports only non-compliant findings with verbatim checklist citations. Use when auditing a Python
  package, config file, README, or any project file for style, formatting, naming, or convention
  compliance, or when the user invokes /audit-style.
user-invocable: true
---

# Style compliance audit

Audits files against the authoritative ataraxis framework style skill checklists, reporting only
non-compliant findings with verbatim checklist citations.

You MUST read this entire skill and load every applicable style skill checklist before starting
an audit. The verification checklist at the end is mandatory before submitting findings.

---

## Scope

**Covers:**
- Auditing single files, directories, or full project trees against applicable style skills
- Structural style: element ordering, imports, formatting, naming, type annotations,
  error-handling patterns, file-section ordering
- Comment and docstring quality: typos, sentence length, length proportionality, redundancy
  with the type signature, narrate-the-code comments, stale references, and
  docstring-implementation drift
- Cross-file consistency: naming, ordering, and idiom drift across the file set
- Cross-skill conflicts where two loaded style skills disagree on the same point

**Does not cover:**
- Factual accuracy of documentation against source code (see `/audit-facts`)
- Code modifications or style fixes (this skill produces findings only)
- Inventing new conventions not present in any loaded style skill checklist
- Codebase exploration (see `/explore-codebase`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

Copy this progress checklist into your response and check off items as you complete them:

```text
Audit Progress:
- [ ] Step 0: Plan produced and confirmed
- [ ] Step 1: Target resolved, style skills bound, tier selected
- [ ] Step 2: Style skill checklists loaded
- [ ] Step 3: Line-by-line sweep complete (all three dimensions)
- [ ] Step 4: Findings categorized
- [ ] Step 5: Sample verification complete
- [ ] Step 6: Report produced
```

### Step 0: Produce audit plan and pause

Emit a plan before any sweep work fires. The plan must list:

- Files in scope (resolved absolute paths)
- Style skills bound to those files (per the binding table in Step 1)
- Tier classification (small, medium, or large)
- Expected finding categories

Pause for user confirmation or a "proceed" signal. This catches misidentified targets before
tokens burn on the wrong scope.

### Step 1: Resolve target and bind style skills

Resolve the target (a single file, a package directory, or a project root) into the set of
files in scope. For directory or project-root targets, every file under the target is in scope.
There is no "covered area" reduction.

For each file in scope, identify the applicable style skill using the binding table:

| File pattern                                                              | Style skill          |
|---------------------------------------------------------------------------|----------------------|
| `*.py`                                                                    | `/python-style`      |
| `*.cs`                                                                    | `/csharp-style`      |
| `*.h`, `*.hpp`, `*.cpp`, `.clang-format`, `.clang-tidy`, `CMakeLists.txt` | `/cpp-style`         |
| `README.md`                                                               | `/readme-style`      |
| `pyproject.toml`                                                          | `/pyproject-style`   |
| `tox.ini`                                                                 | `/tox-config`        |
| `platformio.ini`, `library.json`                                          | `/platformio-config` |
| `docs/*.rst`, `conf.py`, `Makefile`, `make.bat`, `Doxyfile`               | `/api-docs`          |
| `SKILL.md`, `CLAUDE.md`, `AGENTS.md`                                      | `/skill-design`      |
| `.github/ISSUE_TEMPLATE/*.yml`                                            | `/project-layout`    |
| `envs/*`                                                                  | `/project-layout`    |
| Project directory tree                                                    | `/project-layout`    |

If a file in scope matches no binding row, mark it UNAUDITED in the plan and report (no
applicable style skill) and flag no findings against it.

Classify the audit tier:

| Tier   | Indicators                     | Execution                                                                   |
|--------|--------------------------------|-----------------------------------------------------------------------------|
| Small  | 1 file, under 500 lines        | Main agent, sequential                                                      |
| Medium | 2–10 files                     | Main agent, file-by-file                                                    |
| Large  | 10+ files or full project root | Parallel `general-purpose` sub-agents, one per file or per directory group  |

Specify agent type per subsequent step:

| Step                     | Agent type                   | Why                                     |
|--------------------------|------------------------------|-----------------------------------------|
| Per-file sweep (Small)   | main                         | Sequential preserves citation precision |
| Per-file sweep (Medium)  | main                         | Sequential preserves citation precision |
| Per-file sweep (Large)   | `general-purpose` (parallel) | Parallelizes across files               |
| Categorization           | main                         | Needs synthesis across findings         |
| Cross-file consistency   | main                         | Single-agent view required              |
| Sample verification      | main                         | Trust boundary                          |
| Final report             | main                         | Format and ordering live here           |

Do NOT use the `Explore` agent type for sweep work. Explore returns summaries rather than
verbatim citations and breaks the "verbatim checklist quote" discipline.

### Step 2: Load applicable style skill checklists

For every distinct style skill the bindings resolve to, invoke that skill and load its full
verification checklist along with every reference file the skill mentions. The loaded
checklists are the only source of truth for "applicable style point." A convention not present
in any loaded checklist is NOT a violation.

### Step 3: Line-by-line sweep

For every file in scope, walk top to bottom. For every line, evaluate against every applicable
checklist item. Track three parallel dimensions.

**Dimension A — Structural style:** Element ordering, imports, formatting, naming, type
annotations, error-handling patterns, and file-section ordering. The source of truth is the
loaded style skill's main checklist.

**Dimension B — Comment and docstring quality:** Apply the loaded style skill's docstring and
comment checklist to every comment, docstring, and inline annotation. Typical findings include
typos, grammar errors, and prose padded with restatements or trivia the reader can infer from the
code. They also include documentation whose length tracks the size of the code instead of the
difficulty of understanding it, docstrings that restate the type signature, and comments that
narrate obvious code behavior. Common content issues are sentences exceeding 40 words and stale references such as
closed issues, removed code, or outdated version markers. Docstring claims that contradict the
function's signature or observable behavior also qualify. They further include prose that separates
clauses with a semicolon or an em-dash where only full stops and commas belong, together with
contrastive or historical framing ("does X, not Y" or "formerly did Y") that should state present
behavior positively.

**Dimension C — Cross-file consistency:** Naming, ordering, and idiom drift across the file
set. Examples include the same field named differently in two sibling classes, or one module
following a convention that adjacent modules ignore.

List ALL violations in each section. Do NOT stop at the first.

For Large-tier audits, spawn one `general-purpose` sub-agent per file (or per directory group
of related files). Each sub-agent receives the file path, the loaded checklists, and the
categorization rules. Sub-agents return findings in the output format defined below. The main
agent synthesizes after all sub-agents complete.

For Small and Medium tiers, the main agent performs all sweep work sequentially.

### Step 4: Categorize findings

Categorize every violation using one of:

| Category      | Meaning                                                              |
|---------------|----------------------------------------------------------------------|
| BLOCKING      | Checklist explicitly states "MUST" or "blocks release"               |
| STANDARD      | Checklist convention without an explicit blocking flag               |
| INCONSISTENCY | File or file set mixes schemes the checklist permits only separately |
| CONFLICT      | Two loaded style skills disagree on the same point                   |

Also assign a confidence tier to every finding:

| Confidence | Meaning                                                                |
|------------|------------------------------------------------------------------------|
| HIGH       | Verbatim checklist quote and verbatim source quote both present        |
| MEDIUM     | One quote verbatim, the other requires interpretation                  |
| LOW        | Pattern detected but checklist/source mapping is inferred, not literal |

### Step 5: Sample verification

Before emitting the report:

1. Sample 3 random findings from the candidate list (if fewer than 3, sample all).
2. Re-read the cited file line(s) and the cited checklist point without looking at the
   original finding.
3. Re-derive whether the finding holds from scratch.
4. Discard any finding that does not survive re-verification.

This step catches the most common audit failure mode: misapplying a checklist rule.

### Step 6: Produce the findings report

Use the output format below. Skip compliant items entirely. Report HIGH and MEDIUM confidence
findings by default. Report LOW confidence findings only when the user explicitly requests
them via `--include-low` or equivalent invocation.

---

## Output format

For multi-file targets, group findings hierarchically: file -> category -> findings. Within
each file, order findings by severity: BLOCKING -> INCONSISTENCY -> CONFLICT -> STANDARD.

Each finding uses this structure:

```text
[Category]: <BLOCKING | STANDARD | INCONSISTENCY | CONFLICT>
[Confidence]: <HIGH | MEDIUM | LOW>
Skill: <skill name>
Checklist point: "<verbatim quote from the skill's checklist or reference file>"
Location: <path>:<line>-<line>
Current state: "<verbatim quote from the file>"
Required state: <concrete example or the checklist's "should be" form>
Suggested fix: <concrete textual edit>
```

When the same checklist point is violated multiple times within a file, collapse to a single
finding with a count and representative line citations:

```text
[Category]: STANDARD
[Confidence]: HIGH
Skill: /python-style
Checklist point: "Full words used (no abbreviations like pos, idx, val)"
Location: module.py:47, 89, 112, 134 (4 occurrences)
Current state: "idx" used as loop variable
Required state: "index"
Suggested fix: Rename idx -> index throughout module.py.
```

---

## Discipline

You MUST adhere to the following discipline during every audit.

- Anchor every finding to a verbatim checklist quote. No checklist quote, no finding.
- Never invent conventions. If a behavior is not in a loaded checklist, it is not a style
  violation.
- Never flag factual errors, missing content, or source-code mismatches. Those belong to
  `/audit-facts`.
- Never restructure, restyle, or refactor. This skill produces findings only.
- Never flag subjective preferences (tone, ordering, terminology) unless the loaded checklist
  explicitly requires the convention.
- If a file contains an auto-generated block or a documented exception, note the exception and
  skip its enclosing range.

---

## Related skills

| Skill                | Relationship                                                                                     |
|----------------------|--------------------------------------------------------------------------------------------------|
| `/audit-facts`       | Sibling audit for factual accuracy against source code                                           |
| `/python-style`      | Provides the Python style checklist; loaded when scope contains Python files                     |
| `/cpp-style`         | Provides the C++ style checklist; loaded when scope contains C++ files                           |
| `/csharp-style`      | Provides the C# style checklist; loaded when scope contains C# files                             |
| `/readme-style`      | Provides the README style checklist; loaded when scope contains README files                     |
| `/pyproject-style`   | Provides the pyproject.toml style checklist; loaded when scope contains pyproject.toml           |
| `/tox-config`        | Provides the tox.ini style checklist; loaded when scope contains tox.ini                         |
| `/platformio-config` | Provides the platformio.ini/library.json style checklist; loaded when scope contains those files |
| `/api-docs`          | Provides the Sphinx docs style checklist; loaded when scope contains docs files                  |
| `/skill-design`      | Provides the skill and CLAUDE.md style checklist; loaded when scope contains skill files         |
| `/project-layout`    | Provides the directory and issue template checklist, loaded for project roots and .github        |
| `/explore-codebase`  | Provides project structure context; invoke first when auditing an unfamiliar codebase            |

---

## Proactive behavior

Invoke this skill when the user asks to audit a file, package, or project for style compliance.
Run after `/audit-facts` when auditing the same file end to end. Style work performed before
factual corrections wastes effort on prose that may be rewritten.

Do NOT make code or documentation changes during the audit. Present findings and wait for user
direction.

---

## Verification checklist

You MUST verify the audit output against this checklist before presenting it to the user.

```text
Style Compliance Audit Output:
- [ ] Step 0 plan produced and confirmed by user before sweep began
- [ ] Tier classified (small/medium/large) and agent allocation matched the table
- [ ] For Large tier, parallel `general-purpose` sub-agents used and findings synthesized
- [ ] Step 1 file binding executed (every file in scope mapped to its applicable style skill)
- [ ] Step 2 checklists loaded for every applicable style skill
- [ ] Every file in scope walked top to bottom
- [ ] All three dimensions evaluated (structural, comment/docstring quality, cross-file consistency)
- [ ] Every finding anchored to a verbatim checklist quote
- [ ] Every finding cites a file location <path>:<line>
- [ ] Findings categorized (BLOCKING, STANDARD, INCONSISTENCY, CONFLICT)
- [ ] Every finding assigned a confidence tier (HIGH, MEDIUM, LOW)
- [ ] Repeated violations of the same checklist point collapsed with counts
- [ ] Sample verification (3 random findings re-derived) complete
- [ ] LOW-confidence findings excluded unless explicitly requested
- [ ] No compliant items appear in the report
- [ ] No factual errors, missing content, or source mismatches appear (those belong to /audit-facts)
- [ ] No findings invented outside the loaded checklists
- [ ] Findings ordered: BLOCKING -> INCONSISTENCY -> CONFLICT -> STANDARD
- [ ] Cross-skill conflicts surfaced rather than silently resolved
- [ ] Suggested fixes are concrete textual edits, not abstract recommendations
- [ ] No file modifications made during the audit
```
