# Frontmatter reference

Complete reference for YAML frontmatter fields in SKILL.md files.

---

## Required fields

### `name`

The skill's identifier. You MUST ensure it matches the parent directory name.

| Property   | Value                                                            |
|------------|------------------------------------------------------------------|
| Type       | String                                                           |
| Required   | Yes                                                              |
| Max length | 64 characters                                                    |
| Format     | Lowercase letters, digits, and hyphens only, no consecutive `--` |
| Convention | Must match parent directory name: `explore-codebase`, `commit`   |

### `description`

Explains what the skill does and when to use it. Serves as the primary trigger mechanism, because the agent reads all
skill descriptions at session start to decide when to invoke each skill.

| Property     | Value                                                  |
|--------------|--------------------------------------------------------|
| Type         | String (use YAML `>-` for multi-line)                  |
| Required     | Strongly recommended                                   |
| Max length   | 1024 characters                                        |
| Max lines    | 5 wrapped lines in the folded block                    |
| Voice        | Third person                                           |
| Must include | What the skill does AND when to use it ("Use when...") |

**Example:**

```yaml
description: >-
  Drafts style-compliant git commit messages by analyzing all local changes relative to
  the active branch. Use when the user asks to commit, when completing a coding task that should
  be committed, or when the user invokes /commit.
```

---

## Optional fields

### `user-invocable`

Controls whether the skill appears in the `/` slash command menu.

| Property | Value                                                           |
|----------|-----------------------------------------------------------------|
| Type     | Boolean                                                         |
| Default  | `true`                                                          |
| Usage    | Set to `false` for skills that are only invoked by other skills |

### `disable-model-invocation`

Prevents the agent from autoloading the skill based on context. When `true`, the skill can only be invoked explicitly by
the user or by another skill.

| Property | Value                                                   |
|----------|---------------------------------------------------------|
| Type     | Boolean                                                 |
| Default  | `false`                                                 |
| Usage    | Set to `true` for skills that should never auto-trigger |

### `allowed-tools`

Pre-approves the listed tools (skips the permission prompt) while the skill is active. It does NOT restrict the tool
pool. Every other tool remains callable and normal permission settings still apply. To make a skill read-only, use
`disallowed-tools` or run it with `context: fork` plus `agent: Explore` (the Explore agent denies Write/Edit).

| Property | Value                                                    |
|----------|----------------------------------------------------------|
| Type     | String (space/comma-delimited) or YAML list              |
| Default  | No tools pre-approved (normal permissions apply)         |
| Example  | `Read Grep Glob` or `Bash(git add *) Bash(git commit *)` |

### `disallowed-tools`

Removes the listed tools from the agent's available pool while the skill is active. Use for autonomous or read-only
skills that must never call certain tools. The restriction clears on the next user message.

| Property | Value                                       |
|----------|---------------------------------------------|
| Type     | String (space/comma-delimited) or YAML list |
| Default  | No tools removed                            |
| Example  | `Write Edit` (read-only skill)              |

### `argument-hint`

Autocomplete hint displayed in the `/` menu to indicate expected arguments.

| Property | Value                                               |
|----------|-----------------------------------------------------|
| Type     | String                                              |
| Example  | `[issue-number]`, `[file-path]`, `[component-name]` |

### `model`

Overrides the model used while the skill is active.

| Property | Value                     |
|----------|---------------------------|
| Type     | String                    |
| Example  | `sonnet`, `haiku`, `opus` |

### `context`

Controls execution context. Set to `fork` to run the skill in a subagent with a separate context window, keeping the
main conversation context clean.

| Property | Value  |
|----------|--------|
| Type     | String |
| Values   | `fork` |

### `agent`

Specifies which subagent type to use when `context: fork` is set.

| Property | Value                           |
|----------|---------------------------------|
| Type     | String                          |
| Values   | `Explore`, `Plan`, `Bash`, etc. |

### `hooks`

Hooks scoped to the skill's lifecycle. Allows running shell commands when the skill is invoked or completed.

| Property | Value                                              |
|----------|----------------------------------------------------|
| Type     | Object                                             |
| Usage    | Advanced pattern for validation or post-processing |

---

## String substitution variables

SKILL.md content can include variables that are replaced at runtime.

| Variable               | Description                               |
|------------------------|-------------------------------------------|
| `$ARGUMENTS`           | All arguments passed after the skill name |
| `$ARGUMENTS[N]` / `$N` | Positional argument at index N (0-based)  |
| `${CLAUDE_SESSION_ID}` | Unique identifier for the current session |

### Dynamic context injection

Use `` !`command` `` syntax to inject the output of a shell command into the skill content at load time. This runs
during preprocessing before the skill content is presented to the agent.

**Example:**

```markdown
Current git branch: !`git branch --show-current`
```

---

## Naming constraints

| Constraint             | Rule                                                   |
|------------------------|--------------------------------------------------------|
| Character set          | Lowercase letters, digits, and hyphens only            |
| No consecutive hyphens | `my--skill` is invalid                                 |
| Must match directory   | Skill in `skills/my-skill/` must have `name: my-skill` |
| Max length             | 64 characters                                          |
