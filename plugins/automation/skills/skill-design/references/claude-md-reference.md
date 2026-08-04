# CLAUDE.md reference

Complete reference for structuring and maintaining CLAUDE.md project instruction files.

---

## Section ordering

CLAUDE.md files use the following section order. Omit sections that do not apply.

1. **Title**: `# Claude Code Instructions`
2. **Session start behavior**: What the agent should do at session start
3. **Style guide compliance**: Required style conventions and skill references
4. **Cross-referenced library verification**: Dependencies and version checking
5. **Available skills**: Table of project skills with descriptions
6. **MCP server**: MCP server documentation
7. **Downstream library integration**: Related libraries and coordination
8. **Companion library synchronization** *(optional)*: The paired library and what changes in lockstep
9. **Distribution model** *(optional)*: How the project's assets reach the user
10. **Project context**: Architecture, key areas, patterns, and standards

The headings above are the canonical spellings. Write `Style guide compliance` rather than `Style
guide requirements` and `MCP server` rather than `MCP server integration`, so that the same concern
carries one name across every project.

### The two conditional sections

Sections 8 and 9 apply to a subset of projects, and a project without the concern omits the section
rather than writing an empty one.

A **companion library synchronization** section belongs to a project that has a counterpart it must
stay in step with, such as a microcontroller library paired with its host-side interface. It names
the counterpart and states what must change alongside a change here, which is the part an agent
cannot infer from this repository alone.

A **distribution model** section belongs to a project that ships agent assets separately from its
package. It names the marketplace plugin that registers the project's MCP server and skills, so that
an agent asked to add a tool knows which repository publishes it.

---

## File import syntax

CLAUDE.md supports importing other files using the `@` syntax. Imports are resolved recursively
up to 5 levels deep.

```markdown
@path/to/other-file.md
```

**Rules:**
- The `@` must be at the start of a line
- The path is relative to the file containing the import
- Imported content replaces the `@` line at load time
- Maximum 5 levels of recursive imports

---

## Modular rules

The `.claude/rules/*.md` directory provides modular, topic-specific rules that are autoloaded with
the same priority as `.claude/CLAUDE.md`. Each rule file can optionally include frontmatter to
restrict its scope to specific file paths.

### Path-specific rules

```yaml
---
paths:
  - "src/**/*.py"
  - "tests/**/*.py"
---

# Python-specific rules

These rules apply only when working with Python files in the src/ or tests/ directories.
```

### When to use modular rules

| Approach             | When to use                                        |
|----------------------|----------------------------------------------------|
| CLAUDE.md inline     | Instructions that apply to all work in the project |
| `.claude/rules/*.md` | Topic-specific rules (e.g., Python, Docker, CI)    |
| Path-specific rules  | Rules scoped to specific directories or file types |

---

## Personal preferences

`.claude.local.md` (auto-gitignored) stores personal project-specific preferences that should not
be shared with the team:

```markdown
# Personal preferences

- Use vim keybindings in examples
- Prefer verbose output for debugging
- Run tests with --verbose flag
```

---

## Quality criteria

A CLAUDE.md complies when it passes the CLAUDE.md checklist in `/skill-design`. Beyond that
checklist, the default is that every instruction names a command to run or a concrete action to
take, and an instruction that restates a language default is deleted rather than kept.

---

## Content guidelines

**Include:**
- Bash commands the agent cannot guess from code inspection alone
- Code style rules that differ from language defaults
- Testing instructions and commands
- Repository etiquette (branching, PR conventions)
- Architectural decisions specific to the project
- Common gotchas and workarounds

**Exclude:**
- Standard language conventions the agent already knows
- Detailed API documentation (link to external docs instead)
- Information that changes frequently (use `@` imports to live sources)
- File-by-file descriptions of the codebase (let exploration discover these)
- Long explanations or tutorials

---

## Formatting rules

CLAUDE.md follows the same formatting conventions as skill files with these differences:

- **Section separators**: Use `##` headings without horizontal rules between sections
- All other conventions (line length, tables, code blocks, voice) match SKILL.md

---

## AGENTS.md

`AGENTS.md` is the vendor-neutral name for the project instruction file, read by agent tools that do not
load `CLAUDE.md`. No repository in the ataraxis or sollertia libraries carries one today, so there is no
local variant to reconcile. A project that adds one writes it as a CLAUDE.md under the other name, and
the following rules carry over unchanged:

- The section ordering above, with the canonical heading spellings and the two conditional sections
- The quality criteria and the include and exclude content guidelines
- `##` headings without horizontal rules between sections
- The 120 character line limit, pretty table formatting, and language identifiers on code blocks
- Third person for descriptive content, and second person with emphasis for agent directives
- Sentence case for section headings
- Prose punctuation and positive description. The full stop and the comma are the only clause separators,
  with no semicolon or em-dash. The single hyphen stays available as a list marker and in compound words,
  and the prose states what is currently true rather than what it is not or used to be

The CLAUDE.md verification checklist in `/skill-design` applies to `AGENTS.md` unchanged. The `@` import
syntax and the `.claude/rules/*.md` files are Claude Code loading mechanisms, so a tool that reads
`AGENTS.md` alone resolves neither, and content that tool needs belongs in the file itself.
