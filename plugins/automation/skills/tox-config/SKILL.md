---
name: tox-config
description: >-
  Applies tox.ini conventions when creating or modifying tox configuration files. Covers the mamba + uv + tox toolchain,
  envlist patterns, environment definitions, dependency installation strategies, environment naming, and project
  archetype variations (full Python, reduced Python, C++ extension, C++ docs-only). Use when creating or modifying a
  tox.ini, changing tox environments, or when the user asks about tox configuration or the mamba/uv/tox toolchain.
user-invocable: false
---

# tox.ini style guide

Applies conventions for tox.ini configuration files that drive the development automation pipeline.

You MUST read this skill before creating or modifying any tox.ini file. You MUST verify your changes against the
checklist before submitting.

---

## Scope

**Covers:**
- `[tox]` section structure and `requires` conventions
- Envlist ordering and patterns for each project archetype
- Environment definitions (lint, stubs, test, coverage, docs, build, upload, deploy, install, uninstall, create, remove,
  provision, export, import)
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

Mamba owns the persistent per-project development environment, uv installs packages inside it, and tox runs each
pipeline task in its own isolated environment, so the tox environments named `create`, `install`, and `export` call
`automation-cli` to act on the mamba environment. See [Toolchain architecture](references/environment-templates.md) for
the division of labor in full.

Determine which pipeline applies:

| Archetype      | Envlist pattern                                                              | Key indicator                          |
|----------------|------------------------------------------------------------------------------|----------------------------------------|
| Full Python    | uninstall → export → lint → stubs → test → coverage → docs → build → install | `pyproject.toml` + `src/` layout       |
| C++ extension  | Same as full Python, with Doxygen in docs and cibuildwheel in build          | `CMakeLists.txt` + `pyproject.toml`    |
| Reduced Python | Full Python minus test and coverage                                          | Application project with no unit tests |
| C++ docs-only  | docs only                                                                    | `platformio.ini`, no `pyproject`       |

### Step 2: Load reference templates

Read [environment-templates.md](references/environment-templates.md) for the complete envlist and environment
definitions matching the identified archetype.

### Step 3: Determine parameterization

Collect the project-specific values:

| Parameter          | Source                                | Example                   |
|--------------------|---------------------------------------|---------------------------|
| `{package_name}`   | Package directory name under `src/`   | `ataraxis_base_utilities` |
| `{env_abbr}`       | Short project abbreviation            | `axbu`                    |
| `{version}`        | Current ataraxis-automation release   | `9.0.2`                   |
| Python versions    | `requires-python` in `pyproject.toml` | `py312, py313, py314`     |
| `basepython`       | Earliest supported Python version     | `py312`                   |
| `--python-version` | Latest supported Python version       | `3.14`                    |

### Step 4: Apply conventions

Write or modify the tox.ini following all conventions from this file and the loaded templates.

### Step 5: Verify compliance

Complete the verification checklist at the end of this file.

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
- `isolated_build` is absent, and any tox.ini that still carries this tox 3 key drops it, since tox 4 always builds
  through its PEP 517 backend and resolves no such setting.
- `envlist` defines the full pipeline order. Running bare `tox` executes all listed environments sequentially, with
  `uninstall` first to ensure a clean state and `install` last after all checks pass.
- Environment management environments (`create`, `remove`, `provision`, `import`) are defined in the file but NOT
  included in `envlist` because they are invoked manually, while `install`, `uninstall`, and `export` are pipeline
  members and do appear in it.
- Section headers are written without a space after the colon, as `[testenv:lint]`, which tox resolves identically to
  the spaced form, so rewrite the spaced form when editing a file that carries it.
- Sections appear in a mandatory file order: `[tox]`, the optional `[testenv]` base section, then the testenv sections
  in the order [environment-templates.md](references/environment-templates.md) presents them (lint, stubs, test,
  coverage, docs, build, upload, deploy, install, uninstall, create, remove, provision, export, import). An archetype
  omits the environments it does not define and inserts a new environment at its position in this list rather than
  appending it.

---

## Dependency installation patterns

### `dependency_groups = dev`

Tox 4.22+ supports PEP 735 dependency groups natively. This is the form a new or edited tox.ini uses:

```ini
[testenv:lint]
dependency_groups = dev
```

This reads from `[dependency-groups].dev` in `pyproject.toml` and installs those packages before the project itself. Use
this for environments that need the project installed along with its dev tools (lint, stubs, test).

### `extras = dev`

Some projects declare their dev tools as optional dependencies instead, and `extras = dev` reads them from
`[project.optional-dependencies].dev` in `pyproject.toml`. A tox.ini written or edited under this skill uses
`dependency_groups = dev`.

### `deps = ataraxis-automation=={version}` (utility environments)

Environments that need the automation tools alone, without the project installed, use `deps` with a pinned
ataraxis-automation version:

```ini
[testenv:coverage]
skip_install = true
deps = ataraxis-automation==9.0.2
```

This pattern applies to: `coverage`, `build`, `upload`, `deploy`, `install`, `uninstall`, `create`, `remove`,
`provision`, `export`, `import`.

Every utility environment carries the identical pin, and that version falls inside the `ataraxis-automation` bound in
`[dependency-groups].dev` of `pyproject.toml`. An upgrade re-pins every environment and moves the pyproject bound in the
same change.

The Python `docs` environment carries the same pinned `deps` line but omits `skip_install`, because Sphinx autodoc
imports the installed project to read its docstrings. The C++ docs-only pipeline has no project to install, so its
`docs` environment sets `skip_install = true`.

### Self-hosting exception

The ataraxis-automation project itself does NOT use `deps = ataraxis-automation==X.Y.Z` because it IS
ataraxis-automation. Its utility environments omit `skip_install` and `deps`, relying on the project's own installed
tools instead.

---

## Environment conventions

### lint

- `basepython` MUST be set to the earliest supported Python version.
- Runs `automation-cli purge-stubs` first to remove stubs that interfere with mypy.
- Command order: `ruff format` → `ruff check --fix ./src ./tests` → `mypy ./src`. Ruff covers the test directory, which
  activates the `tests/**/*.py` per-file ignores (see `/pyproject-style`). Mypy stays on `./src`, which its exclude list
  already matches. Reduced Python projects have no test directory and pass `./src` to ruff as well, since ruff exits
  with `E902` on a path that does not exist.
- `ruff format` stays bare, so it formats the whole tree from the project root. See
  [environment-templates.md](references/environment-templates.md) for what a path added here narrows.
- Uses `dependency_groups = dev` (or `extras = dev` in legacy projects).
- `mypy ./src` runs serial. See [mypy parallelism](references/environment-templates.md) for the large-codebase exception
  and the measurement it requires.

### stubs

- `depends = lint`, so stubs are generated only after linting passes.
- Runs `automation-cli process-typed-markers` first, then `stubgen -o stubs --include-private -p {package_name} -v`,
  followed by `automation-cli process-stubs`.
- After stub generation: `ruff format` → `ruff check --select I --fix ./src` to clean up stubs.

### test

- Uses parameterized names: `{py312, py313, py314}-test`.
- `package = wheel` forces a wheel build before testing.
- `setenv = COVERAGE_FILE = reports{/}.coverage.{envname}` writes per-version coverage data.
- Runs pytest with `--import-mode=importlib`, `--cov`, `--cov-config=pyproject.toml`, `-n logical`, `--dist loadgroup`.
- `--cov` carries the package name as `--cov={package_name}`, or stays bare when `pyproject.toml` declares
  `source_pkgs` under `[tool.coverage.run]`, since a spawned worker reads the measured package from that file rather
  than from the command line.
- `--dist loadgroup` routes every test carrying the same `@pytest.mark.xdist_group` marker to one worker. See
  `/python-style` for the marker and the cases that require it.
- `--import-mode=importlib` matches the `addopts` declaration in `pyproject.toml`, so a bare `pytest` invocation
  resolves test modules the same way this task does. See `/pyproject-style`.

### coverage

- `skip_install = true`, since the environment needs the coverage tools alone.
- `depends` MUST list the same Python version matrix as the test environment.
- Merges junit XML reports, combines coverage data, generates XML and HTML reports, and applies the 100% coverage gate.
- The `xml` and `html` commands pass `--fail-under=0` so both artifacts are always written, and the trailing `coverage
  report` command applies the `fail_under = 100` gate declared in `pyproject.toml`.
- The `xml`, `html`, and `report` commands each pass `--keep-combined`, so every command in the sequence receives the
  same set of per-version data files. See [environment-templates.md](references/environment-templates.md) for the
  retention mechanism, and `/pyproject-style` for the gate, the `omit` list, and the `[tool.coverage.paths]` mapping.

### docs

- `depends = uninstall`, which ensures a clean state.
- C++ and hybrid projects add `allowlist_externals = doxygen` and run `doxygen Doxyfile` before `sphinx-build`.
- Sphinx command MUST use `-j auto -v` flags.

### build

- `skip_install = true`, since the build runs from source rather than from the installed package.
- Standard projects: `python -m build . --sdist` + `python -m build . --wheel`.
- C++ extension projects: `python -m build . --sdist` + `cibuildwheel --output-dir dist --platform auto`.
- `allowlist_externals = docker` for container-based builds.

### upload and deploy

- Both use `skip_install = true` and stay out of `envlist`, since a release invokes them manually.
- `upload` runs `automation-cli acquire-pypi-token` followed by `automation-cli upload-project`, and accepts
  `{posargs:}` for the `--replace-token` flag.
- `deploy` runs `automation-cli acquire-netlify-token` then `automation-cli deploy-docs`, accepting `{posargs:}` for
  `--replace-token` and `--replace-site`. Run `tox -e docs` first to build the html.
- Both API tokens live in a host-wide shared application directory, so each is entered once per host. The per-project
  Netlify site identifier lives in a tracked `.netlify-site` file at the root.
- `deploy` and `.netlify-site` are one unit, so a project carries both or neither. See
  [environment-templates.md](references/environment-templates.md) for how each half fails without the other, and
  `/project-layout` for the file.

### Environment management (install, uninstall, create, remove, provision, export, import)

- All use `skip_install = true` and `deps = ataraxis-automation=={version}`.
- All call `automation-cli` subcommands with `--environment-name {env_abbr}_dev`.
- `create` and `provision` also pass `--python-version` set to the latest supported version.
- `create`, `provision`, and `install` accept `{posargs:}` to allow passing additional flags at invocation time (e.g.,
  `--prerelease` to enable prerelease package installation).
- `export` has `depends = uninstall`.
- `install` has `depends` listing the full pipeline (lint, stubs, test, coverage, docs, export).

---

## Environment naming conventions

The mamba environment name follows the pattern `{abbr}_dev`. For a multi-repository project component, `{abbr}` is the
project abbreviation plus the initial of each remaining word (`ataraxis-base-utilities` becomes `axbu`). For a
standalone project, `{abbr}` is the project name used as-is (`harvester` becomes `harvester`). `automation-cli` appends
the OS suffix (`_lin`, `_osx`, `_win`) at runtime, so the suffix stays out of tox.ini.

---

## Python version matrix

The test environment matrix MUST list every version in the `requires-python` range declared in `pyproject.toml`.
`basepython` is the earliest of them and controls the lint and mypy ruleset. The `--python-version` passed by `create`
and `provision` is the latest of them.

---

## Comment conventions

### Block comments

Use block comments above the `[tox]` section and before environments that need explanation. See
[environment-templates.md](references/environment-templates.md) for the block comment each template carries.

### Description fields

Every environment MUST have a `description` field, and that field opens with a bare third-person imperative verb naming
what the environment does when it runs ("Runs...", "Combines...", "Builds..."). It is one sentence, two when
the environment reads its configuration from another file or produces one output the commands block does not reveal,
and three when it produces more than one such output, as a coverage task producing both a merged test report and a
coverage report does. Do not prefix it
with the environment name or a "This environment..." opener, do not restate the commands the environment already
declares below it, and do not explain why the task matters. See
[environment-templates.md](references/environment-templates.md) for a compliant and a non-compliant description side by
side.

A description whose `description = <text>` line would pass 120 characters moves to the indented form below, where each
line fills to 120 before it breaks:

```ini
[testenv:lint]
description =
    Runs static code formatting, style, and typing checkers. Follows the configuration defined in the pyproject.toml
    file.
```

### Inline comments

Use inline comments sparingly, only when a setting is non-obvious. Align inline comments vertically within a section, so
a run of commented settings shares one comment column:

```ini
basepython = py312  # Earliest supported version controls lint/mypy ruleset
```

### Comment restraint

An inline comment earns its place only when the setting's name and value leave a question open, such as a version pin
whose reason is external or a flag whose effect is counter-intuitive. Comments describe the configuration as it
currently stands, never the edit that produced it.

### Comment accuracy, length, and width

A comment's claim must be true of the key it sits on as it currently reads, covering the value, the bound, and the
effect tox actually produces. A comment naming a version, a bound, a path, or a coupled file is rewritten or deleted in
the same edit that moves that value. A comment must not reference a removed key, section, or environment, a closed
issue, a superseded tool version, or an outdated TODO. Sentences over 40 words are difficult to parse and MUST be broken
into smaller sentences at natural clause boundaries. Every comment and `description` field must be free of typos and
grammatical errors. Every line stays under the **120 character limit**, matching the `line-length` the project sets for
its Python code, with a single unbreakable value such as a URL or a requirement string exempt.

Break a comment or `description` line only where it would otherwise pass 120 characters, and fill each line to that
limit before breaking. Prose wrapped at a narrower width reads as a rigid block and advertises a limit the file does not
set. The test is mechanical: a wrapped line that ends before column 100 while its next word would still fit under 120 is
re-flowed. A line ending early because the sentence or the comment block ends is already correct.

### Prose punctuation and positive description

These rules apply to every description field, comment, and prose passage in the tox.ini.

Prose uses only the full stop and the comma to separate clauses. Do not use a semicolon or an em-dash (`--`, `—`, or
`–`) as a separator, and use a colon only where it is lexically appropriate. A single hyphen stays available as a list
marker, in tables, and in compound words. State what the setting does and what is currently true. Do not frame it by
what it is not or what it used to be, and keep a "not Y" contrast only when it is load-bearing because it corrects a
counter-intuitive assumption, giving its reason.

---

## Command reference

For the full list of tox environments and the underlying `automation-cli` commands (with options) that the pipeline
runs, see [Command reference](references/environment-templates.md). Agents drive these via `tox -e <env>`. The
`automation-cli` commands are documented there for diagnostics and for answering user questions.

---

## Related skills

| Skill              | Relationship                                                                              |
|--------------------|-------------------------------------------------------------------------------------------|
| `/pyproject-style` | Defines dependency groups, ruff ignore corpora, and mypy exclusions the lint task applies |
| `/api-docs`        | Defines docs/ structure built by the `docs` tox environment                               |
| `/python-style`    | Defines code conventions enforced by the `lint` tox environment                           |
| `/project-layout`  | Defines project directory structure that tox.ini assumes                                  |
| `/commit`          | Should be invoked after completing tox.ini changes                                        |

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

Settled by `tox -l`:
- [ ] envlist order matches archetype pattern (uninstall → export → lint → ... → install)
- [ ] Environment management envs (create, remove, provision, import) NOT in envlist
- [ ] No duplicate environment definitions

Judgment items. No tool inspects these, so this checklist is their only enforcement. Walk every one
against the file you wrote. The three `tox -l` rows above are the only tool-settled ones.

[tox] Section:
- [ ] requires includes tox>=4,<5 and tox-uv>=1,<2
- [ ] isolated_build is absent (a tox 3 key that tox 4 ignores)
- [ ] Section headers written as [testenv:name], with no space after the colon
- [ ] Section order is [tox], optional [testenv], then the testenv sections in reference-template order
      (lint, stubs, test, coverage, docs, build, upload, deploy, install, uninstall, create, remove,
      provision, export, import)

Dependency Patterns:
- [ ] lint, stubs, test use dependency_groups = dev (or extras = dev for legacy projects)
- [ ] Utility envs use deps = ataraxis-automation=={version} (pinned, not range)
- [ ] Utility envs use skip_install = true, except the Python docs env, whose autodoc imports the project
- [ ] Self-hosting exception applied correctly (ataraxis-automation only)
- [ ] Every deps = ataraxis-automation==X.Y.Z pin is the same version, and that version falls inside the
      ataraxis-automation bound in pyproject.toml [dependency-groups].dev

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
- [ ] --cov names the package, or stays bare when pyproject.toml declares source_pkgs under [tool.coverage.run]

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
- [ ] create, provision, and install accept {posargs:} for invocation-time flags such as --prerelease
- [ ] export depends on uninstall
- [ ] install depends on the full pipeline

Formatting:
- [ ] Every environment has a description field
- [ ] Every description opens with a bare third-person imperative verb and states what the environment does in one or
      three sentences (no name restatement, no "This environment..." opener, no command narration)
- [ ] Every inline comment answers a question its setting leaves open
- [ ] Inline comments aligned vertically within their section
- [ ] Comments and descriptions record current configuration only, never the edit that produced it
- [ ] Every comment's claim is true of the value it sits beside and of the effect the tool actually produces
- [ ] No stale references in comments (closed issues, removed keys or environments, superseded tool
      versions, outdated TODOs)
- [ ] Sentences in comments and description fields stay under 40 words
- [ ] Comments and description fields free of typos and grammar errors
- [ ] Lines stay under 120 characters, with unbreakable single values (URLs, requirement strings) exempt
- [ ] Comments and descriptions fill each line to 120 characters, with no line ending before column 100 while its next
      word would still fit
- [ ] Block comments above [tox] section
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons, hyphen bullets, and code
      syntax exempt)
- [ ] Prose states what the setting does, not what it is not or used to be (contrast only when load-bearing)
```
