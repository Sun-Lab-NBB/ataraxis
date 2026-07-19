---
name: pr
description: >-
  Drafts a style-compliant pull request summary by analyzing all changes on the active branch relative to the default
  branch. Produces a concise, bulleted summary of the most impactful changes for the user to paste when opening the
  pull request. GitHub pre-fills the title from the branch's commits, so this skill drafts only the body. Use when the
  user is about to open a pull request, asks for a PR summary, or invokes /pr.
user-invocable: true
---

# Pull request

Drafts a style-compliant pull request summary for the active branch.

---

## Scope

**Covers:**
- Analyzing all changes on the active branch relative to the default branch
- Drafting a concise summary of the most impactful changes for the pull request body
- Presenting the draft for the user to open the pull request manually

**Does not cover:**
- Drafting the pull request title (GitHub pre-fills it from the branch's commits)
- Creating, opening, or merging pull requests (the user does this)
- Committing or pushing changes (see `/commit`)
- Drafting release notes (see `/release`)

---

## Workflow

You MUST follow these steps exactly when this skill is invoked.

### Step 1: Gather context

Run the following git commands in parallel using the Bash tool:

1. Determine the active branch with `git branch --show-current` and the default branch (commonly `main` or
   `master`; confirm via `git symbolic-ref --short refs/remotes/origin/HEAD` and strip the `origin/` prefix when a
   remote exists). If that command errors with `is not a symbolic ref`, run `git remote set-head origin -a` to
   populate `origin/HEAD`, otherwise fall back to checking for `main` then `master`.
2. If the active branch IS the default branch, stop: no pull request can be drafted from the default branch (the
   `<default-branch>..HEAD` diff range is empty). Tell the user to switch to or create a feature branch first.
3. `git log <default-branch>..HEAD --oneline` to list the commits unique to the active branch.
4. `git diff <default-branch>...HEAD --stat` to see the overall scope of changed files (use the three-dot form to
   compare against the merge base).

### Step 2: Analyze the branch

Review the commit list and the aggregate diff to understand the most impactful changes the branch introduces. Group
related commits into themes. Focus on what reviewers need to know: new capabilities, behavior changes, removals, and
notable fixes. Inspect individual file diffs only when the commit history is insufficient to understand a change's
purpose.

### Step 3: Draft the summary

Produce the pull request summary following the format rules below. GitHub pre-fills the title from the branch's
commits, so draft only the summary body. Present the draft to the user. The user opens the pull request manually.

---

## Format rules

The summary is a concise bulleted list of the most impactful changes the branch introduces:

- One bullet per theme.
- Prefix each bullet with `-- `.
- Each bullet starts with a past tense verb (see the verb set in `/commit`) and ends with a period.
- Order bullets from most to least impactful.
- Summarize; do NOT reproduce every commit. Bundle minor changes into a single bullet
  (e.g., `-- Fixed various documentation and code style inconsistencies.`).

PR-body prose is exempt from the project-wide separator rule and may use `--` and `-` bullet lists, for example
when referencing CLI flags or listing changes, while still preferring positive, present-tense description of
what changed.

**Example:**

```text
-- Added interfaces for reading and writing GenTL-compatible camera configuration.
-- Restructured the CLI command groups for clarity.
-- Added inline documentation to the MCP server module.
-- Fixed various documentation and code style inconsistencies.
```

---

## Content rules

**Changes only.** The summary must describe ONLY the changes themselves. Nothing else belongs in it.

**Forbidden content:**
- Authorship details, co-author tags, or attribution lines
- References to tools, agents, or AI assistance unless the user explicitly requests it
- Metadata unrelated to the changes (timestamps, ticket numbers, etc. unless requested)
- Commentary on the process used to make the changes

---

## Related skills

| Skill               | Relationship                                                               |
|---------------------|----------------------------------------------------------------------------|
| `/commit`           | Provides the past tense verb set and `-- ` bullet style the summary reuses |
| `/release`          | Drafts release notes that summarize merged pull requests                   |
| `/explore-codebase` | Provides project context that helps write accurate summaries               |

---

## Proactive behavior

After all changes on a feature branch are committed, proactively offer to draft a pull request summary. For example:
"Would you like me to draft a pull request summary for this branch?"

Do NOT create or open the pull request. Present the drafted summary for the user to use manually.

---

## Verification checklist

**You MUST verify the pull request draft against this checklist before presenting it to the user.**

```text
Pull Request Compliance:
- [ ] Summary bullets prefixed with `-- `, each ending with a period
- [ ] Each bullet starts with a past tense verb
- [ ] Bullets ordered from most to least impactful
- [ ] Summarizes impactful changes; does not reproduce every commit
- [ ] Compared the active branch against the default branch (three-dot diff)
- [ ] Contains NO authorship details, co-author tags, or attribution
- [ ] Contains NO references to tools or AI unless explicitly requested by the user
- [ ] Contains ONLY information about the changes themselves
- [ ] Did NOT create or open the pull request
```
