---
name: commit
description: >-
  Stages all local changes and creates a style-compliant git commit, stopping before any push. Drafts the commit
  message by analyzing all changes relative to the active branch, stages every change, and commits — leaving the push
  for the user. Use when the user asks to commit, when completing a coding task that should be committed, or when the
  user invokes /commit. Proactively offer to commit after completing substantial code changes.
user-invocable: true
---

# Commit

Stages all local changes and creates a style-compliant commit, stopping before push.

---

## Scope

**Covers:**
- Analyzing local git changes (staged, unstaged, and untracked files)
- Drafting commit messages that comply with conventions
- Creating a working branch when committing from the default branch (with user confirmation)
- Staging all changes and creating the commit
- Reporting the ready-to-run push command for the user to run when ready

**Does not cover:**
- Pushing to remote repositories (the user runs the push)
- Creating pull requests (see `/pr`)
- Drafting release notes (see `/release`)

---

## Workflow

You MUST follow these steps exactly when this skill is invoked.

### Step 1: Gather context

Run the following git commands in parallel using the Bash tool:

1. `git status` to see all changed, staged, and untracked files. Never use the `-uall` flag.
2. `git diff` to see unstaged changes and `git diff --cached` to see staged changes.
3. `git log --oneline -10` to see recent commit messages for style context.
4. `git branch --show-current` to determine the active branch, and identify the default branch (commonly `main` or
   `master`; confirm via `git symbolic-ref --short refs/remotes/origin/HEAD` and strip the `origin/` prefix when a
   remote exists).

If `git status` shows no staged, unstaged, or untracked changes, stop and report that there is nothing to commit
rather than running `git add`/`git commit`.

### Step 2: Analyze changes

Review every changed file and understand:

- **What** was changed (new features, bug fixes, refactors, removals, updates)
- **Why** the change was made (the purpose, not the mechanics)
- Whether the changes represent a single focused change or bundled related modifications

Do NOT read files that are not part of the changes unless absolutely necessary to understand the purpose of a change.

### Step 3: Draft the commit message

Generate a commit message following the style rules below. This message will be applied to the commit created in the
following steps.

### Step 4: Resolve the target branch

Using the branch information from Step 1:

- If the active branch is NOT the default branch, commit onto the active branch as-is.
- If the active branch IS the default branch, you MUST ask the user whether to create a new branch before committing.
  Recommend creating one (default to yes), but do NOT proceed until the user confirms. If they confirm, create and
  switch to a descriptively named branch with `git switch -c <branch-name>` (e.g., `feature/...`, `bugfix/...`)
  derived from the change. If they decline, commit directly onto the default branch.

### Step 5: Stage all changes

Stage every change — tracked modifications, deletions, and untracked files — with `git add -A`.

### Step 6: Create the commit

Commit the staged changes using the drafted message. To preserve exact formatting (including the blank line after the
header and the `-- ` bullets), pass the message via standard input:

```bash
git commit -F - <<'EOF'
<header line>

-- <detail bullet>
-- <detail bullet>
EOF
```

For a single-line commit, include only the header line. NEVER append authorship, co-author, or attribution trailers to
the commit message (see the content rules below).

### Step 7: Hand off for push

Do NOT push. Report the commit you created and surface the exact command the user can run when they decide to push:

```bash
git push -u origin <branch-name>
```

Stop there. Pushing is the supervising user's decision.

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

Commit-message prose is exempt from the project-wide separator rule, so it may use `--` and `-` bullet lists, for
example when referencing CLI flags or listing changes. Still prefer a positive description of what changed.

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

| Input (What was done)                              | Output (Commit message)                                      |
|----------------------------------------------------|--------------------------------------------------------------|
| Added user authentication with JWT tokens          | Added JWT-based authentication for user sessions.            |
| Fixed bug where dates displayed incorrectly        | Fixed date formatting in timezone conversion.                |
| Updated dependencies and refactored error handling | Updated dependencies and standardized error response format. |
| Removed deprecated API endpoints                   | Removed deprecated v1 API endpoints from configuration.      |
| Refactored the zone detection logic for clarity    | Refactored zone detection logic to improve readability.      |

---

## Common mistakes

| Wrong                             | Correct                                   | Issue                      |
|-----------------------------------|-------------------------------------------|----------------------------|
| `fixed bug`                       | `Fixed null reference in zone detection.` | Too vague, no punctuation  |
| `Updated stuff`                   | `Updated MQTT topic names to match spec.` | Not specific               |
| `Changes to Task.cs`              | `Added corridor reset logic to Task.`     | Describes file, not change |
| `WIP`                             | `Added initial zone boundary detection.`  | Not descriptive            |
| `Add new feature`                 | `Added new feature.`                      | Present tense, no period   |
| `This commit fixes the login bug` | `Fixed login validation error.`           | Unnecessary preamble       |
| `Fixed bug (Co-Authored-By: ...)` | `Fixed login validation error.`           | Authorship in message      |

---

## Related skills

| Skill               | Relationship                                                       |
|---------------------|--------------------------------------------------------------------|
| `/python-style`     | Provides Python conventions; invoke before making Python changes   |
| `/cpp-style`        | Provides C++ conventions; invoke before making C++ changes         |
| `/csharp-style`     | Provides C# conventions; invoke before making C# changes           |
| `/pr`               | Drafts a pull request summary for the branch after it is committed |
| `/release`          | Drafts release notes summarizing merged pull requests              |
| `/explore-codebase` | Provides project context that helps write accurate commit messages |

---

## Proactive behavior

After completing substantial code changes (new features, bug fixes, refactors), proactively offer to commit. For
example: "Would you like me to stage and commit these changes?"

Stage and commit when invoked, but NEVER push and never offer to push automatically. Always leave the push for the
user to perform.

---

## Verification checklist

**You MUST verify the commit message against this checklist before creating the commit.**

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

**You MUST verify the commit operation against this checklist before handing off.**

```text
Commit Execution Compliance:
- [ ] Determined the active branch and the default branch
- [ ] If on the default branch, asked the user before creating a new branch
- [ ] Staged ALL changes with `git add -A`
- [ ] Created the commit with the drafted, style-compliant message
- [ ] Did NOT push and did NOT offer to push automatically
- [ ] Surfaced the ready-to-run `git push -u origin <branch>` command
```
