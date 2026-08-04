---
name: tox-config
description: >-
  Applies tox.ini conventions when creating or modifying tox configuration files. Covers the
  mamba + uv + tox toolchain, envlist patterns, environment definitions, dependency installation
  strategies, environment naming, and project archetype variations (full Python, reduced Python,
  C++ extension, C++ docs-only). Use when creating or modifying a tox.ini, changing tox
  environments, or when the user asks about tox configuration or the mamba/uv/tox toolchain.
user-invocable: false
---

# tox.ini style guide

Applies conventions for tox.ini configuration files that drive the development automation
pipeline.

You MUST read this skill before creating or modifying any tox.ini file. You MUST verify your
changes against the checklist before submitting.

---

## Scope

**Covers:**
- `[tox]` section structure and `requires` conventions
- Envlist ordering and patterns for each project archetype
- Environment definitions (lint, stubs, test, coverage, docs, build, upload, deploy, install,
  uninstall, create, remove, provision, export, import)
- Mamba + uv + tox toolchain architecture and how the three tools interact
- Dependency installation strategies (`dependency_groups`, `deps`, `skip_install`)
- Environment naming conventions (`{abbr}_dev`)
- Python version parameterization in test environments
- C++ extension and C++ docs-only variations
- The self-hosting exception for ataraxis-automation itself

**Does not cover:**
- pyproject.toml dependency specifications (see `/pyproject-style`)
- Sphinx documentation configuration inside `docs/` (see `/api-docs`)
- Project directory structure (see `/project-layout`)
- Python code style or test code conventions (see `/python-style`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Identify the project archetype

Determine which pipeline applies:

| Archetype      | Envlist pattern                                                              | Key indicator                          |
|----------------|------------------------------------------------------------------------------|----------------------------------------|
| Full Python    | uninstall → export → lint → stubs → test → coverage → docs → build → install | `pyproject.toml` + `src/` layout       |
| C++ extension  | Same as full Python, with Doxygen in docs and cibuildwheel in build          | `CMakeLists.txt` + `pyproject.toml`    |
| Reduced Python | Full Python minus test and coverage                                          | Application project with no unit tests |
| C++ docs-only  | docs only                                                                    | `platformio.ini`, no `pyproject`       |

### Step 2: Load reference templates

Read [environment-templates.md](references/environment-templates.md) for the complete environment
definitions matching the identified archetype.

### Step 3: Determine parameterization

Collect the project-specific values:

| Parameter          | Source                                | Example                   |
|--------------------|---------------------------------------|---------------------------|
| `{package_name}`   | Package directory name under `src/`   | `ataraxis_base_utilities` |
| `{env_abbr}`       | Short project abbreviation            | `axbu`                    |
| `{version}`        | Current ataraxis-automation release   | `9.0.0`                   |
| Python versions    | `requires-python` in `pyproject.toml` | `py312, py313, py314`     |
| `basepython`       | Earliest supported Python version     | `py312`                   |
| `--python-version` | Latest supported Python version       | `3.14`                    |

### Step 4: Apply conventions

Write or modify the tox.ini following all conventions from this file and the loaded templates.

### Step 5: Verify compliance

Complete the verification checklist at the end of this file.

---

## Toolchain architecture

Projects use three tools together. Understanding their roles is essential for correct
tox.ini configuration.

### mamba (environment lifecycle)

Mamba creates and manages persistent conda environments that serve as the development workspace.
Each project has one named environment per OS (e.g., `axbu_dev_lin`, `axbu_dev_osx`,
`axbu_dev_win`). Mamba commands are issued through `automation-cli` and handle:
- Creating bare environments with Python + uv + tox + tox-uv
- Removing, exporting, and importing environments
- The `--use-uv` flag is passed to mamba for uv-accelerated operations

### uv (package installation)

uv replaces pip for all package installation operations. It is used in two contexts:
- **Inside mamba environments**: `automation-cli` calls `uv pip install` to install all project
  dependencies (runtime + dev) from `pyproject.toml` into the mamba environment.
- **Inside tox environments**: The `tox-uv` plugin makes tox use uv as its backend for creating
  isolated test environments, replacing pip with uv for speed.

### tox (task orchestration)

Tox orchestrates the development pipeline. Each tox environment is an **isolated virtual
environment** separate from the mamba environment. Running `tox` (no arguments) executes the
full `envlist` pipeline. Running `tox -e <name>` executes a single environment.

**Critical distinction:** Tox environments (lint, test, docs, etc.) are ephemeral and isolated.
The mamba environment is persistent. The `automation-cli` commands bridge the two worlds — tox
environments like `create`, `install`, and `export` call `automation-cli` to manipulate the
persistent mamba environment.

---

## `[tox]` section conventions

Every tox.ini starts with the `[tox]` section:

```ini
[tox]
requires =
    tox>=4,<5
    tox-uv>=1,<2
envlist =
    uninstall
    export
    lint
    stubs
    {py312, py313, py314}-test
    coverage
    docs
    build
    install
```

### Rules

- `requires` MUST include `tox>=4,<5` and `tox-uv>=1,<2`.
- `isolated_build` is NOT set. The key belongs to tox 3. Tox 4 always builds the project through
  its PEP 517 backend, so it resolves no such setting and silently ignores the key. Remove it from
  any tox.ini that still carries it.
- `envlist` defines the full pipeline order. Running bare `tox` executes all listed environments
  sequentially. The order matters — `uninstall` runs first to ensure a clean state, `install`
  runs last after all checks pass.
- Environment management environments (`create`, `remove`, `provision`, `import`) are defined
  in the file but NOT included in `envlist` because they are invoked manually.
- Section headers are written without a space after the colon, as `[testenv:lint]`. Tox strips the
  whitespace and resolves `[testenv: lint]` to the same environment, so the spaced form is a
  formatting variant rather than a defect. One spelling keeps the headers of a file uniform and keeps
  a grep for a header name reliable, and this is that spelling. Rewrite the spaced form when editing
  a file that carries it.

---

## Dependency installation patterns

### `dependency_groups = dev` (modern convention)

Tox 4.22+ supports PEP 735 dependency groups natively. This is the preferred approach for new
and modernized projects:

```ini
[testenv:lint]
dependency_groups = dev
```

This reads from `[dependency-groups].dev` in `pyproject.toml` and installs those packages before
the project itself. Use this for environments that need the project installed along with its dev
tools (lint, stubs, test).

### `extras = dev` (legacy convention)

Older projects use optional dependencies instead:

```ini
[testenv:lint]
extras = dev
```

This reads from `[project.optional-dependencies].dev` in `pyproject.toml`. This pattern is being
phased out in favor of `dependency_groups` as projects modernize.

### `deps = ataraxis-automation=={version}` (utility environments)

Environments that do NOT need the project installed — only the automation tools — use `deps`
with a pinned ataraxis-automation version:

```ini
[testenv:coverage]
skip_install = true
deps = ataraxis-automation==9.0.0
```

This pattern applies to: `coverage`, `build`, `upload`, `deploy`, `install`, `uninstall`,
`create`, `remove`, `provision`, `export`, `import`.

The Python `docs` environment carries the same pinned `deps` line but omits `skip_install`, because
Sphinx autodoc imports the installed project to read its docstrings. The C++ docs-only pipeline has
no project to install, so its `docs` environment sets `skip_install = true`.

### Self-hosting exception

The ataraxis-automation project itself does NOT use `deps = ataraxis-automation==X.Y.Z` because
it IS ataraxis-automation. Its utility environments omit `skip_install` and `deps`, relying on
the project's own installed tools instead.

---

## Envlist patterns by archetype

### Full Python pipeline

Used by core libraries (`ataraxis-*`) with multi-version testing:

```ini
envlist =
    uninstall
    export
    lint
    stubs
    {py312, py313, py314}-test
    coverage
    docs
    build
    install
```

### Reduced Python pipeline

Used by application projects that do not have unit tests. Their `lint` environment runs
`ruff check --fix ./src` without `./tests`, because ruff exits with `E902` on a path that does not
exist:

```ini
envlist =
    uninstall
    export
    lint
    stubs
    docs
    build
    install
```

### C++ docs-only pipeline

Used by PlatformIO projects (both libraries and firmware):

```ini
envlist = docs
```

---

## Environment conventions

### lint

- `basepython` MUST be set to the earliest supported Python version.
- Runs `automation-cli purge-stubs` first to remove stubs that interfere with mypy.
- Command order: `ruff format` → `ruff check --fix ./src ./tests` → `mypy ./src`. Ruff covers the
  test directory, which activates the `tests/**/*.py` per-file ignores that hold test code to a
  wider ignore list than library code. Mypy stays on `./src`, which its exclude list already
  matches. Reduced Python projects have no test directory and pass `./src` to ruff as well, since
  ruff exits with `E902` on a path that does not exist.
- `ruff format` stays bare. It formats the whole tree from the project root, which keeps test files
  formatted even in the archetypes whose `ruff check` is scoped to `./src` alone. A path added here
  narrows formatting to that path.
- Uses `dependency_groups = dev` (or `extras = dev` in legacy projects).
- `mypy ./src` runs single-threaded by default; keep it serial for typical libraries. For large
  codebases only (a cold `mypy` run takes several seconds), parallel checking via `-n N` is
  available — see [mypy parallelism](references/environment-templates.md) for the rules before
  enabling it.

### stubs

- `depends = lint` — stubs are generated only after linting passes.
- Runs `automation-cli process-typed-markers` first, then `stubgen -o stubs --include-private -p
  {package_name} -v`, followed by `automation-cli process-stubs`.
- After stub generation: `ruff format` → `ruff check --select I --fix ./src` to clean up stubs.

### test

- Uses parameterized names: `{py312, py313, py314}-test`.
- `package = wheel` forces a wheel build before testing.
- `setenv = COVERAGE_FILE = reports{/}.coverage.{envname}` writes per-version coverage data.
- Runs pytest with `--import-mode=importlib`, `--cov`, `--cov-config=pyproject.toml`, `-n logical`,
  `--dist loadgroup`.
- `--dist loadgroup` routes every test carrying the same `@pytest.mark.xdist_group` marker to one
  worker. See `/python-style` for the marker and the cases that require it.
- `--import-mode=importlib` matches the `addopts` declaration in `pyproject.toml`, so a bare `pytest`
  invocation resolves test modules the same way this task does. See `/pyproject-style`.

### coverage

- `skip_install = true` — only needs coverage tools, not the project.
- `depends` MUST list the same Python version matrix as the test environment.
- Merges junit XML reports, combines coverage data, generates XML and HTML reports, and applies the
  100% coverage gate.
- The `xml` and `html` commands pass `--fail-under=0` so both artifacts are always written. The
  trailing `coverage report` command applies the `fail_under = 100` gate declared in
  `pyproject.toml`. See `/pyproject-style` for the gate and for the `omit` list that keeps interface
  modules out of the measured corpus.
- The `xml`, `html`, and `report` commands each pass `--keep-combined` so that every command in the
  sequence receives the same set of per-version data files. Each reporting command performs its own
  implicit combine, and the flag preserves the data files it consumes. See `/pyproject-style` for
  the `[tool.coverage.paths]` mapping that merges those files into one record per source file.

### docs

- `depends = uninstall` — ensures a clean state.
- C++ and hybrid projects add `allowlist_externals = doxygen` and run `doxygen Doxyfile` before
  `sphinx-build`.
- Sphinx command MUST use `-j auto -v` flags.

### build

- `skip_install = true` — builds from source, not from installed package.
- Standard projects: `python -m build . --sdist` + `python -m build . --wheel`.
- C++ extension projects: `python -m build . --sdist` + `cibuildwheel --output-dir dist
  --platform auto`.
- `allowlist_externals = docker` for container-based builds.

### upload and deploy

- Both use `skip_install = true` and stay out of `envlist`, since a release invokes them manually.
- `upload` runs `automation-cli acquire-pypi-token` followed by `automation-cli upload-project`, and
  accepts `{posargs:}` for the `--replace-token` flag.
- `deploy` runs `automation-cli acquire-netlify-token` then `automation-cli deploy-docs`, accepting
  `{posargs:}` for `--replace-token` and `--replace-site`. Run `tox -e docs` first to build the html.
- Both API tokens live in a host-wide shared application directory, so each is entered once per
  host. The per-project Netlify site identifier lives in a tracked `.netlify-site` file at the root.
- `deploy` and `.netlify-site` are one unit. The environment reads the identifier from that file, so
  a project carries both or neither. A `deploy` environment without the file fails at the point it
  resolves the site, and a `.netlify-site` file without the environment names a site nothing
  publishes to. A project that builds documentation without hosting it keeps `docs` and drops both.
  See `/project-layout` for the file.

### Environment management (install, uninstall, create, remove, provision, export, import)

- All use `skip_install = true` and `deps = ataraxis-automation=={version}`.
- All call `automation-cli` subcommands with `--environment-name {env_abbr}_dev`.
- `create` and `provision` also pass `--python-version` set to the latest supported version.
- `create`, `provision`, and `install` accept `{posargs:}` to allow passing additional flags at
  invocation time (e.g., `--prerelease` to enable prerelease package installation).
- `export` has `depends = uninstall`.
- `install` has `depends` listing the full pipeline (lint, stubs, test, coverage, docs, export).
- These environments are defined in the file but NOT included in `envlist` (except `install`,
  `uninstall`, and `export`).

---

## Environment naming conventions

Each project has a short abbreviation used for its mamba environment name:

| Project type                       | Abbreviation rule                                         | Example                            |
|------------------------------------|-----------------------------------------------------------|------------------------------------|
| Multi-repository project component | Project abbreviation + the initial of each remaining word | `ataraxis-base-utilities` → `axbu` |
| Standalone project                 | The project name, used as-is                              | `harvester` → `harvester`          |

The full environment name follows the pattern `{abbr}_dev` (e.g., `axbu_dev` or `harvester_dev`). The OS suffix
(`_lin`, `_osx`, `_win`) is appended automatically by `automation-cli` at runtime — it does NOT
appear in tox.ini.

---

## Python version matrix

The test environment Python version matrix MUST match the `requires-python` range in
`pyproject.toml`:

| Project type | `requires-python` | Test matrix                  | `basepython` | `--python-version` |
|--------------|-------------------|------------------------------|--------------|--------------------|
| Core library | `>=3.12,<3.15`    | `{py312, py313, py314}-test` | `py312`      | `3.14`             |
| Application  | `>=3.14,<3.15`    | `{py314}-test`               | `py314`      | `3.14`             |

- `basepython` is set to the **earliest** supported version (controls lint/mypy ruleset).
- `--python-version` in `create`/`provision` is set to the **latest** supported version.

---

## Comment conventions

### Block comments

Use block comments above the `[tox]` section and before environments that need explanation:

```ini
# This file provides configurations for tox-based project development and management automation
# tasks.
```

### Description fields

Every environment MUST have a `description` field. Use multi-line format for descriptions longer
than 120 characters:

```ini
[testenv:lint]
description =
    Runs static code formatting, style, and typing checkers. Follows the configuration defined
    in the pyproject.toml file.
```

### Inline comments

Use inline comments sparingly, only when a setting is non-obvious:

```ini
basepython = py312  # Earliest supported version controls lint/mypy ruleset
```

### Comment and description restraint

A `description` field states what the environment does when it runs. It is one sentence, two when
the environment reads its configuration from another file. Do not restate the environment name, do
not list the commands the environment already declares below it, and do not explain why the task
matters.

```ini
# Avoid - restates the name, then narrates the commands
[testenv:lint]
description =
    The lint environment. This environment runs the linters. It first runs ruff to check formatting
    and style, then runs mypy to check types, and it is an important part of keeping the codebase
    consistent and maintainable for all contributors.

# Good
[testenv:lint]
description =
    Runs static code formatting, style, and typing checkers. Follows the configuration defined
    in the pyproject.toml file.
```

An inline comment earns its place only when the setting's name and value leave a question open,
such as a version pin whose reason is external or a flag whose effect is counter-intuitive.
Comments describe the configuration as it currently stands, never the edit that produced it.

### Prose punctuation and positive description

These rules apply to every description field, comment, and prose passage in the tox.ini and this skill.

Prose uses only the full stop and the comma to separate clauses. Do not use a semicolon or an em-dash (`--`, `—`, or
`–`) as a separator, and use a colon only where it is lexically appropriate. A single hyphen stays available as a
list marker, in tables, and in compound words. State what the subject does and what is currently true. Do not frame
it by what it is not or what it used to be, and keep a "not Y" contrast only when it is load-bearing because it
corrects a counter-intuitive assumption, giving its reason.

---

## Command reference

For the full list of tox environments and the underlying `automation-cli` commands (with options)
that the pipeline runs, see [Command reference](references/environment-templates.md). Agents drive
these via `tox -e <env>`; the `automation-cli` commands are documented there for diagnostics and
for answering user questions.

---

## Related skills

| Skill              | Relationship                                                    |
|--------------------|-----------------------------------------------------------------|
| `/pyproject-style` | Defines dependency groups, ruff ignore corpora, and mypy exclusions the lint task applies |
| `/api-docs`        | Defines docs/ structure built by the `docs` tox environment     |
| `/python-style`    | Defines code conventions enforced by the `lint` tox environment |
| `/project-layout`  | Defines project directory structure that tox.ini assumes        |
| `/commit`          | Should be invoked after completing tox.ini changes              |

---

## Proactive behavior

You should proactively offer to invoke this skill when:
- Creating a new project that needs a tox.ini
- The user asks about running tox, development automation, or the mamba/uv/tox toolchain
- Adding a new tox environment or modifying an existing pipeline
- Upgrading the ataraxis-automation version pin across downstream projects

---

## Verification checklist

You MUST verify your work against this checklist before submitting any tox.ini changes.

```text
tox.ini Style Compliance:

[tox] Section:
- [ ] requires includes tox>=4,<5 and tox-uv>=1,<2
- [ ] isolated_build is absent (a tox 3 key that tox 4 ignores)
- [ ] envlist order matches archetype pattern (uninstall → export → lint → ... → install)
- [ ] Environment management envs (create, remove, provision, import) NOT in envlist
- [ ] Section headers written as [testenv:name], with no space after the colon

Dependency Patterns:
- [ ] lint, stubs, test use dependency_groups = dev (or extras = dev for legacy projects)
- [ ] Utility envs use deps = ataraxis-automation=={version} (pinned, not range)
- [ ] Utility envs use skip_install = true, except the Python docs env, whose autodoc imports the project
- [ ] Self-hosting exception applied correctly (ataraxis-automation only)

Lint Environment:
- [ ] basepython set to earliest supported Python version
- [ ] Command order: purge-stubs → ruff format → ruff check --fix → mypy
- [ ] ruff check targets ./src ./tests, or ./src alone when the project has no tests/ directory
- [ ] ruff format takes no path argument, so it formats the whole tree
- [ ] mypy targets ./src (not . or other paths), and never ./tests
- [ ] mypy runs serial by default; -n/--num-workers added only for large projects with a measured speedup

Stubs Environment:
- [ ] depends = lint
- [ ] stubgen uses --include-private and -p {package_name}
- [ ] ruff cleanup runs after stubgen

Test Environment:
- [ ] Parameterized names match requires-python range
- [ ] package = wheel is set
- [ ] COVERAGE_FILE uses reports{/}.coverage.{envname}
- [ ] pytest uses --import-mode=importlib, --cov, -n logical, --dist loadgroup

Coverage Environment:
- [ ] depends matches the test environment Python version matrix
- [ ] Merges junit XML, combines coverage, generates xml and html with --fail-under=0
- [ ] A trailing coverage report command applies the 100% gate declared in pyproject.toml
- [ ] The xml, html, and report commands each pass --keep-combined

Docs Environment:
- [ ] depends = uninstall
- [ ] Doxygen runs before sphinx-build (C++ and hybrid projects only)
- [ ] allowlist_externals = doxygen (when Doxygen is used)
- [ ] sphinx-build uses -j auto -v flags

Build Environment:
- [ ] Standard projects use python -m build for sdist and wheel
- [ ] C++ extension projects use cibuildwheel for wheel

Upload and Deploy Environments:
- [ ] upload runs acquire-pypi-token then upload-project, and deploy runs acquire-netlify-token then deploy-docs
- [ ] Both accept {posargs:} for the credential replacement flags and stay out of envlist
- [ ] The deploy environment and a root .netlify-site file are both present or both absent

Environment Management:
- [ ] --environment-name uses correct abbreviation with _dev suffix
- [ ] --python-version set to latest supported version
- [ ] export depends on uninstall
- [ ] install depends on the full pipeline

Formatting:
- [ ] Every environment has a description field
- [ ] Descriptions state what the environment does in one or two sentences (no name restatement, no command narration)
- [ ] Every inline comment answers a question its setting leaves open
- [ ] Comments and descriptions record current configuration only, never the edit that produced it
- [ ] Block comments above [tox] section
- [ ] No duplicate environment definitions
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons and hyphen bullets fine)
- [ ] Prose states what the setting does, not what it is not or used to be (contrast only when load-bearing)
```
