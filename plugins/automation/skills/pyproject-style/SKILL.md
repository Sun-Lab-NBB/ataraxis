---
name: pyproject-style
description: >-
  Applies pyproject.toml conventions when creating or modifying pyproject.toml files. Covers section ordering, metadata
  fields, dependency specifications, tool configurations (ruff, mypy, coverage, hatch), and classifier templates. Use
  when creating a new project, modifying an existing pyproject.toml, adding dependencies, or when the user asks about
  pyproject.toml conventions.
user-invocable: false
---

# pyproject.toml style guide

Applies conventions for pyproject.toml files.

You MUST read this skill and load the relevant reference files before creating or modifying any pyproject.toml file. You
MUST verify your changes against the checklist before submitting.

---

## Scope

**Covers:**
- pyproject.toml section ordering and structure
- Project metadata fields (name, version, description, license, classifiers, etc.)
- Dependency specifications and version constraint conventions
- Dependency groups (PEP 735) and entry point conventions
- Tool configurations (ruff, mypy, coverage, hatch build targets)
- TOML formatting and comment style
- Project type distinctions (core library vs application, pure-Python vs C-extension)

**Does not cover:**
- Python code style (see `/python-style`)
- README file conventions (see `/readme-style`)
- Commit message conventions (see `/commit`)
- tox.ini configuration (see `/tox-config`)

---

## Mirrored identity content requires explicit approval

The `name`, `description`, and `authors` fields state the project's identity, and each one is duplicated outside this
file. The description alone appears in the four canonical locations
[project-metadata.md](references/project-metadata.md) enumerates, and again in the repository description on GitHub and
on the package page on PyPI. Changing one copy obligates changing every copy, so you MUST obtain explicit user
approval, given for the specific edit in front of you, before altering any of them.

The test is whether the content is duplicated outside the file you are editing. When it is, stop, quote the current
text alongside the proposed replacement, name every location that would need the same edit, and wait for an answer. A
blanket instruction to fix findings, apply a documentation change, or implement a plan does NOT authorize these edits,
and neither does the edit being provably correct. Every other field in this file follows the normal rules, as does the
`version` bump `/release` performs under its own confirmation.

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file. The section ordering and core rules below apply to ALL pyproject.toml files.

### Step 2: Load relevant reference files

Based on the task, load the appropriate reference files:

| Task                                       | Reference to load                                           |
|--------------------------------------------|-------------------------------------------------------------|
| Writing or modifying project metadata      | [project-metadata.md](references/project-metadata.md)       |
| Writing or modifying tool configurations   | [tool-configurations.md](references/tool-configurations.md) |
| Creating a new pyproject.toml from scratch | Load both references                                        |

### Step 3: Determine project type

Identify the project type to apply the correct configuration tier:

| Project type | Naming pattern     | MyPy mode   | Python support | Dependency style |
|--------------|--------------------|-------------|----------------|------------------|
| Core library | `ataraxis-*`       | Full strict | `>=3.12,<3.15` | Range (`>=X,<Y`) |
| Application  | (project-specific) | Minimal     | `>=3.14,<3.15` | Range (`>=X,<Y`) |
| C-extension  | Any                | Full strict | `>=3.12,<3.15` | Range (`>=X,<Y`) |

### Step 4: Apply conventions

Write or modify the pyproject.toml following all conventions from this file and the loaded references.

### Step 5: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before submitting work.

---

## Section ordering

pyproject.toml files use the following section order. This order is mandatory for all projects.

1. `[build-system]`
2. `[project]`
3. `[project.urls]`
4. `[project.scripts]` *(if applicable)*
5. `[dependency-groups]`
6. `[tool.hatch.build.targets.sdist]`
7. `[tool.hatch.build.targets.wheel]`
8. `[tool.ruff]` and sub-tables
9. `[tool.mypy]`
10. `[[tool.mypy.overrides]]` *(if applicable)*
11. `[tool.pytest.ini_options]` *(projects with a tests/ directory)*
12. `[tool.coverage.run]`
13. `[tool.coverage.paths]`
14. `[tool.coverage.html]`
15. `[tool.coverage.report]`

Two further tables have optional positions. `[tool.hatch.build.targets.wheel.sources]` follows position 7, and is
present where a wheel remaps a packaged directory onto a different import name. `[tool.importlinter]` and its
`[[tool.importlinter.contracts]]` entries follow position 15, and are present where a project enforces import
boundaries.

`[tool.ruff.format]` comes first, then the `[tool.ruff.lint.*]` sub-tables run `pycodestyle`, `pydocstyle`,
`per-file-ignores`, `isort`, and then the optional `flake8-unused-arguments`.

`[tool.coverage.run]` is omitted only by a project that needs none of its keys. See
[tool-configurations.md](references/tool-configurations.md) for those keys.

`[[tool.mypy.overrides]]` and `[tool.pytest.ini_options]` carry project-specific content, so this skill fixes their
position and leaves their contents to the project. See [tool-configurations.md](references/tool-configurations.md) for
the pytest keys.

For C-extension projects using scikit-build-core, replace the hatch build targets with `[tool.scikit-build]` and
`[tool.cibuildwheel]` sections at positions 6-7.

---

## TOML formatting

pyproject.toml adheres to the **120 character line limit**, matching the `line-length` it sets for the project's Python
code, and comment prose wraps at 120. A single unbreakable value is exempt, such as a URL, a PEP 508 requirement string
with markers, or an SPDX expression.

Break a comment line only where it would otherwise pass 120 characters, and fill each line to that limit before
breaking. Comment prose wrapped at a narrower width reads as a rigid block and advertises a limit the file does not set.
The test is mechanical: a wrapped line that ends before column 100 while its next word would still fit under 120 is
re-flowed. A line ending early because the sentence or the comment block ends is already correct.

### Block comments

Use block comments above sections to describe their purpose:

```toml
# Project metadata section. Provides the general ID information about the project.
[project]
```

### Inline comments

Align inline comments vertically within a section. The comment restraint section below settles which values carry an
inline comment:

```toml
case-sensitive = true
combine-as-imports = true          # Combines multiple "as" imports for the same package
force-wrap-aliases = true          # Wraps "as" imports so that each uses a separate line
force-sort-within-sections = true  # Forces "as" and "from" imports for the same package to be close
length-sort = true                 # Places shorter imports first
```

### Category comments in arrays

Group related items within dependency and classifier arrays using category comments:

```toml
dependencies = [
    # Automation Logic
    "click>=8,<9",
    "platformdirs>=4,<5",

    # Testing
    "pytest>=9,<10",
    "pytest-cov>=7,<8",
]
```

Separate category groups with a blank line. The comment line has no leading blank line before the first category.

### Ruff ignore comments

Each ruff ignore entry must have an inline comment explaining the reason:

```toml
lint.ignore = [
    "COM812",  # Conflicts with the formatter
    "ISC001",  # Conflicts with the formatter
    "D107",    # __init__ is documented inside the main class docstring where applicable
]
```

### Array formatting

- Use multi-line format for arrays with more than two elements
- Always use trailing commas in multi-line arrays
- One element per line
- Single-line format is acceptable for arrays of one or two elements that fit inside the 120 character line limit

### Comment restraint

A TOML key states its own name and its own value, so a comment beside it is justified only by a question the key leaves
open. The qualifying questions are a unit that the value does not carry, a rationale for a specific bound or version,
and a coupling to another file that a reader would otherwise break. A decision that looks wrong until its reason is
given also qualifies. A comment that restates the key is noise, and a file where every key carries one trains the reader
to skip all of them.

Two standing exceptions apply. Every entry in either ruff ignore corpus carries a reason comment, because the entry
records a decision rather than a value. Every section keeps its one-line block comment naming its purpose.

```toml
# Avoid - every comment restates the key it sits on
[project]
name = "ataraxis-video-system"  # The name of the project
version = "3.0.0"               # The version of the project
requires-python = ">=3.12"      # The required python version

# Good - the surviving comment answers a question the key leaves open
[project]
name = "ataraxis-video-system"
version = "3.0.0"
requires-python = ">=3.12,<3.15"  # 3.15 is untested against the encoder backend
```

Comments also describe the configuration as it currently stands, never the edit that produced it. Do not record that a
bound was raised, that a dependency was added, or that an ignore was removed, because the commit message carries that
history.

A comment's claim must be true of the key it sits on as it currently reads, covering the value, the bound, and the
effect the tool actually produces. A comment naming a version, a bound, a path, or a coupled file is rewritten or
deleted in the same edit that moves that value. Comments must not reference closed issue numbers, removed keys or
sections, superseded tool versions, or outdated TODOs. A ruff ignore whose reason comment cites an upstream issue is
re-checked when that issue closes, because the ignore usually goes with it.

### Prose punctuation and positive description

Comment prose in pyproject.toml follows the two project-wide rules for documentation. Prose uses only the full stop and
the comma to separate clauses. Do not use a semicolon or an em-dash (`--`, `—`, or `–`) as a separator, and use a colon
only where it is lexically appropriate. A single hyphen stays available as a list marker, in tables, and in compound
words. State what the setting does and what is currently true. Do not frame it by what it is not or what it used to be,
and keep a "not Y" contrast only when it is load-bearing because it corrects a counter-intuitive assumption, giving its
reason. This rule governs prose only. Code stays exempt, so a `;` in a PEP 508 dependency marker or a `--flag` in a CLI
reference is left as written.

Sentences over 40 words are difficult to parse and must be broken at natural clause boundaries, in block comments,
inline comments, and the `description` field alike. Every block comment, inline comment, and `description` field must be
free of typos and grammatical errors.

---

## Build system

All pure-Python projects use hatchling:

```toml
[build-system]
requires = ["hatchling>=1,<2"]
build-backend = "hatchling.build"
```

C-extension projects use scikit-build-core with nanobind:

```toml
[build-system]
requires = ["scikit-build-core>=1,<2", "nanobind>=2,<3"]
build-backend = "scikit_build_core.build"
```

---

## Version constraints

Projects use the **major-version range** pattern for all dependencies:

```toml
"numpy>=2,<3"
"click>=8,<9"
"scipy>=1,<2"
```

This convention applies to both runtime and development dependencies.

For pre-release or unstable packages (major version 0), the default is `>=0,<1`. Raise the lower bound to a specific
minor release when the project calls an API that first appeared in that release:

```toml
"ruff>=0,<1"
"httpx>=0.28,<1"
```

### Platform-specific dependencies

Use environment markers for platform-conditional dependencies:

```toml
"intel-cmplr-lib-rt>=2026,<2027; sys_platform != 'darwin'"
"tbb4py>=2023,<2024; sys_platform != 'darwin'"
```

---

## Project layout

All Python projects use the **src layout**. For complete directory trees, invoke `/project-layout`.

The wheel configuration always points to the src directory:

```toml
[tool.hatch.build.targets.wheel]
packages = ["src/package_name"]
```

Additional directories (e.g., `notebooks`, `examples`) are included in the wheel when a user of the installed
distribution needs their files, such as an examples directory the README points readers to. A directory that only serves
development inside the repository stays out.

---

## Related skills

| Skill                   | Relationship                                                           |
|-------------------------|------------------------------------------------------------------------|
| `/explore-dependencies` | Explores ataraxis dependency APIs after adding ataraxis deps           |
| `/python-style`         | Provides coding conventions that pyproject.toml tool configs enforce   |
| `/readme-style`         | Provides README conventions for the file the `readme` field names      |
| `/project-layout`       | Provides complete directory trees, while this skill owns wheel config  |
| `/tox-config`           | Consumes dependency groups for tox environments, co-evolving with deps |
| `/commit`               | Should be invoked after completing pyproject.toml changes              |
| `/explore-codebase`     | Provides project context needed when writing project-specific configs  |

---

## Proactive behavior

When creating a new project, proactively offer to generate a pyproject.toml following these conventions. When modifying
Python version support, dependencies, or tool configurations, proactively suggest updating the pyproject.toml to reflect
the changes. After substantial dependency or configuration changes, proactively offer to run the verification checklist.

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to pyproject.toml files.**

```text
pyproject.toml Style Compliance:

Settled by tox -e build followed by twine check dist/*:
- [ ] Explicit user approval obtained before changing name, description, or authors, with every location carrying
      a copy named for the user
- [ ] No duplicate sections or keys
- [ ] license uses SPDX expression string (PEP 639)
- [ ] Classifiers match official PyPI classifier list exactly

Judgment items. No tool inspects these, so this checklist is their only enforcement. Walk every one against the file
you wrote. (The three rows above, plus TOML parse validity, are the only tool-settled items.)

Structure:
- [ ] Section ordering follows canonical order (build-system, project, urls, scripts, deps, ...)
- [ ] All required sections present for project type

Build System:
- [ ] [build-system] uses hatchling (pure-Python) or scikit-build-core (C-extension)
- [ ] Build requirements use range constraints (>=X,<Y)

Project Metadata:
- [ ] name uses lowercase hyphenated format
- [ ] version follows semantic versioning (X.Y.Z or X.Y.ZrcN)
- [ ] pyproject.toml version is the single source of the package version, with no second version literal in
      __init__.py, the docs config, or the README
- [ ] description is a single descriptive sentence
- [ ] description opens with a bare third-person imperative verb, with no language prefix ("A Python library that...")
      and no project-name prefix ("project-name is...")
- [ ] description matches the __init__.py module docstring first line, the docs/source/welcome.rst first paragraph,
      and the README.md one-line description verbatim
- [ ] readme = "README.md"
- [ ] license-files includes the LICENSE file
- [ ] No License :: classifiers present (removed per PEP 639)
- [ ] requires-python specifies supported range
- [ ] authors and maintainers arrays present with correct format
- [ ] keywords array present with relevant terms
- [ ] Classifiers grouped with category comments, covering development status, intended audience and topic, Python
      versions, operating systems, and typing
- [ ] Topic classifier matches the project's domain rather than defaulting to Topic :: Software Development
- [ ] One "Programming Language :: Python :: X.Y" classifier per minor version covered by requires-python
- [ ] "Typing :: Typed" classifier present

Dependencies:
- [ ] All dependencies use major-version range format (>=X,<Y)
- [ ] Dependencies grouped by category with comments
- [ ] Platform-specific deps use environment markers
- [ ] Trailing commas on all array elements

Dependency Groups (PEP 735):
- [ ] [dependency-groups] used instead of [project.optional-dependencies] for dev deps
- [ ] dev group includes tox, uv, tox-uv, and ataraxis-automation (omitted by ataraxis-automation itself)
- [ ] Type stub packages included in dev group where needed

URLs:
- [ ] Homepage points to GitHub repository
- [ ] Documentation URL present (if hosted docs exist)

Scripts:
- [ ] Entry points use package.module:function format
- [ ] Command names are short and memorable, with hyphens separating the words of a multi-word command
- [ ] Command names carry the project abbreviation prefix (e.g. axci, axvs), which namespaces them away from the
      commands already on the user's PATH

Build Targets:
- [ ] sdist excludes [".github"]
- [ ] wheel packages lists src/package_name

Tool Configurations:
- [ ] Ruff: line-length = 120, indent-width = 4
- [ ] Ruff: target-version matches lowest supported Python
- [ ] Ruff: src = ["src"]
- [ ] Ruff: extend-exclude = ["conf.py"]
- [ ] Ruff: lint.select = ["ALL"] with project-specific ignores
- [ ] Ruff: lint.ignore carries the complete shared block of the universal lint.ignore corpus, then any
      project-specific entries below a blank line
- [ ] Ruff: S602, S607, and SLF001 stay out of the shared block of the universal lint.ignore corpus, appearing
      there only as project-specific entries when library code needs them
- [ ] Ruff: When a tests/ directory exists, per-file-ignores use the tests/**/*.py glob and carry the complete
      shared test corpus, which includes SLF001 and T201
- [ ] Ruff: The test glob is the root-anchored tests/**/*.py, not the **/tests/**/*.py variant that also waives the
      corpus for any tests directory nested under src/
- [ ] Ruff: A tests/**/*.py key is paired with a lint task that passes ./tests to ruff check (see /tox-config),
      since the key is inert otherwise
- [ ] Ruff: flake8-unused-arguments, when present, is the last lint sub-table, below isort
- [ ] Ruff: format uses double quotes, space indentation
- [ ] Ruff: pycodestyle max-doc-length = 120, matching the code line-length
- [ ] Ruff: Google docstring convention
- [ ] Ruff: isort configured (case-sensitive, combine-as-imports, etc.)
- [ ] Ruff: __init__.py ignores F401 and F403
- [ ] Ruff: Each entry in lint.ignore and in every per-file-ignores key has an explanatory inline comment
- [ ] MyPy: Configuration tier matches project type (full strict or minimal)
- [ ] MyPy: Standard exclusion list present
- [ ] Pytest: Projects with a tests/ directory declare addopts = "--import-mode=importlib"
- [ ] Pytest: pythonpath = ["."] present only where tests spawn subprocesses that import test modules
- [ ] Coverage: paths, html, and report sections present
- [ ] Coverage: branch = true present only where the suite already passes the 100% gate with it enabled
- [ ] Coverage: Standard exclude_lines list present
- [ ] Coverage: fail_under = 100 and show_missing = true set in [tool.coverage.report]
- [ ] Coverage: [tool.coverage.run] omit lists every interface module excluded from measurement in full
- [ ] Coverage: omit patterns start with a wildcard so they match the src/ and site-packages copies
- [ ] Coverage: [tool.coverage.paths] source lists src/ plus the POSIX and Windows site-packages layouts
- [ ] Coverage: modules listed in omit carry no per-line pragma: no cover comments

Formatting:
- [ ] Block comments above section headers
- [ ] Inline comments aligned within sections
- [ ] Lines stay under 120 characters, with unbreakable single values (URLs, requirement strings, $schema) exempt
- [ ] Comments and descriptions fill each line to 120 characters, with no line ending before column 100 while its next
      word would still fit
- [ ] Multi-line arrays with trailing commas
- [ ] One element per line in multi-line arrays
- [ ] Category comments in dependency and classifier arrays
- [ ] Every inline comment answers a question its key leaves open (no comment restating the key)
- [ ] Comments record current configuration only, never the edit that produced it
- [ ] Every comment's claim is true of the value it sits beside and of the effect the tool actually produces
- [ ] No stale references in comments (closed issues, removed keys or environments, superseded tool versions,
      outdated TODOs)
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons, hyphen bullets, and code
      syntax exempt)
- [ ] Prose states what the setting does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Sentences in comments and description fields stay under 40 words
- [ ] Comments and description fields free of typos and grammar errors
```
