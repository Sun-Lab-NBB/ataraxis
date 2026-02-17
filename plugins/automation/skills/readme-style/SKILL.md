---
name: readme-style
description: >-
  Applies Sun Lab README conventions when creating or updating README.md files. Covers section
  ordering, writing style, standard section templates, badges, MCP server documentation, CLI
  documentation, and codebase cross-referencing. Use when writing a new README, updating an
  existing README, or when the user asks about README conventions.
user-invocable: true
---

# README style guide

Applies Sun Lab conventions for README.md files.

You MUST read this entire skill and load the section templates reference before creating or
modifying any README file. You MUST verify your changes against the verification checklist before
submitting.

---

## Scope

**Covers:**
- README.md section ordering and structure
- Writing style (voice, tense, notes/warnings)
- Standard section templates (badges, installation, developers, acknowledgments, etc.)
- MCP server and CLI documentation sections
- Codebase cross-referencing for technical accuracy
- PyPI rendering compatibility

**Does not cover:**
- Python code style (invoke `/python-style`)
- Commit message conventions (invoke `/commit`)
- Skill file and CLAUDE.md conventions (invoke `/skill-design`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file before making any changes.

### Step 2: Load section templates

Load [section-templates.md](references/section-templates.md). This file contains the exact
templates for every README section (badges, dependencies, installation, developers, MCP server,
standard ending sections, etc.). You MUST use these templates when writing new sections or
verifying existing ones.

### Step 3: Cross-reference the codebase

Before writing or updating technical content, verify all claims against the actual codebase. See
the [codebase cross-referencing](#codebase-cross-referencing) section for the verification
process.

### Step 4: Apply conventions

Write or modify the README following all conventions from this file and the loaded templates.

### Step 5: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before
submitting work.

---

## Section ordering

README files use the following section order. Sections marked as optional may be omitted based on
project type. For the exact template of each section, see
[section-templates.md](references/section-templates.md).

1. **Title**: Project name as H1 heading (`# project-name`)
2. **One-line description**: Bare project description, identical across all canonical locations for the archetype
3. **Badges**: Standard badge set for the project type (blank line separates description from badges)
4. **Horizontal rule**: `___` (triple underscore) to separate header from content
5. **Detailed description**: Expanded explanation of the library's purpose (`## Detailed Description`)
6. **Features** *(optional)*: Bulleted list of key features, ending with license type
7. **Table of contents**: Links to all major sections using lowercase Markdown anchors
8. **Dependencies**: External requirements and automatic installation notes
9. **Installation**: Source and pip installation instructions with standard templates
10. **Usage**: Detailed usage instructions with subsections
11. **API documentation**: Link to hosted documentation
12. **Developers** *(optional)*: Development setup, automation, and troubleshooting
13. **Versioning**: Semantic versioning statement with link to repository tags
14. **Authors**: List of contributors with GitHub profile links
15. **License**: License type with link to LICENSE file
16. **Acknowledgments**: Credits to Sun lab members and dependency creators

---

## Writing style

**Voice**: Use third person throughout. Refer to the project as "this library," "the library," or
by its name. Avoid first person ("I," "we") and second person ("you") where possible.

```markdown
<!-- Good -->
This library abstracts all necessary steps for acquiring and saving video data.
The library supports Windows, Linux, and macOS.

<!-- Avoid -->
We provide tools for acquiring and saving video data.
You can use this library on Windows, Linux, and macOS.
```

**Tense**: Use present tense as the default. Avoid "will" unless omitting it makes the sentence
awkward or unclear.

```markdown
<!-- Good - present tense -->
The method returns a tuple of timestamps.
This command generates a configuration file.

<!-- Good - "will" where natural -->
These dependencies will be automatically resolved when the library is installed.

<!-- Avoid - unnecessary "will" -->
The method will return a tuple of timestamps.
```

**Notes and warnings**: Use `***Note,***` for important information. Use `***Warning!***` or
`***Critical!***` for dangerous operations or essential requirements. Do not use GitHub-specific
alert syntax (`> [!NOTE]`) as it does not render on PyPI.

---

## Horizontal rules

Always use triple underscore (`___`) for horizontal rules between major sections. Do not use
triple dash (`---`) or triple asterisk (`***`).

```markdown
___

## Next Section
```

---

## Heading hierarchy

Use a single H1 (`#`) for the project title. All sections use H2 (`##`). Subsections use H3
(`###`). Never skip heading levels (do not jump from H2 to H4).

---

## Accessibility

- Every image must have meaningful alt text: `![Description of the image](url)`
- Avoid filenames as alt text (e.g., do not use `![screenshot1.png](...)`)
- Use descriptive link text that hints at the destination; avoid "click here" patterns
- Indicate file types for downloads: `[User Guide (PDF)](url)`

---

## PyPI compatibility

README content must render correctly on both GitHub and PyPI. Avoid GitHub-specific Markdown
features that do not render on PyPI:

- Do not use GitHub alert syntax (`> [!NOTE]`, `> [!WARNING]`)
- Do not use `<details>`/`<summary>` collapsible sections
- Do not use `<picture>` elements for dark/light mode images
- Do not use task lists (`- [x]`, `- [ ]`)

Use `***Note,***` and `***Warning!***` for callouts instead.

---

## Codebase cross-referencing

When writing or updating README content that describes how the library works, you MUST
cross-reference against the current state of the codebase to ensure accuracy.

**Sections requiring verification:**
- Architecture descriptions
- API usage examples
- Configuration options
- File paths and directory structures
- Class names, method signatures, and parameters
- Workflow descriptions

**Verification process:**
1. Identify all technical claims in the README section
2. Use `/explore-codebase` skill if unfamiliar with the relevant code
3. Read source files to verify each claim
4. Update README content to match actual implementation
5. Remove references to deprecated or non-existent features

---

## Related skills

| Skill               | Relationship                                                     |
|---------------------|------------------------------------------------------------------|
| `/python-style`     | Provides Python conventions; invoke for Python tasks             |
| `/cpp-style`        | Provides C++ conventions; invoke for C++ tasks                   |
| `/csharp-style`     | Provides C# conventions; invoke for C# tasks                     |
| `/pyproject-style`  | Provides pyproject.toml conventions; one-line description synced |
| `/project-layout`   | Provides project directory conventions; README is a root file    |
| `/skill-design`     | Provides skill conventions; invoke for skill authoring tasks     |
| `/commit`           | Provides commit message conventions; invoke after README changes |
| `/explore-codebase` | Provides project context for cross-referencing README claims     |

---

## Proactive behavior

When creating a new Sun Lab project, proactively offer to generate a README following these
conventions. When modifying code that affects documented behavior (API changes, new features,
removed functionality), proactively suggest updating the README to reflect the changes.

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to README
files.**

```text
README Style Compliance:

Structure:
- [ ] Title as H1 heading with project name (lowercase, hyphenated)
- [ ] One-line description immediately after title (bare, matches all canonical description locations)
- [ ] Correct badge set for project type (see section-templates.md)
- [ ] Blank line between description and badges
- [ ] Horizontal rule uses `___` (not `---`) after badges
- [ ] Detailed description section present (`## Detailed Description` heading, after horizontal rule)
- [ ] Features section ends with license type bullet (if Features present)
- [ ] Table of contents with lowercase Markdown anchors
- [ ] Spelling: "Acknowledgments" (not "Acknowledgements") everywhere
- [ ] Heading hierarchy: single H1, H2 for sections, H3 for subsections, no skips
- [ ] Horizontal rules use `___` consistently throughout (not `---`)

Content:
- [ ] All required sections present (Dependencies, Installation, Usage, etc.)
- [ ] Section templates followed (see section-templates.md)
- [ ] Installation uses standard Source warning and pip/source subsections
- [ ] Dependencies uses standard boilerplate text
- [ ] API Documentation links to hosted docs
- [ ] Developers section uses standard mamba/tox template (if present)
- [ ] Standard ending sections use correct templates (Versioning, Authors, License, Acknowledgments)
- [ ] MCP Server section titled "MCP Server" with tools table (if applicable)
- [ ] CLI commands documented with overview table (if applicable)

Style:
- [ ] Third-person voice throughout (no "I", "we", "you")
- [ ] Present tense as default
- [ ] `***Note,***` / `***Warning!***` for callouts (not GitHub alerts)
- [ ] No GitHub-specific features (alerts, details/summary, picture, task lists)

Quality:
- [ ] All images have meaningful alt text
- [ ] Link text is descriptive (no "click here")
- [ ] Technical descriptions cross-referenced against codebase
- [ ] File paths and class names verified to exist
- [ ] API examples tested against actual implementation
```
