---
name: commit
description: >-
  Drafts Sun Lab style-compliant git commit messages by analyzing all local changes relative to the active branch.
  Use when the user asks to commit, when completing a coding task that should be committed, or when the user invokes
  /commit. Proactively offer to draft a commit message after completing substantial code changes.
user-invocable: true
---

# Commit

Drafts a style-compliant commit message by analyzing all local changes.

---

## Scope

**Covers:**
- Analyzing local git changes (staged, unstaged, and untracked files)
- Drafting commit messages that comply with Sun Lab conventions
- Presenting the draft for the user to commit manually

**Does not cover:**
- Staging or committing files (the user handles this)
- Pushing to remote repositories
- Creating pull requests or managing branches

---

## Workflow

You MUST follow these steps exactly when this skill is invoked.

### Step 1: Gather context

Run the following git commands in parallel using the Bash tool:

1. `git status` to see all changed, staged, and untracked files. Never use the `-uall` flag.
2. `git diff` to see unstaged changes and `git diff --cached` to see staged changes.
3. `git log --oneline -10` to see recent commit messages for style context.

### Step 2: Analyze changes

Review every changed file and understand:

- **What** was changed (new features, bug fixes, refactors, removals, updates)
- **Why** the change was made (the purpose, not the mechanics)
- Whether the changes represent a single focused change or bundled related modifications

Do NOT read files that are not part of the changes unless absolutely necessary to understand the purpose of a change.

### Step 3: Draft the commit message

Generate a commit message following the style rules below. Present the draft to the user. The user will stage and
commit manually.

---

## Content rules

**Changes only.** The commit message must describe ONLY the changes themselves. Nothing else belongs in the message.

**Forbidden content:**
- Authorship details, co-author tags, or attribution lines (e.g., `Co-Authored-By`)
- References to tools, agents, or AI assistance unless the user explicitly requests it
- Metadata unrelated to the changes (timestamps, ticket numbers, etc. unless requested)
- Commentary on the process used to make the changes

The message is a record of *what changed in the code*, not *how or by whom the changes were produced*.

---

## Style rules

### Format

**Header line limit**: The first line (header) must be no longer than 72 characters. This ensures proper display in
Git logs, GitHub, and other tools.

**Single-line commits**: Use for focused, single-purpose changes.

```text
Added Python 3.14 support.
Fixed a bug that allowed valves to violate keepalive guard.
Optimized the behavior of camera ID discovery functionality.
```

**Multi-line commits**: Use for changes that bundle related modifications. Insert a blank line after the header,
then prefix each detail bullet with `-- `.

```text
Added MCP server module for agentic library interaction.

-- Added mcp_server.py exposing camera discovery and video session management.
-- Added 'axvs mcp' CLI command to start the MCP server.
-- Added frame display support to MCP video sessions.
-- Fixed various documentation and code style inconsistencies.
```

### Verb tense

Start with a past tense verb:

| Verb       | Use case                                    |
|------------|---------------------------------------------|
| Added      | New features, files, or functionality       |
| Fixed      | Bug fixes and error corrections             |
| Updated    | Modifications to existing functionality     |
| Refactored | Code restructuring without behavior changes |
| Optimized  | Performance improvements                    |
| Improved   | Enhancements to existing features           |
| Removed    | Deletions of code, files, or features       |
| Deprecated | Marking functionality for future removal    |
| Prepared   | Release preparation tasks                   |
| Finalized  | Completing a feature or release             |

### Punctuation

Always end commit messages (header and every bullet) with a period.

### Content focus

Focus on *what* was changed and *why*, not *how*. Be specific and descriptive.

---

## Examples

**Good commit messages:**

```text
Added trigger_type field to all task templates.
Fixed zone range calculation for occupancy zones.
Updated configuration-verification skill with cross-platform support.
Refactored style guide into separate domain-specific files.
Removed deprecated API endpoints from configuration loader.
```

**Good multi-line commit:**

```text
Refactored skill architecture to support user-invocable skills.

-- Extracted commit style guide into a dedicated /commit skill.
-- Updated python-style skill to reference /commit for commit conventions.
-- Added the /commit skill to CLAUDE.md available skills table.
```

**Avoid:**

```text
fixed bug                          # Too vague, no punctuation
Updated stuff                      # Not specific
Changes to Task.cs                 # Describes file, not change
WIP                                # Not descriptive
Add new feature                    # Present tense, no period
This commit fixes the login bug    # Unnecessary preamble
Co-Authored-By: ...                # Authorship does not belong in messages
Generated with AI assistance       # Tool attribution does not belong
```

---

## Input/output examples

| Input (What was done)                               | Output (Commit message)                                      |
|-----------------------------------------------------|--------------------------------------------------------------|
| Added user authentication with JWT tokens           | Added JWT-based authentication for user sessions.            |
| Fixed bug where dates displayed incorrectly         | Fixed date formatting in timezone conversion.                |
| Updated dependencies and refactored error handling  | Updated dependencies and standardized error response format. |
| Removed deprecated API endpoints                    | Removed deprecated v1 API endpoints from configuration.      |
| Refactored the zone detection logic for clarity     | Refactored zone detection logic to improve readability.      |

---

## Common mistakes

| Wrong                             | Correct                                   | Issue                       |
|-----------------------------------|-------------------------------------------|-----------------------------|
| `fixed bug`                       | `Fixed null reference in zone detection.` | Too vague, no punctuation   |
| `Updated stuff`                   | `Updated MQTT topic names to match spec.` | Not specific                |
| `Changes to Task.cs`              | `Added corridor reset logic to Task.`     | Describes file, not change  |
| `WIP`                             | `Added initial zone boundary detection.`  | Not descriptive             |
| `Add new feature`                 | `Added new feature.`                      | Present tense, no period    |
| `This commit fixes the login bug` | `Fixed login validation error.`           | Unnecessary preamble        |
| `Fixed bug (Co-Authored-By: ...)` | `Fixed login validation error.`           | Authorship in message       |

---

## Related skills

| Skill               | Relationship                                                       |
|---------------------|--------------------------------------------------------------------|
| `/python-style`     | Provides Python conventions; invoke before making Python changes   |
| `/cpp-style`        | Provides C++ conventions; invoke before making C++ changes         |
| `/csharp-style`     | Provides C# conventions; invoke before making C# changes           |
| `/explore-codebase` | Provides project context that helps write accurate commit messages |

---

## Proactive behavior

After completing substantial code changes (new features, bug fixes, refactors), proactively offer to draft a commit
message. For example: "Would you like me to draft a commit message for these changes?"

Do NOT stage or commit files. Present the drafted message for the user to use manually.

---

## Verification checklist

**You MUST verify the commit message against this checklist before presenting it to the user.**

```text
Commit Message Compliance:
- [ ] Starts with past tense verb (Added, Fixed, Updated, Refactored, Removed, etc.)
- [ ] Header line ≤ 72 characters
- [ ] Ends with a period
- [ ] Describes *what* was changed and *why*, not *how*
- [ ] Specific and descriptive (not vague like "Updated stuff")
- [ ] Multi-line format used for bundled changes (if applicable)
- [ ] Multi-line bullets prefixed with `-- ` and each ends with a period
- [ ] Contains NO authorship details, co-author tags, or attribution
- [ ] Contains NO references to tools or AI unless explicitly requested by the user
- [ ] Contains ONLY information about the changes themselves
```
