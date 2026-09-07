---
name: commit
description: >-
  Stages all local changes and creates style-compliant git commits, stopping before any push. Splits a change set
  spanning several isolatable concerns into a chain of commits, one per concern, and creates a single commit when the
  user asks for one. Offers to commit proactively after completing substantial code changes. Use when the user asks to
  commit, when completing a coding task that should be committed, or when the user invokes /commit.
user-invocable: true
---

# Commit

Stages all local changes and creates a style-compliant commit, stopping before push.

---

## Scope

**Covers:**
- Analyzing local git changes (staged, unstaged, and untracked files)
- Splitting a change set into a chain of commits, one per isolatable concern
- Drafting commit messages that comply with conventions
- Creating a working branch when committing from the default branch (with user confirmation)
- Staging all changes and creating the commit
- Reporting the ready-to-run push command for the user to run when ready

**Does not cover:**
- Pushing to remote repositories (the user runs the push)
- Rewriting commits already pushed to a remote, because the chain is built forward from the working tree
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

If `git status` shows no staged, unstaged, or untracked changes, stop and report that there is nothing to commit.

### Step 2: Analyze changes

Review every changed file and understand:

- **What** was changed (new features, bug fixes, refactors, removals, updates)
- **Why** the change was made (the purpose, not the mechanics)
- Whether the changes represent a single focused change or bundled related modifications

Do NOT read files that are not part of the changes unless absolutely necessary to understand the purpose of a change.

### Step 3: Plan the chain

Split the change set into the commits it ships as, following the commit chains section below. A change set spanning
several isolatable concerns becomes a chain, and a change set carrying one concern becomes one commit. An explicit
instruction from the user settles the question before any of that, in either direction.

Record the plan as an ordered list of commits, each naming its files and its message, before staging anything. Report
the plan when it holds more than one commit, so the user sees the split before it lands.

### Step 4: Draft the commit messages

Generate one message per planned commit, following the style rules below.

### Step 5: Resolve the target branch

Using the branch information from Step 1:

- If the active branch is NOT the default branch, commit onto the active branch as-is.
- If the active branch IS the default branch, you MUST ask the user whether to create a new branch before committing.
  Recommend creating one (default to yes), but do NOT proceed until the user confirms. If they confirm, create and
  switch to a branch named under the rules below with `git switch -c <branch-name>`. If they decline, commit directly
  onto the default branch.

**The branch name states the change, not the activity.** An audit, a review, a sweep, a cleanup pass, or the skill that
ran is the activity, so `refactor/correctness-changes`, `bugfix/audit-findings`, and `refactor/style-cleanup` name none
of the code they touch. The test is mechanical: read the name alone and ask which files it predicts. A name predicting
no particular file is rewritten to name the surfaces the branch alters.

An audit is the recurring offender, because a feature carries its own subject while an audit carries only its own name.
Derive the subject from the files the change set touches. Build the name as `<type>/<subject>`, where the type is
`feature`, `bugfix`, `refactor`, or `docs`, and the subject names the altered surfaces in two to five hyphenated words.
Those four types are the whole set, so a name reaching for another one is rewritten to the closest of the four. When one
audit produced fixes across several surfaces, name those surfaces, as in `bugfix/serial-timeout-and-retry-handling`.
When they share no surface, name the subsystem holding them.

### Step 6: Stage all changes

Before staging anything, run `git status --porcelain -uall` and read every line marked `??`. The `-uall` flag matters,
because the default output collapses an untracked directory into a single entry and never names the files inside it. An
untracked file is the one way a working artifact enters the repository, and `git add -A` admits it silently.

For each untracked file, ask whether it occupies a slot the archetype tree defines, which is the test
`/project-layout` states. A new module under `src/`, a new test under `tests/`, and a new page under `docs/source/`
all pass, so stage them. A file that fits no slot fails, and a stray file at the repository root is the usual case.
When one fails, STOP. Do not stage it, do not commit it, and do not add it to `.gitignore` on your own initiative.
Report the file and ask what to do with it.

The untracked audit above runs once over the whole change set, before the first commit. Once every untracked file is
accounted for, stage the plan's first commit. A single commit stages with `git add -A`, covering tracked modifications,
deletions, and the untracked files that belong in the tree. A chain stages one commit at a time with
`git add -- <paths>`, naming that commit's files alone, which stages that pathspec's deletions along with its edits.
Read the staged set back with `git diff --cached --name-only` before committing.

### Step 7: Create the commits

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

For a chain, repeat Step 6 and this step per planned commit, in the planned order. Once the chain is complete, verify it
under the commit chains section below. `git status --porcelain` then prints nothing, because a leftover path means the
plan assigned that path to no commit. The exception is a path that halted Step 6, which stays uncommitted by design.

### Step 8: Hand off for push

Do NOT push. Report every commit you created and surface the exact command the user can run when they decide to push:

```bash
git push -u origin <branch-name>
```

Stop there. Pushing is the supervising user's decision.

---

## Commit chains

A change set spanning several isolatable concerns ships as a chain of commits, one per concern. An agent accumulates
many edits inside one task, so the default for a broad change set is the chain.

**The isolation test decides.** Cover a candidate commit and ask whether a reviewer accepts or reverts it alone, leaving
the rest of the chain standing. When yes, it is its own commit. When reverting it forces the reverting of another
candidate, the two are one commit. Apply the test to each candidate.

**The user overrides the default.** An explicit request for one commit produces one commit at any size, and an explicit
request for a chain produces a chain at any size. Settle this before planning, and do not re-litigate a stated choice.

### Grouping the commits

Group by AREA, meaning the subsystem or directory the files belong to. Change types such as a visibility narrowing, a
diagnostic rewrite, and a documentation pass interleave INSIDE a single file, so no hunk boundary separates a moved
method body from the documentation edits that body contains. An area slice takes whole files and stays reviewable, while
a type slice on interleaved edits produces hunks that belong to no clean group.

Group by change type only when each type occupies whole files that no other type touches.

**Every commit is dependency-closed.** A commit carries the files it changes AND every file whose build depends on those
changes, tests included. A narrowing that closes an access path travels with the callers it closes, and a declaration
that grants access lands in or before the commit that relies on it. Order the chain so each commit builds on its own.

### Verifying the chain

The build gate is the command the project already uses to prove its sources build. A Python package runs its `tox` test
environment, a PlatformIO project runs `pio check` and `pio test`, and a C# Unity project runs the Roslyn compile gate
`/csharp-style` documents. Ask the user which command serves as the gate when the project names none.

Run that gate at EVERY commit, because a chain that only builds at the end is a chain a bisect cannot walk. Check out
each commit into a detached worktree with `git worktree add --detach <path> <sha>`, run the gate there, and remove the
worktree. The working tree stays untouched throughout, so a failure costs a regrouping.

A commit failing the gate is not dependency-closed. Merge it into the adjacent commit supplying what it lacks and run
the gate again. Report the failure and stop when no regrouping clears it.

**State the verification level.** Report whether the chain is build-verified or test-verified, and never imply the
stronger one. A chain whose every commit compiles is build-verified, and it stays merely build-verified when a commit
carries a test assertion updated ahead of the source that assertion pins. Running a full suite per commit is what earns
the test-verified claim.

---

## Content rules

**Changes only.** The commit message must describe ONLY the changes themselves. Nothing else belongs in the message.

**Forbidden content:**
- Authorship details, co-author tags, or attribution lines (e.g., `Co-Authored-By`)
- References to tools, agents, or AI assistance unless the user explicitly requests it
- Metadata unrelated to the changes (timestamps, ticket numbers, etc. unless requested)
- Commentary on the process used to make the changes

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
commit is created, with every symbol name, file name, flag spelling, and version string verified against the diff.

**Sentence length**: Every sentence in the commit header and detail bullets stays under 40 words. Break a longer
sentence at a natural clause boundary, or split it into two bullets. Count the message prose alone, never the git
commands, paths, or code spans quoted inside it.

**Forbidden phrase**: The two-word phrase `rather` followed by `than` is FORBIDDEN in a commit header and in every
detail bullet, with no exception and no load-bearing carve-out. It names an alternative the reader never proposed, and
the sentence keeps its full meaning once the trailing clause is deleted, so delete the clause and state the positive
claim alone. Substituting `instead of`, `as opposed to`, or `in place of` reproduces the same padding under a new
spelling and is equally forbidden. Where the excluded option genuinely carries information, it earns its own sentence
naming what it costs.

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

Branch names carry the same rule, and an audit is where it breaks most often:

| Wrong                          | Correct                                       | Issue                    |
|--------------------------------|-----------------------------------------------|--------------------------|
| `refactor/correctness-changes` | `bugfix/encoder-overflow-and-timeout-guards`  | Names the activity       |
| `bugfix/audit-findings`        | `bugfix/pyproject-style-grayskull-references` | Names the activity       |
| `refactor/cleanup`             | `refactor/plugin-export-surface-sync`         | Predicts no file         |
| `bugfix/various-fixes`         | `bugfix/file-summary-length`                  | Bare outcome category    |
| `refactor/style-audit-pass`    | `refactor/csharp-compile-gate-and-layout`     | Names the skill that ran |

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
- [ ] Header and every bullet free of the phrase `rather` followed by `than` (forbidden with no exception)
- [ ] Multi-line format used for bundled changes (if applicable)
- [ ] Multi-line bullets prefixed with `-- ` and each ends with a period
- [ ] Every bullet occupies one line, so no line after the header begins with whitespace
- [ ] Header names the change itself (no audit, review, or ticket)
- [ ] Header names the change the commit itself carries, so no chain position and no running count appear
- [ ] Contains NO authorship details, co-author tags, or attribution
- [ ] Contains NO references to tools or AI unless explicitly requested by the user
- [ ] Contains ONLY information about the changes themselves
- [ ] Checked against this list once per commit, so a chain runs it for every message
```

### Commit execution

**You MUST verify the commit operation against this checklist before handing off.**

```text
Commit Execution Compliance:
- [ ] Ran `git status` without the `-uall` flag
- [ ] Reported nothing to commit and made no commit when `git status` showed no staged, unstaged, or untracked changes
- [ ] Determined the active branch and the default branch
- [ ] If on the default branch, asked the user before creating a new branch
- [ ] Branch name states the changed surfaces, so it names no audit, review, sweep, cleanup, or skill, and it
      predicts the files the branch touches
- [ ] Every untracked file accounted for before staging, with any file occupying no archetype slot reported
- [ ] Chain planned before staging, with each commit passing the isolation test, or a single commit justified by
      one concern or by an explicit user request
- [ ] Chain plan reported to the user before staging whenever it holds more than one commit
- [ ] Each commit dependency-closed, carrying every file whose build depends on its changes
- [ ] Build gate identified from what the project already runs, or asked of the user when the project names none
- [ ] Build gate run at EVERY commit in the chain through a detached worktree, not at the tip alone
- [ ] `git log --oneline <branch-point>..HEAD` lists exactly the planned commits, in the planned order
- [ ] Verification level reported as build-verified or test-verified, claiming the stronger one only when a full
      suite ran per commit
- [ ] Staged a single commit with `git add -A`, or each chain commit with `git add -- <paths>`
- [ ] Created every commit with its drafted, style-compliant message
- [ ] Did NOT push and did NOT offer to push automatically
- [ ] Surfaced the ready-to-run `git push -u origin <branch>` command
```
