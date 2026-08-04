---
name: project-layout
description: >-
  Applies project directory structure conventions when creating new projects, adding directories, or verifying project
  layout. Covers the five archetypes (Python-only, Python+C++ extension, C++ PlatformIO library, C++ PlatformIO
  firmware, C# Unity), common root files, environment, test, and documentation directories. Use when creating a new
  project, adding top-level directories, restructuring a project, or when the user asks about project directory
  conventions.
user-invocable: false
---

# Project layout

Applies conventions for project directory structure across all five project archetypes.

You MUST read this skill before creating, restructuring, or verifying any project's directory layout. You MUST verify
your changes against the checklist before submitting.

---

## Scope

**Covers:**
- Project directory trees for all five archetypes
- Common root files and their purposes
- Environment directories (`envs/`) with OS-specific files
- Test directory conventions (`tests/` vs `test/`)
- Documentation directory placement (defers to `/api-docs` for internal structure)
- `.github/` directory structure and the shared issue template corpus
- Archetype identification criteria

**Does not cover:**
- File-level ordering within source files (see `/python-style`, `/cpp-style`, `/csharp-style`)
- Documentation file contents and Sphinx configuration (see `/api-docs`)
- pyproject.toml structure and fields (see `/pyproject-style`)
- README file contents (see `/readme-style`)
- Code style or formatting rules (see language-specific style skills)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Identify the project archetype

Classify the repository mechanically before any tree comparison, as one of the five archetypes or as an umbrella
repository. Determine which archetype applies using this table:

| Archetype               | Key indicators                                                       |
|-------------------------|----------------------------------------------------------------------|
| Python-only             | `pyproject.toml` + `src/` layout, no `.clang-format` or `CMakeLists` |
| Python + C++ extension  | `pyproject.toml` + `CMakeLists.txt` + `src/c_extensions/`            |
| C++ PlatformIO library  | `platformio.ini` + `library.json` + `src/*.h`                        |
| C++ PlatformIO firmware | `platformio.ini` + `src/main.cpp`, no `library.json`                 |
| C# Unity                | `Assets/` + `ProjectSettings/` + `*.slnx`                            |

A repository that carries none of these indicators, ships no installable artifact of its own, and instead indexes
sibling libraries or distributes plugins through a marketplace is an umbrella repository. Umbrella repositories carry NO
archetype tree, so this skill prescribes no directory layout for one and a layout audit records the tree as unresolvable
rather than reporting the archetype paths it lacks. Its `README.md` still follows the umbrella order in `/readme-style`.

### Step 2: Load the reference tree

Read [archetype-trees.md](references/archetype-trees.md) and locate the section matching the identified archetype. The
reference tree is the authoritative source for directory structure.

### Step 3: Apply conventions

Create or verify the project structure against the reference tree and the rules below. When creating a new project,
generate all required directories and files. When verifying, report any deviations from the expected structure. Every
tracked top-level path is one the archetype tree sanctions, so a path the tree does not list is removed or its presence
justified, with gitignored build artifacts out of scope.

### Step 4: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass before submitting work.

---

## Every file occupies a slot in the tree

A file added to the repository MUST occupy a slot its layout defines. That test decides the question, rather than
whether the user named the file. A project repository is judged against its archetype tree. An umbrella repository
carries no archetype tree, so it is judged against the plugin and marketplace layout `/skill-design` owns.

Adding a file to a defined slot is ordinary work and needs no permission. A module under `src/{package_name}/`, a test
module under `tests/`, a documentation page under `docs/source/`, a skill file under a skill's own directory, and an
issue template under `.github/ISSUE_TEMPLATE/` are all part of doing the task.

A file that fits NO slot is the violation, and the repository ROOT is where it usually lands, because the root holds a
fixed set this skill enumerates and nothing else. The recurring offenders are working artifacts: notes, findings,
reports, audit output, plans, summaries, checklists, scratch scripts, and generated data. A finding belongs in the
reply to the user, and a plan belongs in the reply to the user. Neither belongs in a tracked file, and a file the user
has to delete afterwards is worse than no file at all.

Work that genuinely needs a file on disk goes OUTSIDE the repository, in the session scratch directory. When a file
fits no slot and the task still seems to need it tracked, stop and ask, because the answer decides whether the file
exists rather than where it goes.

---

## Common root files

These files appear at the root of all (or most) projects:

| File            | All archetypes     | Purpose                                                       |
|-----------------|--------------------|---------------------------------------------------------------|
| `LICENSE`       | Yes                | Apache-2.0 license                                            |
| `README.md`     | Yes                | Project documentation (see `/readme-style`)                   |
| `.gitignore`    | Yes                | Git ignore patterns                                           |
| `CLAUDE.md`     | Yes                | Claude Code project instructions                              |
| `tox.ini`       | Python + C++       | Automation orchestration (lint, type, test, docs)             |
| `.netlify-site` | Projects with docs | Netlify site identifier used by the `deploy` task             |
| `.codegraph/`   | Optional           | CodeGraph index, present when the repository has been indexed |

The `.netlify-site` file stores the identifier of the Netlify site that serves the project's API documentation. The
identifier is not a secret and differs for each project, so the file is tracked by version control. See `/tox-config`
for where the `deploy` and `upload` API tokens live.

The file and the `deploy` tox environment are one unit, because that environment reads the identifier from it. A project
carries both or neither, and a project that builds documentation without hosting it keeps its `docs` environment and
drops both. See `/tox-config` for the environment.

The `.codegraph/` directory holds a generated code index. It is present only in repositories that have been indexed.
Every file inside it is ignored by version control except its own `.gitignore`, which is tracked so that the exclusion
travels with the repository.

### Python-specific root files

| File             | Archetypes                  | Purpose                                   |
|------------------|-----------------------------|-------------------------------------------|
| `pyproject.toml` | Python-only, Python+C++ ext | Build config, metadata, tool settings     |
| `CMakeLists.txt` | Python+C++ ext              | CMake build config for nanobind extension |
| `Doxyfile`       | Python+C++ ext              | Doxygen documentation configuration       |
| `.clang-format`  | Python+C++ ext              | C++ formatting configuration              |
| `.clang-tidy`    | Python+C++ ext              | C++ linting configuration                 |

### PlatformIO-specific root files

| File             | Archetypes                    | Purpose                             |
|------------------|-------------------------------|-------------------------------------|
| `platformio.ini` | PlatformIO lib, PlatformIO fw | PlatformIO build configuration      |
| `library.json`   | PlatformIO lib only           | PlatformIO library manifest         |
| `Doxyfile`       | PlatformIO lib, PlatformIO fw | Doxygen documentation configuration |
| `.clang-format`  | PlatformIO lib, PlatformIO fw | C++ formatting configuration        |
| `.clang-tidy`    | PlatformIO lib, PlatformIO fw | C++ linting configuration           |

### Unity-specific root files

| File                | Purpose                                      |
|---------------------|----------------------------------------------|
| `*.slnx`            | Unity solution file                          |
| `.editorconfig`     | Editor configuration (indentation, encoding) |
| `.csharpierrc.yaml` | CSharpier formatter configuration            |
| `.csharpierignore`  | CSharpier formatter ignore patterns          |

---

## Environment directories

Python projects (Python-only and Python+C++ extension) include an `envs/` directory with OS-specific conda/mamba
environment files:

```text
envs/
├── {abbr}_dev_lin.yml            # Linux conda environment specification
├── {abbr}_dev_osx.yml            # macOS conda environment specification
└── {abbr}_dev_win.yml            # Windows conda environment specification
```

The `{abbr}` placeholder is a short project abbreviation (e.g., `axa` for ataraxis-automation, `axbu` for
ataraxis-base-utilities). Each platform has one `.yml` file, which is the human-readable conda environment specification
used by `mamba env create`. The `export` tox task writes these files, and the `import` task recreates the environment
from them.

`envs/` holds one `.yml` file per platform and nothing else, because the `export` task produces the `.yml` file alone,
so remove any `_spec.txt` file found there.

PlatformIO and Unity projects do NOT have `envs/` directories.

---

## Test directories

| Archetype               | Directory | Framework | File pattern                |
|-------------------------|-----------|-----------|-----------------------------|
| Python-only             | `tests/`  | pytest    | `module_test.py`            |
| Python + C++ extension  | `tests/`  | pytest    | `module_test.py`            |
| C++ PlatformIO library  | `test/`   | Unity (C) | `test_component.cpp`        |
| C++ PlatformIO firmware | (none)    | (none)    | No test directory           |
| C# Unity                | (none)    | (none)    | Unity Play Mode / Edit Mode |

### Python test structure

The `tests/` directory mirrors the `src/package_name/` subpackage structure:

```text
tests/
├── submodule/
│   └── module_test.py
└── standalone_test.py
```

### PlatformIO test structure

The `test/` directory (singular, not `tests/`) follows PlatformIO's native test convention:

```text
test/
└── test_component.cpp
```

---

## Documentation directory

All Python and C++ projects include a `docs/` directory for Sphinx documentation. For the complete internal structure,
Sphinx configuration, and RST templates, invoke `/api-docs`.

C# Unity projects do NOT have a `docs/` directory.

---

## `.github/` directory

Every project published to GitHub as a standalone repository includes a `.github/` directory that holds the shared issue
template corpus. All five archetypes use the same corpus:

```text
.github/
└── ISSUE_TEMPLATE/
    ├── bug_report.yml            # Structured bug report form
    ├── config.yml                # Template chooser configuration
    └── feature_request.yml       # Structured feature request form
```

The corpus uses GitHub issue forms, which validate required fields at submission time and apply the `bug` and
`enhancement` labels that GitHub creates in every repository. Copy all three files from [assets/github/](assets/github/)
when creating or updating a repository.

### Corpus substitution rules

`bug_report.yml` and `feature_request.yml` are identical in every repository. Copy both files verbatim, which keeps the
corpus consistent as it spreads across repositories. The bug report form asks for the environment as free text, so one
form serves Python, PlatformIO, and Unity projects alike.

The example values inside the form placeholders are illustrative. They show the shape of a useful answer rather than the
state of any one project, so the version, environment, and reproduction examples stay as the asset spells them.

`config.yml` carries a single substitution. Replace the `{project}` placeholder in the API documentation link with the
repository name, which produces the Netlify address that serves the project's API documentation:

```yaml
url: https://{project}-api-docs.netlify.app/    # https://ataraxis-automation-api-docs.netlify.app/
```

Projects that build API documentation keep both contact links. Projects that ship without API documentation keep the AI
development assets link alone, which avoids publishing an address that resolves to nothing.

The `blank_issues_enabled: false` setting routes every reported issue through one of the two forms. The contact links
carry the traffic that suits neither form, sending usage questions to the API documentation and skill defects to the
ataraxis repository that hosts the plugin marketplace.

`config.yml` carries only its two sanctioned edits, the `{project}` substitution and the removal of the API
documentation contact link, with `blank_issues_enabled: false` and the remaining contact link left as the asset spells
them.

---

## Source directory conventions

### Python-only (`src/` layout)

```text
src/
└── package_name/
    ├── __init__.py
    ├── module.py
    └── py.typed
```

### Python + C++ extension (`src/` flat namespace)

```text
src/
├── c_extensions/
│   └── module_ext.cpp
├── python_wrapper/
│   ├── __init__.py
│   └── wrapper_module.py
├── __init__.py
├── module_ext.pyi
└── py.typed
```

`.pyi` stub files and the `py.typed` marker are generated artifacts whose presence is release-phase-dependent, so a
missing `.pyi` is not a layout violation. See `/python-style` for the stub-file rule.

### PlatformIO library (header-only `src/`)

```text
src/
├── main.cpp                  # Development entry point (excluded from library)
├── primary_header.h
└── shared_assets.h
```

### PlatformIO firmware (`src/`)

```text
src/
├── main.cpp                  # Firmware entry point (setup/loop)
└── custom_module.h
```

### C# Unity (`Assets/`)

```text
Assets/
├── TaskName/
│   ├── Configurations/
│   ├── Materials/
│   ├── Prefabs/
│   ├── Scripts/
│   │   └── TaskScript.cs
│   ├── Sounds/
│   └── Textures/
├── Plugins/
└── Scenes/
```

---

## Related skills

| Skill                | Relationship                                                                 |
|----------------------|------------------------------------------------------------------------------|
| `/api-docs`          | Owns the internal `docs/` structure. This skill owns its directory placement |
| `/python-style`      | Owns file-level ordering within Python source files                          |
| `/cpp-style`         | Owns file-level ordering within C++ source files                             |
| `/csharp-style`      | Owns file-level ordering within C# source files                              |
| `/pyproject-style`   | Owns `pyproject.toml` structure and references the `src/` layout convention  |
| `/tox-config`        | Owns `tox.ini` conventions, and `tox.ini` is a common root file              |
| `/platformio-config` | Owns `platformio.ini` and `library.json` conventions (C++ archetypes)        |
| `/readme-style`      | Owns `README.md` content conventions                                         |
| `/skill-design`      | Owns `plugins/automation/skills/` directory structure conventions            |
| `/audit-project`     | Runs the wave 1 layout sweep over the archetype trees this skill owns        |

---

## Proactive behavior

You should proactively offer to invoke this skill when:
- Creating a new project from scratch
- The user asks where to place a new file or directory
- A project structure appears to deviate from conventions during exploration
- The user asks about project directory conventions or archetypes

---

## Verification checklist

You MUST verify your work against this checklist before submitting any layout changes.

```text
Project Layout Compliance:
- [ ] Every added file occupies a slot the archetype tree defines, with no working artifact (notes, findings,
      reports, plans, scratch scripts) left in the tree, especially at the repository root

Tool-settled items. `git ls-files` and `ls -a` decide each of these against the archetype tree, so run
them rather than recalling the layout.
- [ ] LICENSE present (Apache-2.0)
- [ ] README.md present
- [ ] .gitignore present
- [ ] CLAUDE.md present
- [ ] Archetype-specific root files present (pyproject.toml, platformio.ini, etc.)
- [ ] No tracked top-level path outside the archetype tree, with any extra directory or root file removed
      or its presence justified (gitignored build artifacts exempt)
- [ ] Python projects use src/ layout with package_name/ subdirectory
- [ ] Python+C++ extension uses flat namespace under src/ (c_extensions/, wrapper/, etc.)
- [ ] PlatformIO projects use src/ with header-only .h files
- [ ] Unity projects use Assets/ with task-specific subdirectories
- [ ] Python projects have envs/ with 3 files, one .yml per supported platform
- [ ] envs/ holds .yml files alone, with any _spec.txt exports removed
- [ ] envs/ file names use correct abbreviation prefix
- [ ] PlatformIO and Unity projects do NOT have envs/
- [ ] Python projects use tests/ (plural) with _test.py suffix, mirroring the src/package_name/ subpackage structure
- [ ] PlatformIO library projects use test/ (singular) with test_ prefix
- [ ] PlatformIO firmware and Unity projects have no dedicated test directory
- [ ] Python and C++ projects have docs/ directory
- [ ] Unity projects do NOT have docs/
- [ ] .github/ISSUE_TEMPLATE/ present for every repository published to GitHub
- [ ] ISSUE_TEMPLATE/ holds exactly bug_report.yml, config.yml, and feature_request.yml

Reader-judged items. No directory listing settles these, so decide each one by reading the files it
names and this skill's ownership boundary.
- [ ] Repository classified mechanically: one of the five archetypes, or umbrella (no archetype tree,
      layout recorded as unresolvable and the remaining rows skipped)
- [ ] Project archetype correctly identified from key indicators
- [ ] Reference tree loaded from archetype-trees.md
- [ ] .netlify-site and the deploy tox environment are both present or both absent
- [ ] No hand-authored or stale .pyi stubs committed mid-development (stubs are generated at release time via tox -e stubs)
- [ ] bug_report.yml and feature_request.yml copied verbatim from assets/github/
- [ ] config.yml {project} placeholder replaced with the repository name
- [ ] config.yml otherwise unchanged from assets/github/, keeping blank_issues_enabled: false and the
      retained contact link destinations
- [ ] API documentation contact link retained only for projects that build API documentation
- [ ] Directory trees not duplicated in other skills (api-docs owns docs/ internals)
- [ ] File-level ordering not specified (owned by language style skills)
```
