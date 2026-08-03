# Progressive disclosure patterns

Detailed guidance on structuring skill content across multiple files for optimal context usage.

---

## Core principle

Keep SKILL.md under 500 lines. Move detailed reference material into separate files that the agent
loads on demand. This preserves context window space for the actual task while keeping detailed
reference material available when needed.

---

## Prose restraint

A skill file is loaded into the context of every session that triggers it, so its length is a
recurring cost paid by every future invocation. Write to the shortest form that still decides the
cases the skill exists to decide. Splitting an overlong file into references lowers the cost of any
one invocation, and writing less lowers it everywhere at once.

The default for a rule is one sentence. A worked example is earned when the rule governs a judgment
an agent gets wrong without seeing the correction, and a table is earned when the rule varies across
more than two cases. Motivation is written only where an agent that misunderstands it would apply
the rule incorrectly.

Do not restate a rule another skill owns. Reference the owning skill by name instead, which keeps
the two copies from drifting. Where a rule genuinely must appear in several skills, such as the
prose conventions each style skill states for its own medium, keep the wording aligned across every
copy so that a reader who has learned one recognizes the others.

Skill prose describes the skill as it currently stands, never the edit that produced it. Do not
record that a section was added, that a rule was relaxed, or that a convention changed. Editing a
skill is not a reason to lengthen it, so when a change leaves the surrounding rules intact, rewrite
in place and delete what the change made redundant. The artifacts whose subject is change itself,
which are commit messages, pull request summaries, and release notes, stay outside this rule and
are governed by `/commit`, `/pr`, and `/release`.

---

## Directory structure

```text
plugins/<plugin-name>/skills/skill-name/
├── SKILL.md              # Main instructions (loaded when triggered)
├── references/           # Detailed reference material (loaded on demand)
│   ├── field-reference.md
│   └── patterns.md
├── examples/             # Working examples (loaded on demand)
│   └── example-output.md
├── scripts/              # Validation and utility scripts
│   └── validate.sh
└── assets/               # Static resources (templates, schemas)
    └── template.yaml
```

### When to use each directory

| Directory     | Purpose                                        | When to use                       |
|---------------|------------------------------------------------|-----------------------------------|
| `references/` | Detailed rules, field specs, API documentation | Content too detailed for SKILL.md |
| `examples/`   | Working code samples, output examples          | Examples that exceed 20 lines     |
| `scripts/`    | Validation, scaffolding, testing utilities     | Automated operations              |
| `assets/`     | Templates, schemas, static config files        | Non-markdown resources            |

You MUST only add directories when they are needed. Do NOT create empty placeholder directories or
auxiliary documentation files (README.md, CHANGELOG.md, INSTALLATION_GUIDE.md).

---

## Disclosure patterns

### Pattern 1: High-level guide with references

SKILL.md provides an overview and workflow. Detailed rules live in reference files.

**Use when:** The skill has extensive rules or specifications that most invocations only partially
need.

**Structure:**

```markdown
# SKILL.md

## Formatting conventions

### Line length

All files must adhere to the 120 character line limit.

For complete formatting rules, see
[detailed-formatting.md](references/detailed-formatting.md).
```

### Pattern 2: Domain-specific organization

Reference files organized by domain or topic area. The agent loads only the relevant domain.

**Use when:** The skill serves multiple distinct use cases that each have their own rules.

**Structure:**

```markdown
# SKILL.md

## Workflow selection

| Task                  | Reference to load                                   |
|-----------------------|-----------------------------------------------------|
| Writing Python code   | [python-conventions.md](references/python.md)       |
| Writing README files  | [readme-conventions.md](references/readme.md)       |
```

### Pattern 3: Conditional details

Basic content lives inline in SKILL.md. Advanced or rarely-needed content lives in reference files.

**Use when:** Most invocations need only the basics, but some require deep detail.

**Structure:**

```markdown
# SKILL.md

## YAML frontmatter

Every SKILL.md requires `name` and `description`. Add `user-invocable: true` for slash commands.

For the complete field reference, see
[frontmatter-reference.md](references/frontmatter-reference.md).
```

---

## Referencing conventions

- Reference supplementary files using standard Markdown links:
  `[frontmatter-reference.md](references/frontmatter-reference.md)`
- References must be one level deep from SKILL.md (no chains of references to references)
- Each reference file must be self-contained (readable without other reference files)
- Reference files follow the same formatting conventions as SKILL.md (120 char limit, pretty tables,
  sentence case headers, code blocks with language identifiers)
