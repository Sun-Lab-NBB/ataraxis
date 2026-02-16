---
name: pyproject-style
description: >-
  Applies Sun Lab pyproject.toml conventions when creating or modifying pyproject.toml files. Covers
  section ordering, metadata fields, dependency specifications, tool configurations (ruff, mypy,
  coverage, hatch), and classifier templates. Use when creating a new project, modifying an existing
  pyproject.toml, adding dependencies, or when the user asks about pyproject.toml conventions.
user-invocable: true
---

# pyproject.toml style guide

Applies Sun Lab conventions for pyproject.toml files.

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
- Python code style (invoke `/python-style`)
- README file conventions (invoke `/readme-style`)
- Commit message conventions (invoke `/commit`)
- tox.ini configuration (managed separately from pyproject.toml)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file. The section ordering and core rules below apply to ALL pyproject.toml files.

### Step 2: Load relevant reference files

Based on the task, load the appropriate reference files:

| Task                                       | Reference to load                                             |
|--------------------------------------------|---------------------------------------------------------------|
| Writing or modifying project metadata      | [project-metadata.md](references/project-metadata.md)         |
| Writing or modifying tool configurations   | [tool-configurations.md](references/tool-configurations.md)   |
| Creating a new pyproject.toml from scratch | Load both references                                          |

### Step 3: Determine project type

Identify the project type to apply the correct configuration tier:

| Project type | Naming pattern | MyPy mode   | Python support | Dependency style |
|--------------|----------------|-------------|----------------|------------------|
| Core library | `ataraxis-*`   | Full strict | `>=3.12,<3.15` | Range (`>=X,<Y`) |
| Application  | `sl-*`         | Minimal     | `>=3.14,<3.15` | Range (`>=X,<Y`) |
| C-extension  | Any            | Full strict | `>=3.12,<3.15` | Range (`>=X,<Y`) |

### Step 4: Apply conventions

Write or modify the pyproject.toml following all conventions from this file and the loaded
references.

### Step 5: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before
submitting work.

---

## Section ordering

pyproject.toml files use the following section order. This order is mandatory for all Sun Lab
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
10. `[tool.coverage.paths]`
11. `[tool.coverage.html]`
12. `[tool.coverage.report]`

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
case-sensitive = true              # Takes case into account when sorting imports
combine-as-imports = true          # Combines multiple "as" imports for the same package
force-wrap-aliases = true          # Wraps "as" imports so that each uses a separate line
force-sort-within-sections = true  # Forces "as" and "from" imports to be close together
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

---

## Build system

All pure-Python Sun Lab projects use hatchling:

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

Sun Lab projects use the **major-version range** pattern for all dependencies:

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

All Sun Lab Python projects use the **src layout**. For complete directory trees, invoke
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

| Skill               | Relationship                                                          |
|---------------------|-----------------------------------------------------------------------|
| `/python-style`     | Provides coding conventions that pyproject.toml tool configs enforce  |
| `/readme-style`     | Provides README conventions; the `readme` field references the README |
| `/project-layout`   | Provides complete directory trees; this skill owns wheel config       |
| `/tox-config`       | Consumes dependency groups for tox environments; co-evolves with deps |
| `/commit`           | Should be invoked after completing pyproject.toml changes             |
| `/explore-codebase` | Provides project context needed when writing project-specific configs |

---

## Proactive behavior

When creating a new Sun Lab project, proactively offer to generate a pyproject.toml following
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
- [ ] Ruff: format uses double quotes, space indentation
- [ ] Ruff: Google docstring convention
- [ ] Ruff: isort configured (case-sensitive, combine-as-imports, etc.)
- [ ] Ruff: __init__.py ignores F401 and F403
- [ ] Ruff: Each ignore has an explanatory inline comment
- [ ] MyPy: Configuration tier matches project type (full strict or minimal)
- [ ] MyPy: Standard exclusion list present
- [ ] Coverage: paths, html, and report sections present
- [ ] Coverage: Standard exclude_lines list present

Formatting:
- [ ] Block comments above section headers
- [ ] Inline comments aligned within sections
- [ ] Multi-line arrays with trailing commas
- [ ] One element per line in multi-line arrays
- [ ] Category comments in dependency and classifier arrays
```
