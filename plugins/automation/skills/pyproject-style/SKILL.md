---
name: pyproject-style
description: >-
  Applies pyproject.toml conventions when creating or modifying pyproject.toml files. Covers
  section ordering, metadata fields, dependency specifications, tool configurations (ruff, mypy,
  coverage, hatch), and classifier templates. Use when creating a new project, modifying an existing
  pyproject.toml, adding dependencies, or when the user asks about pyproject.toml conventions.
user-invocable: false
---

# pyproject.toml style guide

Applies conventions for pyproject.toml files.

You MUST read this skill and load the relevant reference files before creating or modifying any
pyproject.toml file. You MUST verify your changes against the checklist before submitting.

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

Write or modify the pyproject.toml following all conventions from this file and the loaded
references.

### Step 5: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before
submitting work.

---

## Section ordering

pyproject.toml files use the following section order. This order is mandatory for all
projects.

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

`[tool.ruff.format]` comes first, then the `[tool.ruff.lint.*]` sub-tables run `pycodestyle`,
`pydocstyle`, `per-file-ignores`, `isort`, and then the optional `flake8-unused-arguments`.

`[tool.coverage.run]` holds the `omit` list that excludes interface modules from coverage
measurement, along with the `parallel`, `concurrency`, and `branch` keys used by projects that run
tests across processes or measure branch coverage. It is omitted only by projects that need none of
those keys.

`[[tool.mypy.overrides]]` and `[tool.pytest.ini_options]` carry project-specific content, so this
skill fixes their position and leaves their contents to the project. See
[tool-configurations.md](references/tool-configurations.md) for the pytest keys.

For C-extension projects using scikit-build-core, replace the hatch build targets with
`[tool.scikit-build]` and `[tool.cibuildwheel]` sections at positions 6-7.

---

## TOML formatting

### Block comments

Use block comments above sections to describe their purpose:

```toml
# Project metadata section. Provides the general ID information about the project.
[project]
```

### Inline comments

Use inline comments for individual values when clarification is needed. Align inline comments
vertically within a section:

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
    "tomli>=2,<3",

    # Testing
    "pytest>=9,<10",
    "pytest-cov>=7,<8",
]
```

Separate category groups with a blank line. The comment line has no leading blank line before the
first category.

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
- Single-line format is acceptable for arrays with one or two short elements

### Comment restraint

A TOML key states its own name and its own value, so a comment beside it is justified only by a
question the key leaves open. The qualifying questions are a unit that the value does not carry, a
rationale for a specific bound or version, a coupling to another file that a reader would otherwise
break, and a decision that looks wrong until its reason is given. A comment that restates the key
is noise, and a file where every key carries one trains the reader to skip all of them.

Two standing exceptions apply. Every entry in either ruff ignore corpus carries a reason comment,
because the entry records a decision rather than a value. Every section keeps its one-line block
comment naming its purpose.

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

Comments also describe the configuration as it currently stands, never the edit that produced it.
Do not record that a bound was raised, that a dependency was added, or that an ignore was removed,
because the commit message carries that history.

### Prose punctuation and positive description

Comment prose in pyproject.toml follows the two project-wide rules for documentation. Prose uses
only the full stop and the comma to separate clauses. Do not use a semicolon or an em-dash (`--`,
`—`, or `–`) as a separator, and use a colon only where it is lexically appropriate. A single hyphen
stays available as a list marker, in tables, and in compound words. State what the setting does and
what is currently true. Do not frame it by what it is not or what it used to be, and keep a "not Y"
contrast only when it is load-bearing because it corrects a counter-intuitive assumption, giving its
reason.

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
requires = ["scikit-build-core>=0,<1", "nanobind>=2,<3"]
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

This pattern allows patch and minor updates while preventing breaking major version changes. This
convention applies to both runtime and development dependencies.

For pre-release or unstable packages (major version 0), pin to the minor version when the minor
version is significant, or use `>=0,<1` when the full range is acceptable:

```toml
"ruff>=0,<1"
"httpx>=0.28,<1"
```

### Platform-specific dependencies

Use environment markers for platform-conditional dependencies:

```toml
"intel-cmplr-lib-rt>=2025,<2026; sys_platform != 'darwin'"
"tbb4py>=2022,<2023; sys_platform != 'darwin'"
```

---

## Project layout

All Python projects use the **src layout**. For complete directory trees, invoke
`/project-layout`.

The wheel configuration always points to the src directory:

```toml
[tool.hatch.build.targets.wheel]
packages = ["src/package_name"]
```

Additional directories (e.g., `notebooks`, `examples`) may be included in the wheel when they are
part of the distributed package.

---

## Related skills

| Skill                   | Relationship                                                          |
|-------------------------|-----------------------------------------------------------------------|
| `/explore-dependencies` | Explores ataraxis dependency APIs after adding ataraxis deps          |
| `/python-style`         | Provides coding conventions that pyproject.toml tool configs enforce  |
| `/readme-style`         | Provides README conventions; the `readme` field references the README |
| `/project-layout`       | Provides complete directory trees; this skill owns wheel config       |
| `/tox-config`           | Consumes dependency groups for tox environments; co-evolves with deps |
| `/commit`               | Should be invoked after completing pyproject.toml changes             |
| `/explore-codebase`     | Provides project context needed when writing project-specific configs |

---

## Proactive behavior

When creating a new project, proactively offer to generate a pyproject.toml following
these conventions. When modifying Python version support, dependencies, or tool configurations,
proactively suggest updating the pyproject.toml to reflect the changes. After substantial
dependency or configuration changes, proactively offer to run the verification checklist.

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to
pyproject.toml files.**

```text
pyproject.toml Style Compliance:

Structure:
- [ ] Section ordering follows canonical order (build-system, project, urls, scripts, deps, ...)
- [ ] All required sections present for project type
- [ ] No duplicate sections or keys

Build System:
- [ ] [build-system] uses hatchling (pure-Python) or scikit-build-core (C-extension)
- [ ] Build requirements use range constraints (>=X,<Y)

Project Metadata:
- [ ] name uses lowercase hyphenated format
- [ ] version follows semantic versioning (X.Y.Z or X.Y.ZrcN)
- [ ] description is a single descriptive sentence
- [ ] readme = "README.md"
- [ ] license uses SPDX expression string (PEP 639)
- [ ] license-files includes the LICENSE file
- [ ] No License :: classifiers present (removed per PEP 639)
- [ ] requires-python specifies supported range
- [ ] authors and maintainers arrays present with correct format
- [ ] keywords array present with relevant terms
- [ ] Classifiers grouped with category comments
- [ ] Classifiers match official PyPI classifier list exactly
- [ ] "Typing :: Typed" classifier present

Dependencies:
- [ ] All dependencies use major-version range format (>=X,<Y)
- [ ] Dependencies grouped by category with comments
- [ ] Platform-specific deps use environment markers
- [ ] Trailing commas on all array elements

Dependency Groups (PEP 735):
- [ ] [dependency-groups] used instead of [project.optional-dependencies] for dev deps
- [ ] dev group includes tox, uv, tox-uv, and ataraxis-automation
- [ ] Type stub packages included in dev group where needed

URLs:
- [ ] Homepage points to GitHub repository
- [ ] Documentation URL present (if hosted docs exist)

Scripts:
- [ ] Entry points use package.module:function format
- [ ] Command names are descriptive and do not conflict with system commands

Build Targets:
- [ ] sdist excludes [".github", "recipe"]
- [ ] wheel packages lists src/package_name

Tool Configurations:
- [ ] Ruff: line-length = 120, indent-width = 4
- [ ] Ruff: target-version matches lowest supported Python
- [ ] Ruff: src = ["src"]
- [ ] Ruff: lint.select = ["ALL"] with project-specific ignores
- [ ] Ruff: lint.ignore carries the complete shared block, then any project-specific entries below a blank line
- [ ] Ruff: S602, S607, and SLF001 stay out of the shared block of lint.ignore, appearing there only as
      project-specific entries when library code needs them
- [ ] Ruff: When a tests/ directory exists, per-file-ignores use the tests/**/*.py glob and carry the complete
      shared test corpus, which includes SLF001 and T201
- [ ] Ruff: The test glob is the root-anchored tests/**/*.py, not the **/tests/**/*.py variant that also waives the
      corpus for any tests directory nested under src/
- [ ] Ruff: A tests/**/*.py key is paired with a lint task that passes ./tests to ruff check (see /tox-config),
      since the key is inert otherwise
- [ ] Ruff: flake8-unused-arguments, when present, is the last lint sub-table, below isort
- [ ] Ruff: format uses double quotes, space indentation
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
- [ ] Multi-line arrays with trailing commas
- [ ] One element per line in multi-line arrays
- [ ] Category comments in dependency and classifier arrays
- [ ] Every inline comment answers a question its key leaves open (no comment restating the key)
- [ ] Comments record current configuration only, never the edit that produced it
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons and hyphen bullets fine)
- [ ] Prose states what the setting does, not what it is not or used to be (contrast only when load-bearing)
```
