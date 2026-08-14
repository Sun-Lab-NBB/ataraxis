# Environment templates

Complete tox.ini environment templates for each project archetype. All templates use `dependency_groups = dev`, which
tox 4.22+ supports natively. Projects that declare their dev tools under `[project.optional-dependencies]` read them
with `extras = dev` instead, as described in the dependency installation patterns of [the skill](../SKILL.md).

---

## Toolchain architecture

Projects use three tools together.

### mamba (environment lifecycle)

Mamba creates and manages persistent conda environments that serve as the development workspace. Each project has one
named environment per OS (e.g., `axbu_dev_lin`, `axbu_dev_osx`, `axbu_dev_win`). Mamba commands are issued through
`automation-cli` and handle:
- Creating bare environments with Python + uv + tox + tox-uv
- Removing, exporting, and importing environments
- Passing the `--use-uv` flag to mamba for uv-accelerated operations

### uv (package installation)

uv replaces pip for all package installation operations. It is used in two contexts:
- **Inside mamba environments**: `automation-cli` calls `uv pip install` to install all project dependencies (runtime +
  dev) from `pyproject.toml` into the mamba environment.
- **Inside tox environments**: The `tox-uv` plugin makes tox use uv as its backend for creating isolated test
  environments, replacing pip with uv for speed.

### tox (task orchestration)

Tox orchestrates the development pipeline. Each tox environment is an **isolated virtual environment** separate from the
mamba environment. Running `tox` (no arguments) executes the full `envlist` pipeline. Running `tox -e <name>` executes a
single environment.

**Critical distinction:** Tox environments (lint, test, docs, etc.) are ephemeral and isolated, while the mamba
environment is persistent. The `automation-cli` commands bridge the two worlds, so tox environments like `create`,
`install`, and `export` call `automation-cli` to manipulate the persistent mamba environment.

---

## Full Python pipeline

Canonical template for Python-only and Python + C++ extension projects. The ataraxis-automation project itself is the
root source of truth for this pipeline.

### `[tox]` section

```ini
# This file provides configurations for tox-based project development and management automation tasks.

# Base tox configurations. Note, the 'envlist' runs in the listed order whenever 'tox' is used without an -e specifier.
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

### `[testenv]` base section (optional)

Use this section when any environment needs prerelease packages. When present, it applies to all environments that do
not override `setenv`.

```ini
# Allows installing prerelease packages.
[testenv]
setenv =
    UV_PRERELEASE = allow
```

tox does not merge `setenv`. An env that defines its own `setenv` block, as test and coverage both do for
`COVERAGE_FILE`, fully replaces the inherited base `setenv` rather than extending it. So a project that needs
prereleases during testing re-declares `UV_PRERELEASE = allow` inside the test env `setenv` alongside `COVERAGE_FILE`.

### lint environment

```ini
# Note: The 'basepython' argument should always be set to the earliest supported Python version. The 'ruff check'
# command also covers the test directory, which activates the 'tests/**/*.py' per-file ignores configured in the
# pyproject.toml file. The 'mypy' command stays on the source directory, as the test directory is part of the mypy
# exclusion list.
[testenv:lint]
description =
    Runs static code formatting, style, and typing checkers. Follows the configuration defined in the pyproject.toml
    file.
dependency_groups = dev
basepython = py312
commands =
    automation-cli purge-stubs
    ruff format
    ruff check --fix ./src ./tests
    mypy ./src
```

**Parameterization:**
- `basepython`: Set to the earliest Python version in the supported range.
- `dependency_groups`: Always `dev`. Installs the project's dev dependency group (tox, uv, tox-uv, ataraxis-automation,
  and type stubs). The `ataraxis-automation` entry is what puts `automation-cli`, `ruff`, `mypy`, and `stubgen` on the
  path of the lint and stubs environments, so omitting it breaks both on their first run.
- `ruff check` paths: `./src ./tests` for a project that has a `tests/` directory, which is every archetype except
  Reduced Python. A project without a test directory passes `./src` alone. Ruff reports `E902 No such file or directory`
  and exits non-zero when a path on its command line does not exist, so a blanket `./tests` breaks the lint task of a
  project that has no tests.
- `ruff format` takes no path and formats the whole tree from the project root, which is what keeps test files formatted
  in every archetype including the ones whose `ruff check` is scoped to `./src`. Adding a path here narrows formatting
  to that path, so leave the command bare.

**Linting scope.** Passing `./tests` to `ruff check` is what activates the `tests/**/*.py` per-file-ignores key. A lint
task that checks `./src` alone leaves the key inert, so the test directory goes unlinted while the project's
configuration claims otherwise. See `/pyproject-style` for that key and the shared test corpus it carries.

`mypy` stays on `./src` in every archetype, because the test directory is already a member of the `[tool.mypy]` exclude
list. See `/pyproject-style` for that list.

**Self-hosting exception (ataraxis-automation only):** Since ataraxis-automation IS the automation provider, its lint
environment does not need `deps = ataraxis-automation==X.Y.Z`. It uses `dependency_groups = dev` (or `extras = dev` in
legacy form) to get its own dev dependencies.

#### mypy parallelism (large projects only)

`mypy ./src` runs single-threaded, which is correct for every ataraxis library and application project, since their
binary and SQLite incremental caches (on by default since mypy 2.0) already make re-runs effectively free. You MUST NOT
add parallelism to a project's lint command by default.

mypy 2.x adds experimental parallel type checking through `-n N` or `--num-workers N`, the `num_workers` config-file
key, and the `MYPY_NUM_WORKERS` environment override. Add it only to a project whose cold `mypy ./src` takes more than a
few seconds, and only after `time mypy ./src` with and without `-n` on a cleared `.mypy_cache` shows a measured win. Set
`N` to the CI runner's physical core count. Parallel mode implicitly enables `--native-parser`, may produce minor
semantic differences from serial mode, and slows warm incremental runs through worker-startup overhead, so it pays off
on fresh CI checkouts alone.

### stubs environment

```ini
[testenv:stubs]
description =
    Generates the py.typed marker and the .pyi stub files using the project's sdist distribution.
depends = lint
dependency_groups = dev
commands =
    automation-cli process-typed-markers
    stubgen -o stubs --include-private -p {package_name} -v
    automation-cli process-stubs
    ruff format
    ruff check --select I --fix ./src
```

**Parameterization:**
- `{package_name}`: The underscore-separated Python package name (e.g., `ataraxis_base_utilities`)

This pass stays on `./src` in every archetype. It sorts the imports of the `.pyi` files that `automation-cli
process-stubs` writes into the source tree, and stub generation writes nothing under `./tests`. The lint environment
already sorts the test directory's imports as part of its full check.

### test environment

```ini
[testenv:{py312, py313, py314}-test]
package = wheel
description =
    Runs unit and integration tests for each of the python versions listed in the task name and aggregates test
    coverage data. Uses 'loadgroup' balancing and all logical cores to optimize task runtime speed.
dependency_groups = dev
setenv =
    COVERAGE_FILE = reports{/}.coverage.{envname}
    # Re-declare here only if the project needs prereleases during testing. This setenv block fully replaces the base
    # [testenv] setenv (tox does not merge).
    # UV_PRERELEASE = allow
commands =
    pytest --import-mode=importlib --cov={package_name} --cov-config=pyproject.toml \
    --cov-report=xml --junitxml=reports/pytest.xml.{envname} -n logical --dist loadgroup
```

**Parameterization:**
- Python version matrix `{py312, py313, py314}`: Must match the `requires-python` range in `pyproject.toml`. Core
  libraries (`ataraxis-*`) test 3 versions, and applications may test fewer.
- `{package_name}` in `--cov`: The underscore-separated package name. A project that declares `source_pkgs` under
  `[tool.coverage.run]` in `pyproject.toml` passes a bare `--cov` instead. Spawned worker processes read the measured
  package from that config file rather than from the command line, so the name belongs in exactly one place.
- `package = wheel`: Forces the project to be built as a wheel before testing.
- `--import-mode=importlib`: Matches the `addopts` declaration in `pyproject.toml`, so a bare `pytest` invocation
  resolves test modules the same way this task does. See `/pyproject-style` for that declaration.

### coverage environment

```ini
# Note: the 'xml' and 'html' commands run with '--fail-under=0' so that both reports are always written, the html
# report to the 'reports' directory and the xml report to the project root. The trailing 'report' command applies the
# 100% coverage gate configured in the pyproject.toml file and prints the statements that remain uncovered. Every
# reporting command also combines any data files it finds, so all of them run with '--keep-combined' to preserve the
# per-version data files consumed by the 'combine' command. Without the flag, the first reporting command deletes those
# files and the task only succeeds once per test run.
[testenv:coverage]
skip_install = true
description =
    Combines test-coverage data from multiple test runs (for different python versions) into a single html file and
    verifies that the combined data covers 100% of the measured statements. The file can be viewed by loading the
    'reports/coverage_html/index.html'. The task also merges the per-version JUnit test-result reports into
    'reports/pytest.xml' and writes an xml coverage report to the project root.
deps = ataraxis-automation=={version}
setenv = COVERAGE_FILE = reports/.coverage
depends = {py312, py313, py314}-test
commands =
    junitparser merge --glob reports/pytest.xml.* reports/pytest.xml
    coverage combine --keep
    coverage xml --fail-under=0 --keep-combined
    coverage html --fail-under=0 --keep-combined
    coverage report --keep-combined
```

**Parameterization:**
- `deps = ataraxis-automation=={version}`: Pin to the exact current release version.
- `depends`: Must list the same Python version matrix as the test environment.

**Coverage gate:** The `xml` and `html` commands pass `--fail-under=0` so both artifacts are always written, and the
trailing `coverage report` command applies the gate. See `/pyproject-style` for the `fail_under` setting, the `omit`
list that keeps interface modules out of the measured corpus, and the `# pragma: no cover` marker for individual
unreachable statements.

**Data file retention:** `coverage combine --keep` writes the combined record while retaining the per-version data
files. Each reporting command that follows performs its own implicit combine, so `xml`, `html`, and `report` all carry
`--keep-combined` to hand the same set of data files to the command after them. See `/pyproject-style` for the
`[tool.coverage.paths]` mapping that merges those files into one record per source file. The flag requires coverage 7.15
or newer, which every project receives through the pinned ataraxis-automation dependency of this environment.

**Self-hosting exception:** ataraxis-automation omits `skip_install` and `deps` since it provides these tools itself.

### docs environment

**Python-only projects** (no Doxygen):

```ini
[testenv:docs]
description =
    Builds the API documentation from source code docstrings using Sphinx. The result can be viewed by loading
    'docs/build/html/index.html'.
deps = ataraxis-automation=={version}
depends = uninstall
commands =
    sphinx-build -b html -d docs/build/doctrees docs/source docs/build/html -j auto -v
```

**Python + C++ extension projects** (with Doxygen):

```ini
[testenv:docs]
description =
    Builds the API documentation from source code docstrings using Doxygen, Breathe and Sphinx. The result can be
    viewed by loading 'docs/build/html/index.html'.
deps = ataraxis-automation=={version}
depends = uninstall
allowlist_externals = doxygen
commands =
    doxygen Doxyfile
    sphinx-build -b html -d docs/build/doctrees docs/source docs/build/html -j auto -v
```

**Parameterization:**
- Add `allowlist_externals = doxygen` and the `doxygen Doxyfile` command only for projects that have a `Doxyfile` at the
  project root.
- `depends = uninstall`: Ensures a clean environment state before building.

### build environment

**Standard Python projects:**

```ini
[testenv:build]
skip_install = true
description =
    Builds the project's source code distribution (sdist) and binary distribution (wheel), clearing the 'dist'
    directory beforehand so that artifacts built for an earlier version cannot be carried into the upload task.
deps = ataraxis-automation=={version}
commands =
    python -c "import shutil; shutil.rmtree('dist', ignore_errors=True)"
    python -m build . --sdist
    python -m build . --wheel
```

**C++ extension projects** (cibuildwheel):

```ini
[testenv:build]
skip_install = true
description =
    Builds the project's source code distribution (sdist) and binary distribution (wheel), clearing the 'dist'
    directory beforehand so that artifacts built for an earlier version cannot be carried into the upload task.
deps = ataraxis-automation=={version}
allowlist_externals = docker
commands =
    python -c "import shutil; shutil.rmtree('dist', ignore_errors=True)"
    python -m build . --sdist
    cibuildwheel --output-dir dist --platform auto
```

### upload environment

```ini
# Note: use 'tox -e upload -- --replace-token' command to replace the token stored in the shared .pypirc file before
# uploading the project.
[testenv:upload]
skip_install = true
description =
    Uses twine to upload the wheel ('*.whl') and source ('*.tar.gz') distributions found inside the
    project's 'dist' directory to PyPI.
deps = ataraxis-automation=={version}
commands =
    automation-cli acquire-pypi-token {posargs:}
    automation-cli upload-project
```

The PyPI API token is stored in a `.pypirc` file inside a host-wide shared application directory resolved with
`platformdirs`, so every project managed on the host reuses the same token.

### deploy environment

```ini
# Note: use 'tox -e deploy -- --replace-token' command to replace the token stored in the shared .netlifyrc file, and
# 'tox -e deploy -- --replace-site' to replace the site identifier stored in the project's .netlify-site file, before
# deploying the documentation.
[testenv:deploy]
skip_install = true
description =
    Uploads the API documentation built by the 'docs' task to the project's Netlify site. Build the documentation with
    'tox -e docs' before calling this task.
deps = ataraxis-automation=={version}
commands =
    automation-cli acquire-netlify-token {posargs:}
    automation-cli deploy-docs
```

The Netlify API token is stored in a `.netlifyrc` file inside the same shared application directory as the PyPI token.
The site identifier differs for each project and is not a secret, so it lives in a `.netlify-site` file at the project
root that is tracked by version control. The deployment uploads `docs/build/html` as a ZIP archive, and Netlify replaces
the whole site with its contents.

The `deploy` environment and the `.netlify-site` file are one unit. The environment reads the identifier from that file,
so a project carries both or neither. A `deploy` environment without the file fails at the point it resolves the site,
and a `.netlify-site` file without the environment names a site nothing publishes to. A project that builds
documentation without hosting it keeps `docs` and drops both. See `/project-layout` for the file.

Both `upload` and `deploy` are defined in the tox.ini but stay out of `envlist`, since they are invoked manually as part
of a release.

### install environment

```ini
[testenv:install]
skip_install = true
deps = ataraxis-automation=={version}
depends =
    lint
    stubs
    {py312, py313, py314}-test
    coverage
    docs
    export
description =
    Builds and installs the project into its development mamba environment.
commands =
    automation-cli install-project --environment-name {env_abbr}_dev {posargs:}
```

**Parameterization:**
- `depends`: Must list the complete pipeline that should pass before final installation.
- `{env_abbr}_dev`: The project's environment abbreviation (e.g., `axbu_dev`).
- `{posargs:}`: Allows passing additional flags at invocation time (e.g., `--prerelease` to enable prerelease package
  installation).

### uninstall environment

```ini
[testenv:uninstall]
skip_install = true
deps = ataraxis-automation=={version}
description =
    Uninstalls the project from its development mamba environment.
commands =
    automation-cli uninstall-project --environment-name {env_abbr}_dev
```

### create environment

```ini
[testenv:create]
skip_install = true
deps = ataraxis-automation=={version}
description =
    Creates the project's development mamba environment using the requested python version and installs runtime and
    development project dependencies extracted from the pyproject.toml file.
commands =
    automation-cli create-environment --environment-name {env_abbr}_dev --python-version 3.14 {posargs:}
```

**Parameterization:**
- `--python-version`: Set to the latest Python version in the supported range.
- `{posargs:}`: Allows passing additional flags at invocation time (e.g., `--prerelease` to enable prerelease package
  installation).

### remove environment

```ini
[testenv:remove]
skip_install = true
deps = ataraxis-automation=={version}
description =
    Removes the project's development mamba environment.
commands =
    automation-cli remove-environment --environment-name {env_abbr}_dev
```

### provision environment

```ini
[testenv:provision]
skip_install = true
deps = ataraxis-automation=={version}
description =
    Provisions the project's development mamba environment by verifying that the new environment specification
    resolves, then removing and (re)creating the environment and installing the project dependencies into it.
commands =
    automation-cli provision-environment --environment-name {env_abbr}_dev --python-version 3.14 {posargs:}
```

### export environment

```ini
[testenv:export]
skip_install = true
deps = ataraxis-automation=={version}
depends = uninstall
description =
    Exports the project's development mamba environment to the 'envs' project directory as a .yml file.
commands =
    automation-cli export-environment --environment-name {env_abbr}_dev
```

### import environment

```ini
[testenv:import]
skip_install = true
deps = ataraxis-automation=={version}
description =
    Creates or updates the project's development mamba environment using the .yml file stored in the 'envs' project
    directory.
commands =
    automation-cli import-environment --environment-name {env_abbr}_dev
```

---

## C++ docs-only pipeline

Template for pure C++ PlatformIO projects (libraries and firmware). These projects have only a `docs` environment
because they are not Python packages.

```ini
# This file provides configurations for tox-based project development and management automation tasks.

# Base tox configurations.
[tox]
requires =
    tox>=4,<5
    tox-uv>=1,<2
envlist = docs

[testenv:docs]
skip_install = true
description =
    Builds the API documentation from source code docstrings using Doxygen, Breathe and Sphinx. The result can be
    viewed by loading 'docs/build/html/index.html'.
deps = ataraxis-automation=={version}
allowlist_externals = doxygen
commands =
    doxygen Doxyfile
    sphinx-build -b html -d docs/build/doctrees docs/source docs/build/html -j auto -v

# Note: use 'tox -e deploy -- --replace-token' command to replace the token stored in the shared .netlifyrc file, and
# 'tox -e deploy -- --replace-site' to replace the site identifier stored in the project's .netlify-site file, before
# deploying the documentation.
[testenv:deploy]
skip_install = true
description =
    Uploads the API documentation built by the 'docs' task to the project's Netlify site. Build the documentation with
    'tox -e docs' before calling this task.
deps = ataraxis-automation=={version}
commands =
    automation-cli acquire-netlify-token {posargs:}
    automation-cli deploy-docs
```

**Key differences from the Python pipeline:**
- `envlist = docs`, a single environment. `deploy` stays out of `envlist`, as it does in every archetype.
- `skip_install = true`, since there is no Python package to install.
- Always runs `doxygen Doxyfile` before `sphinx-build`.
- No lint, stubs, test, coverage, build, upload, or environment management environments.
- `deploy` is present wherever the project hosts its documentation, paired with a `.netlify-site` file.

---

## Reduced Python pipeline

Some Python application projects omit test and coverage environments from their envlist, typically because the project
is an application that integrates with hardware or external systems and cannot be meaningfully unit-tested in isolation.

The tox.ini structure is identical to the full pipeline except:
- `envlist` omits `{pyXXX}-test` and `coverage`.
- The test and coverage environment definitions may be omitted entirely or left as unused definitions for future use.
- The `install` environment's `depends` list omits test and coverage dependencies.
- The `lint` environment runs `ruff check --fix ./src`, without `./tests`, for the `E902` reason the lint environment
  template gives above. The lint block comment drops its sentence about the test directory to match, and
  `pyproject.toml` carries no shared test corpus (see `/pyproject-style`).

A project that later gains a test directory moves to the full pipeline, which means restoring the test and coverage
environments, adding `./tests` to the lint command, and adding the shared test corpus to `pyproject.toml`. The three
changes travel together, since the corpus does nothing until `ruff check` reaches the directory it covers.

---

## Description field restraint

A `description` field states what the environment does when it runs. It is one sentence, two when the environment reads
its configuration from another file or produces one output the commands block does not reveal, and three when it
produces more than one such output. It opens with a bare third-person imperative verb, does not restate the environment
name, does not list the commands the environment already declares below it, and does not explain why the task matters.
See [the skill](../SKILL.md) for the full comment and description conventions.

```ini
# Avoid - restates the name, then narrates the commands
[testenv:lint]
description =
    The lint environment. This environment runs the linters. It first runs ruff to check formatting and style, then
    runs mypy to check types, and it is an important part of keeping the codebase consistent and maintainable for all
    contributors.

# Good
[testenv:lint]
description =
    Runs static code formatting, style, and typing checkers. Follows the configuration defined in the pyproject.toml
    file.
```

---

## Command reference

These are the development-automation commands. Agents normally drive them via `tox -e <env>`. The underlying
`automation-cli` commands are documented for diagnostics and for answering user questions.

### tox environments

| Environment                | Purpose                                                                  |
|----------------------------|--------------------------------------------------------------------------|
| `lint`                     | Purges `.pyi` stubs, then runs `ruff format`, `ruff check --fix`, `mypy` |
| `stubs`                    | Regenerates the `py.typed` marker and `.pyi` stub files                  |
| `{py312,py313,py314}-test` | Runs the test suite under each Python version, collecting coverage       |
| `coverage`                 | Combines per-version coverage and junit reports, applies the 100% gate   |
| `docs`                     | Builds API docs with Sphinx (and Doxygen for C++/hybrid projects)        |
| `build`                    | Builds the sdist and wheel distributions                                 |
| `upload`                   | Uploads the `dist/` files to PyPI via twine                              |
| `deploy`                   | Uploads the built html documentation to the project's Netlify site       |
| `install`                  | Builds and installs the project into its development mamba environment   |
| `uninstall`                | Uninstalls the project from its development mamba environment            |
| `create`                   | Creates the development mamba environment and installs dependencies      |
| `remove`                   | Removes (deletes) the development mamba environment                      |
| `provision`                | Recreates the development mamba environment and installs dependencies    |
| `export`                   | Exports the mamba environment to `envs/` as a `.yml` file                |
| `import`                   | Creates or updates the mamba environment from the stored `.yml` file     |

`tox -e lint` purges `.pyi` stubs (via `automation-cli purge-stubs`) so they do not interfere with mypy, and `tox -e
stubs` regenerates them afterward. See the lint environment template above for the paths `ruff check` and `mypy` each
receive.

### automation-cli commands

| Command                 | Key options                                                                                         | Purpose                                                             |
|-------------------------|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| `process-typed-markers` | none                                                                                                | Places the `py.typed` marker only at the library root               |
| `process-stubs`         | none                                                                                                | Distributes generated stubs from `stubs/` into the source tree      |
| `purge-stubs`           | none                                                                                                | Removes all `.pyi` stub files from the source tree                  |
| `acquire-pypi-token`    | `-rt`/`--replace-token`                                                                             | Ensures a valid PyPI token is stored in the shared `.pypirc`        |
| `upload-project`        | none                                                                                                | Uploads the `dist/` distributions to PyPI with twine                |
| `acquire-netlify-token` | `-rt`/`--replace-token`, `-rs`/`--replace-site`                                                     | Ensures the Netlify token and the project's site identifier are set |
| `deploy-docs`           | none                                                                                                | Uploads `docs/build/html` to the project's Netlify site             |
| `install-project`       | `-e`/`--environment-name`, `-ed`/`--environment-directory`, `--prerelease`                          | Builds and installs the project into the mamba environment          |
| `uninstall-project`     | `-e`/`--environment-name`, `-ed`/`--environment-directory`                                          | Uninstalls the project from the mamba environment                   |
| `create-environment`    | `-e`/`--environment-name`, `-p`/`--python-version`, `-ed`/`--environment-directory`, `--prerelease` | Creates the mamba environment and installs dependencies             |
| `remove-environment`    | `-e`/`--environment-name`, `-ed`/`--environment-directory`                                          | Removes (deletes) the mamba environment                             |
| `provision-environment` | `-e`/`--environment-name`, `-p`/`--python-version`, `-ed`/`--environment-directory`, `--prerelease` | Recreates the environment, then installs dependencies               |
| `import-environment`    | `-e`/`--environment-name`, `-ed`/`--environment-directory`                                          | Creates or updates the mamba environment from the stored `.yml`     |
| `export-environment`    | `-e`/`--environment-name`, `-ed`/`--environment-directory`                                          | Exports the mamba environment to `envs/` as a `.yml` file           |
