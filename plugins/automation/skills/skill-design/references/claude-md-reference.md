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
8. **Project context**: Architecture, key areas, patterns, and standards

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

| Approach             | When to use                                                  |
|----------------------|--------------------------------------------------------------|
| CLAUDE.md inline     | Instructions that apply to all work in the project           |
| `.claude/rules/*.md` | Topic-specific rules (e.g., Python, Docker, CI)              |
| Path-specific rules  | Rules scoped to specific directories or file types           |

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

Use these weighted criteria when evaluating CLAUDE.md quality:

| Criterion            | Weight | Description                                                 |
|----------------------|--------|-------------------------------------------------------------|
| Commands/workflows   | High   | Copy-paste ready commands for build, test, lint, deploy     |
| Architecture clarity | High   | Clear directory structure and component relationships       |
| Non-obvious patterns | Medium | Project-specific gotchas and conventions                    |
| Conciseness          | Medium | Dense, information-rich content without filler              |
| Currency             | High   | Up-to-date with current codebase state                      |
| Actionability        | High   | Every instruction leads to a concrete action                |

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
