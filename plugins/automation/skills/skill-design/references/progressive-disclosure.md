# Progressive disclosure patterns

Detailed guidance on structuring skill content across multiple files for optimal context usage.

---

## Core principle

Keep SKILL.md under 500 lines. Move detailed reference material into separate files that the agent
loads on demand. This preserves context window space for the actual task while keeping detailed
reference material available when needed.

---

## Directory structure

```text
skill-name/
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
