---
name: skill-design
description: >-
  Generates, updates, and verifies Claude Code skill files and CLAUDE.md and AGENTS.md project instructions. Covers
  SKILL.md structure, YAML frontmatter, formatting conventions, inter-skill relationships, scope declarations, and
  verification checklists. Use when creating or modifying skills, updating CLAUDE.md or AGENTS.md files, or when the
  user asks about skill conventions.
user-invocable: false
---

# Skill design

Generates, updates, and verifies Claude Code skill and project instruction files.

You MUST read this entire skill before creating or modifying any skill file or CLAUDE.md. The verification checklists at
the end are mandatory before submitting any work.

---

## Scope

**Covers:**
- SKILL.md structure, YAML frontmatter, and formatting conventions
- CLAUDE.md and AGENTS.md structure, section ordering, and formatting conventions
- Skill creation workflow from scratch
- Inter-skill relationships, scope declarations, and progressive disclosure
- Verification checklists for skill files and project instructions

**Does not cover:**
- Python code style conventions (see `/python-style`)
- README file conventions (see `/readme-style`)
- Commit message conventions (see `/commit`)
- Codebase exploration workflows (see `/explore-codebase`)

---

## Design principles

Effective skills are focused, composable, and verifiable. You MUST apply these principles when designing any skill.

### Single responsibility

Each skill addresses one well-defined concern. A skill that tries to do too much becomes difficult to maintain and
triggers inconsistently. If a skill covers multiple unrelated tasks, split it.

**Scope declaration**: Every skill must make clear what it DOES and DOES NOT cover. This prevents scope creep and helps
the agent select the right skill for the task.

### Composability

Skills must work independently and combine freely without conflicts. No skill should assume or require another skill's
internal state. If skill A needs information from skill B, it must reference skill B explicitly rather than duplicating
its content.

**Test**: Can this skill be invoked in isolation and still produce correct results? If not, it has a hidden dependency
that must be made explicit.

### Degrees of freedom

Match instruction specificity to task fragility:

| Level  | Format                            | When to use                                       |
|--------|-----------------------------------|---------------------------------------------------|
| Low    | Specific commands, few parameters | Operations that must be exactly reproduced        |
| Medium | Pseudocode or parameterized steps | A preferred pattern exists but details may vary   |
| High   | Text instructions                 | Multiple valid approaches, agent chooses best one |

Use **low freedom** for operations that must be reproducible (commit message format, frontmatter structure, formatting
rules, field specs). Use **medium freedom** for preferred patterns with room for adaptation (workflow steps). Use **high
freedom** for creative or analytical tasks (exploration summaries, architectural analysis). Default to low freedom when
in doubt, because reproducibility and consistency are preferred over flexibility.

### Terminology consistency

Every concept in a skill must have ONE canonical name used consistently:

- Do not use synonyms interchangeably ("skill" vs "agent" vs "role" for the same concept)
- Do not overload terms (using "template" to mean both a file structure and a code pattern)
- Define terms at their canonical home and reference from elsewhere, never duplicate definitions

### Verifiability

A skill is verifiable when its output can be checked against concrete criteria. Every skill must include a verification
checklist that the agent completes before submitting work. Checklists prevent subjective quality assessments and ensure
consistent compliance.

**Rules name their test**: A rule stated as a quality is unenforceable, because two readers grade it differently. State
instead the procedure that decides compliance, along with the default the rule departs from. "Documentation is concise"
is a preference. "The default is the summary line alone, and a longer block is earned by a nameable non-obvious
property" is a rule, because an agent is able to apply it and a reviewer is able to contest the result.

**Checklists separate the enforced from the enforceable**: Where a formatter or a linter already resolves an item, group
those items together and name the command that settles them. An item a tool already enforces spends attention the agent
owes to the items nothing else checks, and mixing the two kinds is the most common way a long checklist stops working.

### Progressive disclosure

Keep SKILL.md under 500 lines. See [progressive-disclosure.md](references/progressive-disclosure.md) for concrete
patterns and directory structure conventions.

---

## Workflow

You MUST follow these steps when creating a new skill from scratch.

### Step 1: Define the skill's purpose

Identify a single, well-defined concern the skill will address. Write a one-sentence description of what the skill does.
If the description requires "and" to connect unrelated concerns, consider splitting into multiple skills.

### Step 2: Declare scope and triggers

Write the scope declaration (Covers / Does not cover) and the YAML description with explicit trigger conditions. The
description is the primary mechanism the agent uses to decide when to invoke the skill, so trigger conditions must be
specific and comprehensive.

**Good triggers:** "Use when the user asks to commit", "Use when writing new Python code", "Use when the user invokes
/skill-name".

**Bad triggers:** "Use when appropriate", "Use for coding tasks".

### Step 3: Determine degrees of freedom

For each aspect of the skill's behavior, decide the appropriate freedom level using the degrees of freedom table above.

### Step 4: Create the directory structure

In this repo every skill lives under `plugins/<plugin-name>/skills/<skill-name>/`. The plugin's
`.claude-plugin/plugin.json` with `"skills": "./skills/"` is what registers the skills directory:

```text
plugins/<plugin-name>/skills/skill-name/
├── SKILL.md                  # Main instructions (always loaded)
└── references/               # Detailed material (loaded on demand)
    └── detailed-rules.md     # Only if SKILL.md would exceed 500 lines
```

Add `references/`, `examples/`, `scripts/`, or `assets/` directories only when needed. Do NOT create README.md,
CHANGELOG.md, or other auxiliary documentation files. The skill directory must contain only what the agent needs to
execute the task.

### Step 5: Write the SKILL.md

Follow the SKILL.md conventions below. You MUST include:

1. YAML frontmatter with all required fields
2. One-sentence description
3. Scope declaration
4. Workflow or main content
5. Related skills table (if applicable)
6. Proactive behavior (if applicable, before verification checklist)
7. Verification checklist

### Step 6: Validate and test

Run through the verification checklist at the end of this skill. Then invoke the new skill in the current repository to
verify it produces correct behavior. Test with both explicit invocation (`/skill-name`) and contextual descriptions to
confirm the trigger conditions work.

A new skill under an existing plugin needs no manual registration, because the plugin.json `"skills": "./skills/"` glob
auto-discovers it. If the new skill introduces a new plugin, add that plugin to the repo's
`.claude-plugin/marketplace.json` `plugins` array. Adding or materially changing a plugin's skills should bump `version`
in that plugin's `.claude-plugin/plugin.json`.

Bump that `version` EXACTLY ONCE per branch, relative to `main`. Read the branch's version and the `main` version before
editing, with `git show main:plugins/{plugin}/.claude-plugin/plugin.json`, and leave the version untouched wherever the
branch already carries a bump. A branch that revises ten skills across twenty commits ships one bump rather than twenty,
because the version names the release the branch produces rather than the edits inside it.

---

## SKILL.md conventions

### YAML frontmatter

Every SKILL.md requires YAML frontmatter with `name` and `description`. For the complete field reference including all
optional fields, see [frontmatter-reference.md](references/frontmatter-reference.md).

**Required fields:**

```yaml
---
name: explore-codebase
description: >-
  Performs in-depth codebase exploration at the start of a coding session. Builds comprehensive
  understanding of project structure, architecture, key components, and patterns. Use when starting
  a new session or when the user asks to understand the codebase or its architecture.
user-invocable: true
---
```

**Name**: Must exactly match the parent directory name. Lowercase letters, digits, and hyphens only, max 64 characters.
Examples: `explore-codebase`, `commit`, `skill-design`.

**Description**: Third person. Include what the skill does AND when to use it. End with explicit trigger conditions
("Use when..."). Max 1024 characters, and keep the folded description to 5 wrapped lines or fewer. Trim wordy
descriptions to the essential coverage and triggers to avoid frontmatter bloat.

**`user-invocable`**: Set to `true` for skills invokable via `/skill-name`. Defaults to `true`.

### Structure

A well-structured skill follows this layout:

```markdown
# Skill title

Brief description of the skill's purpose (one sentence).

---

## Scope

What this skill covers and what it does NOT cover.

---

## Workflow

Step-by-step instructions the agent follows when the skill is invoked.

---

## Rules / conventions

Detailed rules, formatting requirements, and examples.

---

## Related skills

Table of skills this skill interacts with and how.

---

## Proactive behavior

When the agent should proactively offer to invoke this skill (before verification checklist).

---

## Verification checklist

Mandatory checklist the agent completes before submitting work.
```

Not every skill requires all sections. Omit sections that do not apply, but always include the workflow (or equivalent
main content) and verification checklist.

### Scope declarations

Every skill must declare its boundaries in a `## Scope` section holding a `**Covers:**` bullet list followed by a
`**Does not cover:**` bullet list, as in the Scope section of this file.

### Inter-skill references

When a skill relates to other skills, declare the relationship explicitly using a table:

```markdown
## Related skills

| Skill               | Relationship                                                |
|---------------------|-------------------------------------------------------------|
| `/python-style`     | Provides coding conventions that inform code review skills  |
| `/commit`           | Should be invoked after completing code changes             |
| `/explore-codebase` | Provides project context that informs implementation skills |
```

### Proactive behavior

Skills may declare when the agent should proactively offer to invoke them. Place this in a dedicated section at the end
of the skill, before the verification checklist.

### Workflow chaining

When a skill's workflow naturally leads to another skill, document this as a final workflow step.

---

## Formatting conventions

### Line length

All skill and reference Markdown files must adhere to the **120 character line limit**. This matches the Python code
formatting standard.

- Break a prose line only where it would otherwise pass 120 characters, and fill each line to that limit before
  breaking. Prose wrapped at a narrower width reads as a rigid block and re-wraps badly at any other viewport. The test
  is mechanical: a wrapped line that ends before column 100 while its next word would still fit under 120 is re-flowed.
  A line ending early because the sentence or the paragraph ends, or because it holds a table row, a list item, or a
  code span, is already correct.
- Code blocks may exceed 120 characters only when necessary for readability
- Tables may exceed 120 characters when proper column alignment aids clarity

### Table formatting

Use **pretty table formatting** with proper column alignment:

```markdown
| Field  | Type | Required | Description                              |
|--------|------|----------|------------------------------------------|
| `name` | str  | Yes      | Visual identifier (e.g., 'A', 'Gray')    |
| `code` | int  | Yes      | Unique uint8 code for MQTT communication |
```

Rules:

1. Align all `|` characters vertically
2. Size each column to fit its widest cell, but no wider
3. Use dashes (`-`) that span the full column width
4. Pad narrower cells to match the column width
5. Use backticks for field names, types, and values

### Code blocks

Every fence carries a language identifier, such as `yaml`, `python`, `text`, or `markdown`. A block that shows Markdown
containing its own fences uses a four-backtick outer fence.

### Section organization

Separate major sections with horizontal rules (`---`). Use `##` for major sections and `###` for subsections.

### Voice

Skill files use two voice styles:

- **Descriptive content**: Third-person imperative mood. Example: "Extracts zone positions from configuration files."
- **Agent directives**: Second person with "You MUST", "You should". Example: "You MUST use the Agent tool."

### Sentence case

Use sentence case for all section headers ("Verification checklist", not "Verification Checklist").

### Content restraint

The default for a rule is one sentence, and examples, tables, and motivation are earned rather than assumed. Sentences
over 40 words are broken at a natural clause boundary, in SKILL.md, reference files, and CLAUDE.md alike. Cover each
sentence and delete it when you are able to reconstruct it from the skill name, the section heading, and the rule it
sits under. A section starts with its rule, so an opening sentence that announces the section or restates the
frontmatter description is deleted. Every skill file, reference file, and CLAUDE.md is free of typos and grammatical
errors. A rule appears once per file, because a file loads as a unit and a second copy inside it earns nothing. The same
rule in both SKILL.md and a reference file is permitted, because SKILL.md loads on every invocation while a reference
loads only when the agent opens it. See [progressive-disclosure.md](references/progressive-disclosure.md) for the full
rule set.

### Prose punctuation and positive description

Prose in skill files and CLAUDE.md follows the same two rules the language style skills apply to code documentation.
Prose uses only the full stop and the comma to separate clauses. Do not use a semicolon or an em-dash (`--`, `—`, or
`–`) as a separator, and use a colon only where it is lexically appropriate. A single hyphen stays available as a list
marker, in tables, and in compound words, so bulleted change lists are unaffected. State what the skill does and what is
currently true. Do not frame it by what it is not or what it used to be, and keep a "not Y" contrast only when it is
load-bearing because it corrects a counter-intuitive assumption, giving its reason.

---

## CLAUDE.md conventions

The `CLAUDE.md` file at the project root provides project-wide instructions loaded at the start of every session.
`AGENTS.md` is the vendor-neutral name for that same file, read by agent tools that do not load `CLAUDE.md`. No ataraxis
or sollertia repository carries one today, and a project that adds one follows every CLAUDE.md convention unchanged. For
the complete reference including import syntax, modular rules, quality criteria, and the AGENTS.md rules, see
[claude-md-reference.md](references/claude-md-reference.md). For common mistakes, see
[anti-patterns.md](references/anti-patterns.md).

### Structure

CLAUDE.md files open with the `# Claude Code Instructions` title and run through a fixed section order that ends in
project context. Two of its sections are conditional, applying to projects with a companion library or a separate
distribution channel for their agent assets. The ordering carries the canonical heading spellings, so the same concern
takes one name across every project. See [claude-md-reference.md](references/claude-md-reference.md) for the ordering
and the two conditional sections.

### Formatting rules

CLAUDE.md follows the same conventions as skill files with one difference:

- **Section separators**: Use `##` headings without horizontal rules between sections

### Voice

- **Descriptive content**: Third person. Example: "This library provides shared assets..."
- **Agent directives**: Second person with emphasis. Example: "You MUST invoke the `/python-style` skill..."

### Content guidelines

- Keep CLAUDE.md focused on project-specific instructions
- Reference skills rather than duplicating their content
- Include workflow guidance for common tasks
- Document integration points with other libraries
- Only include instructions the agent cannot infer from code inspection alone
- Leave the file no longer than it started unless the edit adds a genuinely new instruction

---

## Related skills

| Skill               | Relationship                                                                           |
|---------------------|----------------------------------------------------------------------------------------|
| `/python-style`     | Provides formatting conventions that skill files must also follow                      |
| `/cpp-style`        | Provides C++ conventions relevant when skills reference C++ code                       |
| `/csharp-style`     | Provides C# conventions relevant when skills reference C# code                         |
| `/project-layout`   | Provides general project directory conventions, while this skill owns `skills/` layout |
| `/readme-style`     | Provides README conventions relevant when skills reference READMEs                     |
| `/commit`           | Should be invoked after completing skill file changes                                  |
| `/pr`               | Owns pull request summaries, the change-narrating artifact skills exclude              |
| `/release`          | Owns release notes, the change-narrating artifact skills exclude                       |
| `/explore-codebase` | Provides project context needed when writing project-specific skills                   |

---

## Verification checklist

You MUST verify your work against the appropriate checklist before submitting.

### Skill files (SKILL.md)

```text
Skill File Compliance, tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines ≤ 120 characters (tables/code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines (split to reference files if needed)
- [ ] Tables use pretty formatting with aligned columns
- [ ] Major sections separated with horizontal rules (`---`)
- [ ] Code blocks include language identifiers
- [ ] Sentence case for section headers
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons, hyphen bullets, and code
      syntax exempt)

Skill File Compliance, reader-judged:
- [ ] YAML frontmatter with `name` and `description`
- [ ] `user-invocable: true` set if skill is directly invocable via slash command
- [ ] Name matches parent directory name exactly
- [ ] Description in third person, includes what AND when to use (max 1024 chars, ≤ 5 wrapped lines)
- [ ] Scope declaration present (what skill covers and does not cover)
- [ ] Skill addresses one well-defined concern, with unrelated tasks split into separate skills
- [ ] Skill produces correct results invoked in isolation, with every dependency on another skill referenced explicitly
- [ ] Degrees of freedom appropriate (low for reproducible, high for creative tasks)
- [ ] Third-person imperative mood for descriptions
- [ ] Second person for agent directives ("You MUST...")
- [ ] References one level deep from SKILL.md (no chains)
- [ ] Inter-skill references documented if applicable
- [ ] Verification checklist included
- [ ] Terminology consistent (no synonyms or overloaded terms)
- [ ] Every rule names the procedure that decides compliance and the default it departs from
- [ ] Checklist groups tool-enforced items separately and names the command that settles them
- [ ] Each rule defaults to one sentence, with examples and tables earned by the judgment they govern
- [ ] Each retained sentence survives the cover test (unable to be reconstructed from the skill name, the section
      heading, and the rule it sits under)
- [ ] No rule stated twice inside one file (the same rule in SKILL.md and in a reference is permitted)
- [ ] Sentences in skill prose, reference files, and CLAUDE.md stay under 40 words
- [ ] Prose fills each line to 120 characters, with no line ending before column 100 while its next word would still fit
- [ ] No section preamble restating the heading or the frontmatter description above it
- [ ] No rule restated from a skill that owns it (reference the owning skill instead)
- [ ] Rules duplicated across skills by necessity keep aligned wording
- [ ] Skill prose records the current skill only, never the edit that produced it
- [ ] Edits leave the skill no longer than it started unless new cases are genuinely covered
- [ ] Prose states what the skill does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Skill prose, reference files, and CLAUDE.md free of typos and grammar errors
- [ ] No auxiliary documentation files (README.md, CHANGELOG.md, etc.)
- [ ] New plugin registered in `.claude-plugin/marketplace.json`, and `version` bumped in the owning plugin's
      `.claude-plugin/plugin.json` for material skill changes, exactly once per branch relative to main and
      left untouched where the branch already carries a bump
```

### Project instructions (CLAUDE.md)

```text
CLAUDE.md Compliance, tool-settled (run `rg -n '.{121,}' <file>`, `rg -n '^---$' <file>`, and `wc -l <file>`):
- [ ] All lines ≤ 120 characters (tables/code blocks may exceed for clarity)
- [ ] CLAUDE.md under 300 lines (move domain rules into `.claude/rules/*.md` or `@` imports)
- [ ] Tables use pretty formatting with aligned columns
- [ ] Code blocks include language identifiers
- [ ] Uses `##` headings without horizontal rules between sections
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons, hyphen bullets, and code
      syntax exempt)

CLAUDE.md Compliance, reader-judged:
- [ ] Title is `# Claude Code Instructions`
- [ ] Third person for descriptive content
- [ ] Second person with emphasis for directives ("You MUST...")
- [ ] Sections follow recommended order (Session start behavior, Style guide compliance, Available skills, etc.)
- [ ] Section headings use the canonical spellings (Style guide compliance, MCP server)
- [ ] Optional sections (Companion library synchronization, Distribution model) present only where the concern applies
- [ ] Workflow guidance included for common extension tasks
- [ ] Technical claims cross-referenced against codebase
- [ ] New skills listed in the Available skills table
- [ ] No personality instructions or generic advice
- [ ] Prose states what the project does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Sentences in skill prose, reference files, and CLAUDE.md stay under 40 words
- [ ] Prose fills each line to 120 characters, with no line ending before column 100 while its next word would still fit
- [ ] No section preamble restating the heading or the frontmatter description above it
- [ ] Edits leave CLAUDE.md no longer than it started unless a genuinely new instruction was added
- [ ] Skill prose, reference files, and CLAUDE.md free of typos and grammar errors
- [ ] No stale references to removed files or features
```
