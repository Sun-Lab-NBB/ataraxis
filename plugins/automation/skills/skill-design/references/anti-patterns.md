# Anti-patterns

Common mistakes to avoid when writing skill files and CLAUDE.md instructions.

---

## Skill anti-patterns

### Including auxiliary documentation files

Do NOT create README.md, INSTALLATION_GUIDE.md, QUICK_REFERENCE.md, CHANGELOG.md, or other
auxiliary files in the skill directory. The skill directory must contain only SKILL.md and optional
reference/example/script/asset subdirectories.

**Wrong:**

```text
my-skill/
├── SKILL.md
├── README.md              ← unnecessary
├── CHANGELOG.md           ← unnecessary
└── QUICK_REFERENCE.md     ← unnecessary
```

**Correct:**

```text
my-skill/
├── SKILL.md
└── references/
    └── detailed-rules.md  ← loaded on demand by the agent
```

### Duplicating content across skills

If two skills need the same information, one must reference the other rather than duplicating the
content. Duplication causes inconsistencies when one copy is updated but not the other.

**Wrong:** Copying the commit message rules into both `/commit` and `/python-style`.

**Correct:** `/python-style` references `/commit`: "For commit message conventions, invoke the
`/commit` skill."

### Overly broad scope

A skill that tries to cover too many concerns becomes difficult to maintain and triggers
inconsistently.

**Wrong:** A single skill covering Python style, README formatting, commit messages, and skill
design.

**Correct:** Four separate skills (`/python-style`, `/commit`, `/skill-design`) each with a
focused scope declaration.

### Missing trigger conditions

A description that only says what the skill does but not when to use it leads to inconsistent
triggering.

**Wrong:**

```yaml
description: Generates commit messages.
```

**Correct:**

```yaml
description: >-
  Drafts Sun Lab style-compliant git commit messages by analyzing all local changes relative to
  the active branch. Use when the user asks to commit, when completing a coding task that should
  be committed, or when the user invokes /commit.
```

### Generic instructions the agent already knows

Do not include instructions for things the agent handles well without guidance. These waste context
and dilute the impact of instructions that actually matter.

**Wrong:**

```markdown
## Error handling

When an error occurs, read the error message carefully and try to understand what went wrong
before attempting a fix.
```

**Correct:** Omit entirely. The agent already handles errors intelligently. Only include error
handling instructions when the project has specific, non-obvious error handling patterns.

---

## CLAUDE.md anti-patterns

### Excessive length

CLAUDE.md files over 300 lines begin to dilute the impact of critical instructions. Every line
competes for attention.

**Mitigation:** Use `@path/to/import` syntax or `.claude/rules/*.md` to keep CLAUDE.md concise.
Move domain-specific rules into modular rule files.

### Personality instructions

Personality directives waste tokens and do not improve output quality.

**Wrong:**

```markdown
You are a senior software engineer with 15 years of experience. You care deeply about code
quality and always write tests before implementation.
```

**Correct:** Omit entirely. Provide specific, actionable rules instead.

### Embedding full file contents

Importing entire source files with `@` loads all content every session, even if most of it is
irrelevant to the current task.

**Wrong:**

```markdown
@src/ataraxis_automation/automation.py
```

**Correct:** Reference the file by path and let the agent read it when needed:

```markdown
The core automation logic is in `src/ataraxis_automation/automation.py`. Read this file before
modifying environment management.
```

### Documenting obvious defaults

Do not document conventions the agent follows by default. Only document project-specific deviations
from standard conventions.

**Wrong:**

```markdown
- Use meaningful variable names
- Add comments to explain complex logic
- Follow PEP 8 conventions
```

**Correct:**

```markdown
- 120 character line limit (not PEP 8's 79)
- Google-style docstrings (not NumPy-style)
```

### Stale information

CLAUDE.md that references outdated file paths, removed features, or deprecated APIs causes the
agent to produce incorrect code.

**Mitigation:** You MUST cross-reference technical claims against the actual codebase before
committing CLAUDE.md changes. Include this check in the verification checklist.
