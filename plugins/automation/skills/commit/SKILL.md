---
name: commit
description: >-
  Stages all local changes and creates a style-compliant git commit, stopping before any push. Drafts the commit message
  by analyzing all changes relative to the active branch, stages every change, and commits, leaving the push for the
  user. Offers to commit proactively after completing substantial code changes. Use when the user asks to commit, when
  completing a coding task that should be committed, or when the user invokes /commit.
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
   `master`). Confirm the default branch via `git symbolic-ref --short refs/remotes/origin/HEAD` and strip the `origin/`
   prefix when a remote exists. If that command errors with `is not a symbolic ref`, run `git remote set-head origin -a`
   to populate `origin/HEAD`, otherwise fall back to checking for `main` then `master`.

If `git status` shows no staged, unstaged, or untracked changes, stop and report that there is nothing to commit rather
than running `git add`/`git commit`.

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
  switch to a descriptively named branch with `git switch -c <branch-name>` (e.g., `feature/...`, `bugfix/...`) derived
  from the change. If they decline, commit directly onto the default branch.

### Step 5: Stage all changes

Stage every change with `git add -A`, covering tracked modifications, deletions, and untracked files.

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

**The header names the change, not its origin.** Cover the header and ask what it tells a reader about the code. A
header naming the activity that produced the change, such as an audit, a review, a ticket, or a user request, describes
the process and is rewritten to name the change instead. A reader running `git log` a year later has no access to that
activity, so naming it spends the one line they do read. When the change set is a group of unrelated fixes, name the
areas they touch, as in `Fixed various bugs in environment, credential, and upload handling.`

---

## Style rules

### Format

**Header line limit**: The first line (header) must be no longer than 72 characters. This ensures proper display in Git
logs, GitHub, and other tools.

**Single-line commits**: Use for focused, single-purpose changes.

```text
Added Python 3.14 support.
Fixed a bug that allowed valves to violate keepalive guard.
Optimized the behavior of camera ID discovery functionality.
```

**Multi-line commits**: Use for changes that bundle related modifications. Insert a blank line after the header, then
prefix each detail bullet with `-- `.

```text
Added MCP server module for agentic library interaction.

-- Added mcp_server.py exposing camera discovery and video session management.
-- Added 'axvs mcp' CLI command to start the MCP server.
-- Added frame display support to MCP video sessions.
-- Fixed various documentation and code style inconsistencies.
```

### Line breaks

Each bullet occupies exactly one line, however long that line runs. Git, GitHub, and terminal pagers soft-wrap the text
themselves, so a hand-wrapped bullet renders as a rigid block that reflows badly at every other width. The test is
mechanical: in the message source, no line after the header begins with whitespace. Bullets carry no character cap, and
one running past roughly 30 words is usually two changes that split into two bullets.

```text
Updated the environment export to write through a temporary file.

-- Replaced the shell pipeline with a captured mamba call, so a failed export leaves the committed file intact.
-- Added a dependency check to the export, so a contentless specification cannot overwrite the stored pins.
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
example when referencing CLI flags or listing changes.

State what the commit now does, and do not frame a bullet by what the code no longer does or by how it used to behave
beyond the removal verb itself. Keep a "not Y" contrast only when it is load-bearing because it corrects a
counter-intuitive assumption, giving its reason.

### Prose quality

**Typo-free and grammatical**: The commit message must be free of typos and grammatical errors, checked before the
commit is created, with every symbol name, file name, flag spelling, and version string verified against the diff rather
than from memory.

**Sentence length**: Every sentence in the commit header and detail bullets stays under 40 words. Break a longer
sentence at a natural clause boundary, or split it into two bullets. Count the message prose alone, never the git
commands, paths, or code spans quoted inside it.

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
| `Fixed the audit findings`        | `Fixed various environment bugs.`         | Names activity, not change |

---

## Related skills

| Skill               | Relationship                                                       |
|---------------------|--------------------------------------------------------------------|
| `/python-style`     | Provides Python conventions, invoked before making Python changes  |
| `/cpp-style`        | Provides C++ conventions, invoked before making C++ changes        |
| `/csharp-style`     | Provides C# conventions, invoked before making C# changes          |
| `/audit-project`    | Runs the change-mode gate over new code, before this skill commits |
| `/pr`               | Drafts a pull request summary for the branch after it is committed |
| `/release`          | Drafts release notes summarizing merged pull requests              |
| `/explore-codebase` | Provides project context that helps write accurate commit messages |

---

## Proactive behavior

After completing substantial code changes (new features, bug fixes, refactors), proactively offer to commit. For
example: "Would you like me to stage and commit these changes?"

Stage and commit when invoked, but NEVER push and never offer to push automatically. Always leave the push for the user
to perform.

---

## Verification checklist

### Commit message

**You MUST verify the commit message against this checklist before creating the commit.**

```text
Commit Message Compliance:
- [ ] Starts with past tense verb (Added, Fixed, Updated, Refactored, Removed, etc.)
- [ ] Header line ≤ 72 characters
- [ ] Ends with a period
- [ ] Describes *what* was changed and *why*, not *how*
- [ ] Specific and descriptive (not vague like "Updated stuff")
- [ ] Free of typos and grammar errors
- [ ] Every sentence in the drafted text stays under 40 words
- [ ] Bullets state what the change now does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Multi-line format used for bundled changes (if applicable)
- [ ] Multi-line bullets prefixed with `-- ` and each ends with a period
- [ ] Every bullet occupies one line, so no line after the header begins with whitespace
- [ ] Header names the change rather than the activity that produced it (no audit, review, or ticket)
- [ ] Contains NO authorship details, co-author tags, or attribution
- [ ] Contains NO references to tools or AI unless explicitly requested by the user
- [ ] Contains ONLY information about the changes themselves
```

### Commit execution

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
