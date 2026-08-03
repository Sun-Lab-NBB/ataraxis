---
name: readme-style
description: >-
  Applies README conventions when creating or updating README.md files. Covers section
  ordering, writing style, standard section templates, badges, MCP server documentation, CLI
  documentation, and codebase cross-referencing. Use when writing a new README, updating an
  existing README, or when the user asks about README conventions.
user-invocable: false
---

# README style guide

Applies conventions for README.md files.

You MUST read this entire skill and load the section templates reference before creating or
modifying any README file. You MUST verify your changes against the verification checklist before
submitting.

---

## Scope

**Covers:**
- README.md section ordering and structure for the library and umbrella archetypes
- Writing style (voice, tense, notes/warnings)
- Standard section templates (badges, installation, developers, acknowledgments, etc.)
- MCP server and CLI documentation sections
- Codebase cross-referencing for technical accuracy
- PyPI rendering compatibility

**Does not cover:**
- Python code style (see `/python-style`)
- Commit message conventions (see `/commit`)
- Skill file and CLAUDE.md conventions (see `/skill-design`)

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

Two section orders exist. Select between them mechanically before writing anything:

- **Library README**: The repository ships an installable artifact of its own, such as a PyPI
  package, a PlatformIO registry library, or Unity package sources. Use the library order.
- **Umbrella README**: The repository ships no installable artifact and instead indexes sibling
  libraries or distributes plugins through a marketplace. Use the umbrella order.

A repository that ships an installable artifact and also indexes siblings is a library, so it takes
the library order and covers the siblings inside its own sections.

Sections marked as optional may be omitted based on project type. For the exact template of each
section, see [section-templates.md](references/section-templates.md).

### Library README order

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
12. **AI-Assisted Development** *(optional)*: Agent assets, promoted from its usual H3 position
13. **Developers** *(optional)*: Development setup, automation, and troubleshooting
14. **Versioning**: Semantic versioning statement with link to repository tags
15. **Authors**: List of contributors with GitHub profile links
16. **License**: License type with link to LICENSE file
17. **Acknowledgments**: Credits to dependency developers and everyone who contributed to the project

### Umbrella README order

An umbrella repository indexes other repositories, so the sections a library README spends on its
own dependencies, installation, API, and versioning have no subject here. The badge row carries the
license badge alone, because no published package exists to report a version, a Python version, a
wheel, or a release status for. The table of contents and the license bullet that closes Features
belong to the library order alone. Every other convention in this skill applies unchanged, including
the `___` horizontal rules, the heading hierarchy, and the writing style rules.

1. **Title**: Repository name as H1 heading in its display casing (`# Ataraxis`)
2. **Tagline**: Bold one-line statement of what the framework is for, in place of the bare
   one-line description
3. **Badges**: The license badge alone
4. **Framework description**: Two or three paragraphs stating what the framework provides and the
   principle that organizes it, written without a `## Detailed Description` heading
5. **Attribution**: Author line and copyright line
6. **Horizontal rule**: `___` (triple underscore) to separate header from content
7. **Features**: Bulleted capability list grouped under H3 headings
8. **Architecture** *(optional)*: Diagram of how the indexed components relate at runtime
9. **Libraries**: Every indexed repository, grouped under H3 category headings. Each entry is a bold
   link to the repository, its language in parentheses, and one or two sentences of description
10. **Getting Started**: Install commands for the indexed artifacts, one block per distribution channel
11. **Claude Code Plugins** *(optional)*: Marketplace installation steps and one table per distributed
    plugin listing what that plugin provides
12. **Example Workflows** *(optional)*: Fenced transcripts showing an agent driving the framework
13. **Adoption Roadmap** *(optional)*: Numbered steps a new adopter follows, ending with a pointer to
    any platform built on the framework
14. **Citation** *(optional)*: BibTeX entry for the framework
15. **License**: License covering every indexed repository
16. **Acknowledgments**: Credits to the projects the framework builds on, ending with a contact
    address

Versioning and Authors carry no umbrella content, because the indexed repositories tag their own
releases and the attribution block in the header already names the authors.

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

### Prose punctuation and positive description

Prose uses only the full stop and the comma to separate clauses. Do not use a semicolon or an
em-dash (`--`, `—`, or `–`) as a separator, and use a colon only where it is lexically appropriate.
A single hyphen stays available as a list marker, in tables, and in compound words. State what the
library does and what is currently true. Do not frame it by what it is not or what it used to be,
and keep a "not Y" contrast only when it is load-bearing because it corrects a counter-intuitive
assumption, giving its reason.

### Content restraint

A README earns a reader's attention section by section, so every sentence must carry information
the reader is unable to obtain faster somewhere else. The three sources that outrank the README are
the code itself, the hosted API documentation, and the one-line description at the top of the file.

**The cover test**: Before keeping a sentence, cover it and try to reconstruct it from the project
name, the one-line description, and the section heading it sits under. A sentence you are able to
reconstruct carries no information, so delete it. Apply the test to one sentence at a time rather
than to the section as a whole.

**No API reproduction**: The README shows a reader what the library is for and how to start using
it. It does not reproduce the API surface. Do not list every parameter of a function, every field
of a configuration class, or every method of a public class, because the hosted documentation
generates all of it from the docstrings and stays correct as the code changes. Link to the API
documentation instead and keep the README's usage material to the smallest path that gets a reader
to a working first result.

**No marketing prose**: State what the library does. Do not describe it as powerful, flexible,
seamless, robust, comprehensive, or easy to use, and do not explain why its problem matters. A
reader who opened the README has already accepted the premise.

**No section preamble**: A section starts with its content. Sentences that announce the section,
such as "This section describes the installation process", restate the heading directly above them.

**No change narration**: The README describes the library as it currently stands. Version history,
migration notes, and the record of what changed belong to the release notes, so do not accumulate
them here.

**No ratchet**: Updating a README section is not a reason to lengthen it. When a change leaves the
documented behavior intact, leave the section as it stands. When behavior changes, rewrite the
affected sentences and delete the ones the change made redundant.

**Worked reduction:**

```markdown
<!-- Avoid -->
## Detailed Description

This section provides a detailed description of the library. This powerful and flexible library
provides a comprehensive and easy-to-use solution for acquiring video data. Video acquisition is an
important part of any behavioral experiment, and getting it right is difficult. The library exposes
a VideoSystem class, which accepts a camera index, a frame rate, a resolution tuple, an output
directory, and a codec name, and which provides the start(), stop(), and snapshot() methods.

<!-- Good -->
## Detailed Description

This library acquires, encodes, and saves video data from multiple cameras with hardware-accurate
frame timestamps. It exposes the VideoSystem class as its entry point. See the
[API documentation](https://example.readthedocs.io/) for the full interface.
```

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

When creating a new project, proactively offer to generate a README following these
conventions. When modifying code that affects documented behavior (API changes, new features,
removed functionality), proactively suggest updating the README to reflect the changes.

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to README
files.**

```text
README Style Compliance:

Structure:
- [ ] Archetype selected mechanically: library (ships an installable artifact) or umbrella (indexes siblings)
- [ ] Section order matches the selected archetype's list, with only its optional sections omitted
- [ ] Title H1 uses the package name lowercase-hyphenated (library) or the display casing (umbrella)
- [ ] Line after the title is the bare canonical description (library) or the bold tagline (umbrella)
- [ ] Correct badge set for project type, umbrella READMEs carrying the license badge alone
- [ ] Blank line between description and badges
- [ ] Horizontal rule uses `___` (not `---`) after badges
- [ ] Detailed Description heading present (library) or unheaded framework description present (umbrella)
- [ ] Features section ends with license type bullet (library, if Features present)
- [ ] Table of contents with lowercase Markdown anchors (library order only, umbrella READMEs omit it)
- [ ] Spelling: "Acknowledgments" (not "Acknowledgements") everywhere
- [ ] Heading hierarchy: single H1, H2 for sections, H3 for subsections, no skips
- [ ] Horizontal rules use `___` consistently throughout (not `---`)

Content:
- [ ] All sections required by the selected order are present (library Installation, umbrella Libraries)
- [ ] Section templates followed (see section-templates.md)
- [ ] Installation uses standard Source warning and pip/source subsections
- [ ] Dependencies uses standard boilerplate text
- [ ] API Documentation links to hosted docs
- [ ] Developers section uses standard mamba/tox template (if present)
- [ ] Standard ending sections use correct templates (Versioning, Authors, License, Acknowledgments)
- [ ] Umbrella READMEs omit Versioning and Authors, carrying attribution in the header block
- [ ] MCP Server section titled "MCP Server" with tools table (if applicable)
- [ ] CLI commands documented with overview table (if applicable)

Style:
- [ ] Third-person voice throughout (no "I", "we", "you")
- [ ] Present tense as default
- [ ] `***Note,***` / `***Warning!***` for callouts (not GitHub alerts)
- [ ] No GitHub-specific features (alerts, details/summary, picture, task lists)
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons and hyphen bullets fine)
- [ ] Prose states what the project does, not what it is not or used to be (contrast only when load-bearing)

Quality:
- [ ] Each retained sentence survives the cover test (unable to be reconstructed from name, description, and heading)
- [ ] API surface is linked rather than reproduced (no per-parameter or per-method listings)
- [ ] No marketing adjectives (powerful, flexible, seamless, robust, comprehensive, easy to use)
- [ ] No section preamble restating the heading above it
- [ ] No version history or migration notes (those belong to the release notes)
- [ ] Updates leave sections no longer than they started unless behavior genuinely changed
- [ ] All images have meaningful alt text
- [ ] Link text is descriptive (no "click here")
- [ ] Technical descriptions cross-referenced against codebase
- [ ] File paths and class names verified to exist
- [ ] API examples tested against actual implementation
```
