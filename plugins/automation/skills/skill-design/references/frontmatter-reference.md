# Frontmatter reference

Complete reference for YAML frontmatter fields in SKILL.md files.

---

## Required fields

### `name`

The skill's identifier. You MUST ensure it matches the parent directory name.

| Property   | Value                                                            |
|------------|------------------------------------------------------------------|
| Type       | String                                                           |
| Required   | No by schema, which defaults to the filename. Required by ataraxis convention |
| Max length | 64 characters, by ataraxis convention                            |
| Format | Lowercase letters, digits, and hyphens only, no consecutive `--`, by convention |
| Convention | Must match parent directory name: `explore-codebase`, `commit`   |

### `description`

Explains what the skill does and when to use it. Serves as the primary trigger mechanism, because the agent reads all
skill descriptions at session start to decide when to invoke each skill.

| Property     | Value                                                  |
|--------------|--------------------------------------------------------|
| Type         | String (use YAML `>-` for multi-line)                  |
| Required     | Yes, by ataraxis convention                            |
| Max length   | 1024 characters, an ataraxis budget rather than a schema cap |
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
| Usage    | Set to `false` for a skill the model invokes through the Skill tool, hiding its slash command  |

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

Controls execution context. Set to `fork` to run the skill in a sub-agent with a separate context window, keeping the
main conversation context clean.

| Property | Value  |
|----------|--------|
| Type     | String |
| Values   | `inline`, `fork` |

### `agent`

Specifies which sub-agent type to use when `context: fork` is set.

| Property | Value                           |
|----------|---------------------------------|
| Type     | String                          |
| Values   | `Explore`, `Plan`, `general-purpose`, etc. |

### `hooks`

Hooks registered while the skill is active, using the same shape as the `hooks` block in settings.json.

| Property | Value                                              |
|----------|----------------------------------------------------|
| Type     | Object                                             |
| Usage    | Advanced pattern for validation or post-processing |

### `when_to_use`

Guidance for when the model should reach for this skill. It becomes part of the tool description, so a trigger stated
here does not need to be packed into `description`.

| Property | Value  |
|----------|--------|
| Type     | String |

### `paths`

Glob patterns this skill applies to. The skill loads only when the model touches a matching file.

| Property | Value           |
|----------|-----------------|
| Type     | Array of string |

### `effort`

Thinking effort the model applies while the skill is active.

| Property | Value                                          |
|----------|------------------------------------------------|
| Values   | `low`, `medium`, `high`, `max`, or an integer  |

### `shell`

Shell used for `` !`command` `` blocks. Defaults to bash on every platform, so set it only for a Windows-only skill.

| Property | Value              |
|----------|--------------------|
| Values   | `bash`, `powershell` |

### `background`

Applies to `context: fork` alone. A fork runs as a background agent reporting through a task notification, and setting
this to `false` keeps the caller waiting for the result in-line.

| Property | Value   |
|----------|---------|
| Type     | Boolean |
| Default  | `true`  |

### `metadata`

Free-form map for the skill author's own use, such as entitlement or catalog fields. It is preserved on the loaded
skill and reaches nothing in the harness.

| Property | Value  |
|----------|--------|
| Type     | Object |

---

## String substitution variables

SKILL.md content can include variables that are replaced at runtime.

| Variable               | Description                               |
|------------------------|-------------------------------------------|
| `$ARGUMENTS`           | All arguments passed after the skill name |
| `$ARGUMENTS[N]` / `$N` | Positional argument at index N (0-based)  |
| `${CLAUDE_SESSION_ID}` | Unique identifier for the current session |
| `${CLAUDE_SKILL_DIR}`  | Directory holding this SKILL.md, for pointing a Read or Bash call at its own references/ |
| `${CLAUDE_PROJECT_DIR}`| Root directory of the active project                                                     |
| `${CLAUDE_EFFORT}`     | Thinking effort currently in force                                                       |

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
| Max length             | 64 characters, by ataraxis convention                  |
