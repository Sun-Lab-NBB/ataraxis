---
name: skill-design
description: >-
  Generates, updates, and verifies Claude Code skill files and CLAUDE.md project instructions. Covers
  SKILL.md structure, YAML frontmatter, formatting conventions, inter-skill relationships, scope
  declarations, and verification checklists. Use when creating new skills, modifying existing skills,
  updating CLAUDE.md files, or when the user asks about skill conventions.
user-invocable: true
---

# Skill design

Generates, updates, and verifies Claude Code skill and project instruction files.

You MUST read this entire skill before creating or modifying any skill file or CLAUDE.md. The
verification checklists at the end are mandatory before submitting any work.

---

## Scope

**Covers:**
- SKILL.md structure, YAML frontmatter, and formatting conventions
- CLAUDE.md structure, section ordering, and formatting conventions
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

Effective skills are focused, composable, and verifiable. You MUST apply these principles when
designing any skill.

### Single responsibility

Each skill addresses one well-defined concern. A skill that tries to do too much becomes difficult
to maintain and triggers inconsistently. If a skill covers multiple unrelated tasks, split it.

**Scope declaration**: Every skill must make clear what it DOES and DOES NOT cover. This prevents
scope creep and helps the agent select the right skill for the task.

### Composability

Skills must work independently and combine freely without conflicts. No skill should assume or
require another skill's internal state. If skill A needs information from skill B, it must
reference skill B explicitly rather than duplicating its content.

**Test**: Can this skill be invoked in isolation and still produce correct results? If not, it has
a hidden dependency that must be made explicit.

### Degrees of freedom

Match instruction specificity to task fragility:

| Level  | Format                            | When to use                                       |
|--------|-----------------------------------|---------------------------------------------------|
| Low    | Specific commands, few parameters | Operations that must be exactly reproduced        |
| Medium | Pseudocode or parameterized steps | A preferred pattern exists but details may vary   |
| High   | Text instructions                 | Multiple valid approaches; agent chooses best one |

Use **low freedom** for operations that must be reproducible (commit message format, frontmatter
structure, formatting rules). Use **high freedom** for creative or analytical tasks (exploration
summaries, architectural analysis). Default to low freedom when in doubt — reproducibility and
consistency are preferred over flexibility.

### Terminology consistency

Every concept in a skill must have ONE canonical name used consistently:

- Do not use synonyms interchangeably ("skill" vs "agent" vs "role" for the same concept)
- Do not overload terms (using "template" to mean both a file structure and a code pattern)
- Define terms at their canonical home and reference from elsewhere, never duplicate definitions

### Verifiability

A skill is verifiable when its output can be checked against concrete criteria. Every skill must
include a verification checklist that the agent completes before submitting work. Checklists
prevent subjective quality assessments and ensure consistent compliance.

### Progressive disclosure

Keep SKILL.md under 500 lines. Move detailed reference material into separate files that the agent
loads on demand. See [progressive-disclosure.md](references/progressive-disclosure.md) for
concrete patterns and directory structure conventions.

---

## Skill creation workflow

You MUST follow these steps when creating a new skill from scratch.

### Step 1: Define the skill's purpose

Identify a single, well-defined concern the skill will address. Write a one-sentence description
of what the skill does. If the description requires "and" to connect unrelated concerns, consider
splitting into multiple skills.

### Step 2: Declare scope and triggers

Write the scope declaration (Covers / Does not cover) and the YAML description with explicit
trigger conditions. The description is the primary mechanism the agent uses to decide when to
invoke the skill, so trigger conditions must be specific and comprehensive.

**Good triggers:** "Use when the user asks to commit", "Use when writing new Python code",
"Use when the user invokes /skill-name".

**Bad triggers:** "Use when appropriate", "Use for coding tasks".

### Step 3: Determine degrees of freedom

For each aspect of the skill's behavior, decide the appropriate freedom level:

- **Low freedom** for operations that must be exactly reproducible (formatting rules, field specs)
- **Medium freedom** for preferred patterns with room for adaptation (workflow steps)
- **High freedom** for creative or analytical tasks (exploration, summarization)

### Step 4: Create the directory structure

```text
skill-name/
├── SKILL.md                  # Main instructions (always loaded)
└── references/               # Detailed material (loaded on demand)
    └── detailed-rules.md     # Only if SKILL.md would exceed 500 lines
```

Add `references/`, `examples/`, `scripts/`, or `assets/` directories only when needed. Do NOT
create README.md, CHANGELOG.md, or other auxiliary documentation files. The skill directory must
contain only what the agent needs to execute the task.

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

Run through the verification checklist at the end of this skill. Then invoke the new skill in the
current repository to verify it produces correct behavior. Test with both explicit invocation
(`/skill-name`) and contextual descriptions to confirm the trigger conditions work.

---

## SKILL.md conventions

### YAML frontmatter

Every SKILL.md requires YAML frontmatter with `name` and `description`. For the complete field
reference including all optional fields, see
[frontmatter-reference.md](references/frontmatter-reference.md).

**Required fields:**

```yaml
---
name: explore-codebase
description: >-
  Performs in-depth codebase exploration at the start of a coding session. Builds comprehensive
  understanding of project structure, architecture, key components, and patterns. Use at session
  start or when the user asks to understand the codebase.
user-invocable: true
---
```

**Name**: Must exactly match the parent directory name. Lowercase letters, digits, and hyphens
only, max 64 characters. Examples: `explore-codebase`, `commit`, `skill-design`.

**Description**: Third person. Include what the skill does AND when to use it. End with explicit
trigger conditions ("Use when..."). Max 1024 characters.

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

Not every skill requires all sections. Omit sections that do not apply, but always include the
workflow (or equivalent main content) and verification checklist.

### Scope declarations

Every skill must declare its boundaries using the Covers / Does not cover format:

```markdown
## Scope

**Covers:**
- Generating commit messages from local git changes
- Staging and committing files

**Does not cover:**
- Pushing to remote repositories
- Creating pull requests
- Branch management
```

### Inter-skill references

When a skill relates to other skills, declare the relationship explicitly using a table:

```markdown
## Related skills

| Skill               | Relationship                                                |
|---------------------|-------------------------------------------------------------|
| `/python-style`    | Provides coding conventions that inform code review skills  |
| `/commit`           | Should be invoked after completing code changes             |
| `/explore-codebase` | Provides project context that informs implementation skills |
```

### Proactive behavior

Skills may declare when the agent should proactively offer to invoke them. Place this in a
dedicated section at the end of the skill, before the verification checklist.

### Workflow chaining

When a skill's workflow naturally leads to another skill, document this as a final workflow step.

---

## Formatting conventions

### Line length

All skill and reference Markdown files must adhere to the **120 character line limit**. This
matches the Python code formatting standard.

- Wrap prose text at 120 characters
- Break long sentences at natural boundaries (after punctuation, between clauses)
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

Use fenced code blocks with language identifiers:

````markdown
```yaml
cues:
  - name: "A"
    code: 1
```

```python
def process_data() -> None:
    pass
```
````

### Section organization

Separate major sections with horizontal rules (`---`). Use `##` for major sections and `###` for
subsections.

### Voice

Skill files use two voice styles:

- **Descriptive content**: Third person imperative. Example: "Extracts zone positions from
  configuration files."
- **Agent directives**: Second person with "You MUST", "You should". Example: "You MUST use the
  Task tool." This deliberate use of strong directives ensures reproducible, consistent behavior
  across sessions and is an intentional project convention.

### Sentence case

Use sentence case for all section headers ("Verification checklist", not "Verification Checklist").

---

## CLAUDE.md conventions

The `CLAUDE.md` file at the project root provides project-wide instructions loaded at the start of
every session. For the complete CLAUDE.md reference including import syntax, modular rules, and
quality criteria, see [claude-md-reference.md](references/claude-md-reference.md). For common
mistakes, see [anti-patterns.md](references/anti-patterns.md).

### Structure

CLAUDE.md files use the following section order (omit sections that do not apply):

1. **Title**: `# Claude Code Instructions`
2. **Session start behavior**: What the agent should do at session start
3. **Style guide compliance**: Required style conventions
4. **Cross-referenced library verification**: Dependencies and version checking
5. **Available skills**: Table of project skills with descriptions
6. **MCP server**: MCP server documentation
7. **Downstream library integration**: Related libraries and coordination
8. **Project context**: Architecture, key areas, patterns, and standards

### Formatting rules

CLAUDE.md follows the same conventions as skill files with one difference:

- **Section separators**: Use `##` headings without horizontal rules between sections

### Voice

- **Descriptive content**: Third person. Example: "This library provides shared assets..."
- **Agent directives**: Second person with emphasis. Example: "You MUST invoke the `/python-style`
  skill..."

### Content guidelines

- Keep CLAUDE.md focused on project-specific instructions
- Reference skills rather than duplicating their content
- Include workflow guidance for common tasks
- Document integration points with other libraries
- Only include instructions the agent cannot infer from code inspection alone

---

## Related skills

| Skill               | Relationship                                                          |
|---------------------|-----------------------------------------------------------------------|
| `/python-style`     | Provides formatting conventions that skill files must also follow     |
| `/readme-style`     | Provides README conventions relevant when skills reference READMEs    |
| `/commit`           | Should be invoked after completing skill file changes                 |
| `/explore-codebase` | Provides project context needed when writing project-specific skills  |

---

## Verification checklist

You MUST verify your work against the appropriate checklist before submitting.

### Skill files (SKILL.md)

```text
Skill File Compliance:
- [ ] YAML frontmatter with `name` and `description`
- [ ] `user-invocable: true` set if skill is directly invocable via slash command
- [ ] Name matches parent directory name exactly
- [ ] Description in third person, includes what AND when to use (max 1024 chars)
- [ ] Scope declaration present (what skill covers and does not cover)
- [ ] Degrees of freedom appropriate (low for reproducible, high for creative tasks)
- [ ] All lines ≤ 120 characters (tables/code blocks may exceed for clarity)
- [ ] Tables use pretty formatting with aligned columns
- [ ] Major sections separated with horizontal rules (`---`)
- [ ] Code blocks include language identifiers
- [ ] Third person imperative for descriptions
- [ ] Second person for agent directives ("You MUST...")
- [ ] Sentence case for section headers
- [ ] SKILL.md under 500 lines (split to reference files if needed)
- [ ] References one level deep from SKILL.md (no chains)
- [ ] Inter-skill references documented if applicable
- [ ] Verification checklist included
- [ ] Terminology consistent (no synonyms or overloaded terms)
- [ ] No auxiliary documentation files (README.md, CHANGELOG.md, etc.)
```

### Project instructions (CLAUDE.md)

```text
CLAUDE.md Compliance:
- [ ] Title is `# Claude Code Instructions`
- [ ] All lines ≤ 120 characters (tables/code blocks may exceed for clarity)
- [ ] Tables use pretty formatting with aligned columns
- [ ] Code blocks include language identifiers
- [ ] Third person for descriptive content
- [ ] Second person with emphasis for directives ("You MUST...")
- [ ] Sections follow recommended order (Session Start, Style Guide, Skills, etc.)
- [ ] Uses `##` headings without horizontal rules between sections
- [ ] Workflow guidance included for common extension tasks
- [ ] Technical claims cross-referenced against codebase
- [ ] New skills listed in Available Skills table
- [ ] No personality instructions or generic advice
- [ ] No stale references to removed files or features
```
