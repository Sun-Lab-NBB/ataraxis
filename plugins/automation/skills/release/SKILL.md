---
name: release
description: >-
  Drafts style-compliant release notes by summarizing the pull requests merged since the previous release. Infers and
  recommends the release type (major, minor, or patch) for the user to confirm, then produces a numbered list of the
  most impactful changes. Includes a sibling-library compatibility statement for sollertia releases only. Use when
  preparing a release, drafting release notes, or when the user invokes /release.
user-invocable: true
---

# Release

Drafts style-compliant release notes summarizing the pull requests merged since the previous release.

---

## Scope

**Covers:**
- Identifying the previous release and the pull requests merged since then
- Inferring and recommending the release type (major, minor, patch) for user confirmation
- Drafting release notes with a numbered list of the most impactful changes
- Including a sibling-library compatibility statement for sollertia releases

**Does not cover:**
- Creating tags, GitHub releases, or publishing (the user does this)
- The `## What's Changed` and `**Full Changelog**` sections (GitHub generates these automatically on release)
- Committing, pushing, or opening pull requests (see `/commit` and `/pr`)

---

## Workflow

You MUST follow these steps exactly when this skill is invoked.

### Step 1: Gather context

Run the following git commands using the Bash tool:

1. Identify the previous release tag with `git tag --sort=-v:refname` (most recent first) or
   `git describe --tags --abbrev=0`.
2. List the pull requests merged since that tag with `git log <previous-tag>..HEAD --merges --oneline`. Each merge
   commit names the pull request number and source branch, and its body carries the pull request title.
3. Review the aggregate change scope with `git diff <previous-tag>..HEAD --stat`.

If the repository has no prior release tag, summarize the full history (`git log --merges --oneline`) and note that
this is the first release.

### Step 2: Determine the platform

Determine whether the project is an ataraxis library or a sollertia library from its package or repository name:

- **sollertia-*** — the sollertia platform ships as a unified set of cross-dependent libraries, so the release notes
  MUST include a compatibility statement (Step 4).
- **ataraxis-*** — the ataraxis libraries are independent and have no enforced cross-dependency web, so the release
  notes MUST NOT include a compatibility statement.

### Step 3: Infer and recommend the release type

Infer the release type from the change set and recommend it to the user. You MUST ask the user to confirm the type
before finalizing the notes — never decide it unilaterally.

| Type  | Use case                                                                                 |
|-------|------------------------------------------------------------------------------------------|
| Major | Breaking API changes, removals, re-licensing, or migrations requiring downstream updates |
| Minor | New, backward-compatible features or capabilities                                        |
| Patch | Bug fixes and minor corrections with no new features                                     |

### Step 4: Resolve the compatibility statement (sollertia only)

For sollertia releases, the compatibility statement names the sibling-library versions this release is designed to
work with. You cannot reliably infer these — ask the user to confirm the compatible sibling-library versions and
include them verbatim. For ataraxis releases, skip this step entirely.

### Step 5: Draft the release notes

Produce the release notes following the format below. Present the draft to the user. The user creates the tag and
GitHub release manually.

---

## Format

```text
### {Major|Minor|Patch} Release

[sollertia only] This release is designed to work with <sibling-library> vN, <sibling-library> vM, ...

**Major Changes:**

1. <Most impactful change, past tense, descriptive sentence.>
2. <Next change.>
```

### Rules

- The first line is `### {type} Release` using the user-confirmed type.
- For sollertia releases, follow the header with a single compatibility sentence naming the confirmed sibling-library
  versions. Omit this line entirely for ataraxis releases.
- `**Major Changes:**` introduces a numbered list (`1.`, `2.`, …) of the most impactful changes, ordered from most to
  least impactful.
- Each item is a complete, descriptive sentence in past tense (see the verb set in `/commit`) ending with a period.
  Items may span multiple lines.
- Condense many pull requests into a few impactful themes. Do NOT list every pull request.
- Do NOT include a `## What's Changed` section or a `**Full Changelog**` line. GitHub generates these automatically
  when the release is published.

### Examples

**Good (ataraxis — no compatibility statement):**

```text
### Major Release

**Major Changes:**

1. Re-licensed the library to Apache 2.0 from GPL 3.
2. Added MCP assets and a companion plugin that enables AI agent interfacing with all library components.
3. Added a log processing (data extraction) pipeline.
```

**Good (sollertia — compatibility statement included):**

```text
### Minor Release

This release remains compatible with sollertia-experiment v5, sollertia-unity-tasks v3, and sollertia-forgery v1.

**Major Changes:**

1. Added shared project and animal data-hierarchy assets for walking the project tree without SessionData.
2. Introduced the data root as a fourth persistent host setting, with MCP tools and CLI commands to manage it.
```

---

## Content rules

**Changes only.** The release notes must describe ONLY the changes themselves. Nothing else belongs in them.

**Forbidden content:**
- Authorship details, co-author tags, or attribution lines
- References to tools, agents, or AI assistance unless the user explicitly requests it
- Metadata unrelated to the changes (timestamps, ticket numbers, etc. unless requested)
- Commentary on the process used to make the changes

---

## Related skills

| Skill               | Relationship                                                             |
|---------------------|--------------------------------------------------------------------------|
| `/commit`           | Provides the past tense verb set and punctuation conventions reused here |
| `/pr`               | Drafts the per-branch pull request summaries that releases aggregate     |
| `/explore-codebase` | Provides project context that helps write accurate summaries             |

---

## Proactive behavior

When the user signals that a release is imminent (a version bump, "preparing a release", or similar), proactively
offer to draft release notes. For example: "Would you like me to draft release notes for this version?"

Do NOT create tags or GitHub releases. Present the drafted notes for the user to use manually.

---

## Verification checklist

**You MUST verify the release notes against this checklist before presenting them to the user.**

```text
Release Notes Compliance:
- [ ] First line is `### {Major|Minor|Patch} Release` with the user-confirmed type
- [ ] Release type was recommended AND confirmed by the user
- [ ] Compatibility statement included for sollertia releases, omitted for ataraxis releases
- [ ] Sibling-library versions confirmed by the user (sollertia only)
- [ ] `**Major Changes:**` numbered list ordered from most to least impactful
- [ ] Each item is past tense and ends with a period
- [ ] Condenses many pull requests into a few impactful themes (does not list every pull request)
- [ ] Does NOT include `## What's Changed` or `**Full Changelog**`
- [ ] Contains NO authorship details, co-author tags, or attribution
- [ ] Contains NO references to tools or AI unless explicitly requested by the user
- [ ] Did NOT create a tag or GitHub release
```
