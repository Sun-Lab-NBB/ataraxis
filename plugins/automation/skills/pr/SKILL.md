---
name: pr
description: >-
  Drafts a style-compliant pull request summary by analyzing all changes on the active branch relative to the default
  branch. Produces a concise, bulleted summary of the most impactful changes for the user to paste when opening the pull
  request, and a title as well when the branch carries more than one commit. Use when the user is about to open a pull
  request, asks for a PR summary, or invokes /pr.
user-invocable: true
---

# Pull request

Drafts a style-compliant pull request summary for the active branch.

---

## Scope

**Covers:**
- Analyzing all changes on the active branch relative to the default branch
- Drafting a concise summary of the most impactful changes for the pull request body
- Offering a pull request title when the branch carries more than one commit
- Presenting the draft for the user to open the pull request manually

**Does not cover:**
- Drafting a title for a single-commit branch (see Step 3)
- Creating, opening, or merging pull requests (the user does this)
- Committing or pushing changes (see `/commit`)
- Drafting release notes (see `/release`)

---

## Workflow

You MUST follow these steps exactly when this skill is invoked.

### Step 1: Gather context

Run the following git commands in parallel using the Bash tool:

1. Determine the active branch with `git branch --show-current` and the default branch, which is commonly `main` or
   `master`. Confirm the default branch via `git symbolic-ref --short refs/remotes/origin/HEAD` and strip the `origin/`
   prefix when a remote exists. If that command errors with `is not a symbolic ref`, run `git remote set-head origin -a`
   to populate `origin/HEAD`, otherwise fall back to checking for `main` then `master`.
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

Produce the pull request summary following the format rules below. GitHub pre-fills a usable title only when the branch
carries exactly one commit, in which case it uses that commit's subject and no title draft is needed. When the branch
carries more than one commit, GitHub pre-fills the title from the branch name instead (for example
`refactor/project-auditing-optimization`). In that case, also draft a one-line title covering the branch as a whole,
following the title rules under Format rules. Present that title above the body. Present the draft to the user. The user
opens the pull request manually.

---

## Format rules

The summary is a concise bulleted list of the most impactful changes the branch introduces:

- One bullet per theme.
- Prefix each bullet with `-- `.
- Each bullet starts with a past tense verb (see the verb set in `/commit`) and ends with a period.
- Each bullet occupies one line, under the line-break rule `/commit` defines. GitHub soft-wraps the body, so a
  hand-wrapped bullet renders as a rigid block.
- Order bullets from most to least impactful.
- Summarize, and do NOT reproduce every commit. Bundle minor changes into a single bullet (e.g., `-- Fixed various
  documentation and code style inconsistencies.`).

**Title length limit**: The drafted title must be no longer than 72 characters, because a squash merge writes it into
the commit subject line where the same display constraint applies. Summary bullets carry no length cap, since they
render as wrapped markdown rather than a git log column.

**Title punctuation and tense**: The drafted title starts with a past tense verb from the verb set in `/commit` and ends
with a period, matching the commit header it becomes on a squash merge.

PR-body prose is exempt from the project-wide separator rule, so it may use `--` and `-` bullet lists, for example when
referencing CLI flags or listing changes. The past tense rule above applies to every bullet without exception.

**Example:**

```text
-- Added interfaces for reading and writing GenTL-compatible camera configuration.
-- Restructured the CLI command groups for clarity.
-- Added inline documentation to the MCP server module.
-- Fixed various documentation and code style inconsistencies.
```

---

## Content rules

The pull request title and body obey the content rules defined in `/commit`, plus the rules below.

**Title names the change**: The drafted title names what the branch changed, not the activity that produced it, such as
an audit, a review, or a ticket. A squash merge writes that title into the permanent commit subject line.

**What and why, not how**: Each bullet states *what* the branch changed and *why*, not *how*, and is specific and
descriptive rather than vague like "Updated various modules".

**Positive description**: State what the branch now does. Do not frame a bullet by what the code no longer does or by
how it used to behave, beyond the removal verb itself. Keep a "not Y" contrast only when it is load-bearing because it
corrects a counter-intuitive assumption, and give its reason.

**Sentence length**: Every sentence in the drafted title and summary bullets stays under 40 words, broken at a natural
clause boundary or split into two bullets when it runs longer.

**Typo-free and grammatical**: The drafted title and summary bullets must be free of typos and grammatical errors, with
every symbol name, file name, and flag spelling verified against the diff rather than recalled from memory.

---

## Related skills

| Skill               | Relationship                                                                               |
|---------------------|--------------------------------------------------------------------------------------------|
| `/commit`           | Provides the past tense verb set, `-- ` bullet style, and content rules the summary reuses |
| `/release`          | Drafts release notes that summarize merged pull requests                                   |
| `/explore-codebase` | Provides project context that helps write accurate summaries                               |

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
- [ ] Each bullet occupies one line, with no hand-wrapped continuation
- [ ] Each bullet starts with a past tense verb
- [ ] Bullets ordered from most to least impactful
- [ ] Drafted a title as well when the active branch carries more than one commit
- [ ] Drafted title ≤ 72 characters (a squash merge writes it into the commit subject line)
- [ ] Drafted title starts with a past tense verb and ends with a period
- [ ] Drafted title names the change rather than the activity that produced it (no audit, review, or ticket)
- [ ] Summarizes impactful changes; does not reproduce every commit
- [ ] Each bullet describes *what* changed and *why*, not *how*, and is specific rather than vague (not "Updated
      various modules")
- [ ] Bullets state what the change now does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Every sentence in the drafted text stays under 40 words
- [ ] Free of typos and grammar errors
- [ ] Compared the active branch against the default branch (three-dot diff)
- [ ] Contains NO authorship details, co-author tags, or attribution
- [ ] Contains NO references to tools or AI unless explicitly requested by the user
- [ ] Contains ONLY information about the changes themselves
- [ ] Did NOT create or open the pull request
```
