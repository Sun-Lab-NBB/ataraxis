# Environment templates

Complete tox.ini environment templates for each project archetype. All templates use the
modern convention (`dependency_groups = dev`) introduced in tox 4.22+. Legacy projects may still
use `extras = dev` with `[project.optional-dependencies]` — see the migration note in SKILL.md.

---

## Full Python pipeline

Canonical template for Python-only and Python + C++ extension projects. The ataraxis-automation
project itself is the root source of truth for this pipeline.

### `[tox]` section

```ini
# This file provides configurations for tox-based project development and management automation
# tasks.

# Base tox configurations. Note, the 'envlist' runs in the listed order whenever 'tox' is used
# without an -e specifier.
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

# This forces tox to create a 'sterile' environment into which the project with all dependencies
# is installed prior to running the requested tasks, isolating the process from the rest of the
# system.
isolated_build = True
```

### `[testenv]` base section (optional)

Use this section when any environment needs prerelease packages. When present, it applies to all
environments that do not override `setenv`.

```ini
# Allows installing prerelease packages.
[testenv]
setenv =
    UV_PRERELEASE = allow
```

tox does not merge `setenv` — an env that defines its own `setenv` block (test and coverage both
set `COVERAGE_FILE`) fully replaces the inherited base `setenv` rather than extending it. So if a
project needs prereleases during testing, re-declare `UV_PRERELEASE = allow` inside the test env
`setenv` alongside `COVERAGE_FILE` (see ataraxis-transport-layer-pc/tox.ini, which does this;
ataraxis-communication-interface and ataraxis-video-system keep prereleases out of test envs by
omitting it).

### lint environment

```ini
# Note: The 'basepython' argument should always be set to the earliest supported Python version.
[testenv: lint]
description =
    Runs static code formatting, style, and typing checkers. Follows the configuration defined
    in the pyproject.toml file.
dependency_groups = dev
basepython = py312
commands =
    automation-cli purge-stubs
    ruff format
    ruff check --fix ./src
    mypy ./src
```

**Parameterization:**
- `basepython`: Set to the earliest Python version in the supported range.
- `dependency_groups`: Always `dev`. Installs the project's dev dependency group (tox, uv,
  tox-uv, and type stubs).

**Self-hosting exception (ataraxis-automation only):** Since ataraxis-automation IS the automation
provider, its lint environment does not need `deps = ataraxis-automation==X.Y.Z`. It uses
`dependency_groups = dev` (or `extras = dev` in legacy form) to get its own dev dependencies.

#### mypy parallelism (large projects only)

`mypy ./src` runs single-threaded by default. This is correct for the overwhelming majority of
ataraxis libraries and application projects: their `src/` is small enough that a warm, incremental `mypy` run
finishes in well under a second, and the binary + SQLite incremental caches (on by default since
mypy 2.0) already make re-runs effectively free. You MUST NOT add parallelism to a project's lint
command by default.

mypy 2.x adds experimental parallel type checking via `-n N` / `--num-workers N` (config-file key
`num_workers`; environment override `MYPY_NUM_WORKERS`). It only helps large codebases — those
where a *cold* `mypy ./src` takes more than a few seconds (large application projects).
For those projects only, append the flag to the lint command:

```ini
commands =
    automation-cli purge-stubs
    ruff format
    ruff check --fix ./src
    mypy -n 8 ./src
```

Rules for enabling it:

- Confirm a real, measured speedup first: compare `time mypy ./src` with and without `-n` on a
  cold cache (`rm -rf .mypy_cache`). Adopt the flag only if it is a clear win on that project.
- Set `N` to the CI runner's physical core count (the upstream "up to 5x" figure assumes 8 workers
  on a large project).
- Parallel mode slows *warm/incremental* runs because of worker-startup overhead, so it benefits
  fresh CI checkouts, not local iterative development.
- Parallel mode implicitly enables `--native-parser` and may produce minor semantic differences
  from serial mode (it is still experimental). Keep small projects serial to avoid this risk.

### stubs environment

```ini
[testenv: stubs]
description =
    Generates the py.typed marker and the .pyi stub files using the project's wheel distribution.
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

### test environment

```ini
[testenv: {py312, py313, py314}-test]
package = wheel
description =
    Runs unit and integration tests for each of the python versions listed in the task name and
    aggregates test coverage data. Uses 'loadgroup' balancing and all logical cores to optimize
    task runtime speed.
dependency_groups = dev
setenv =
    COVERAGE_FILE = reports{/}.coverage.{envname}
    # Re-declare here only if the project needs prereleases during testing — this setenv block
    # fully replaces the base [testenv] setenv (tox does not merge).
    # UV_PRERELEASE = allow
commands =
    pytest --import-mode=append --cov={package_name} --cov-config=pyproject.toml \
    --cov-report=xml --junitxml=reports/pytest.xml.{envname} -n logical --dist loadgroup
```

**Parameterization:**
- Python version matrix `{py312, py313, py314}`: Must match the `requires-python` range in
  `pyproject.toml`. Core libraries (`ataraxis-*`) test 3 versions; applications may
  test fewer.
- `{package_name}` in `--cov`: The underscore-separated package name.
- `package = wheel`: Forces the project to be built as a wheel before testing.

### coverage environment

```ini
[testenv:coverage]
skip_install = true
description =
    Combines test-coverage data from multiple test runs (for different python versions) into a
    single html file. The file can be viewed by loading the 'reports/coverage_html/index.html'.
deps = ataraxis-automation=={version}
setenv = COVERAGE_FILE = reports/.coverage
depends = {py312, py313, py314}-test
commands =
    junitparser merge --glob reports/pytest.xml.* reports/pytest.xml
    coverage combine --keep
    coverage xml
    coverage html
```

**Parameterization:**
- `deps = ataraxis-automation=={version}`: Pin to the exact current release version.
- `depends`: Must list the same Python version matrix as the test environment.

**Self-hosting exception:** ataraxis-automation omits `skip_install` and `deps` since it provides
these tools itself.

### docs environment

**Python-only projects** (no Doxygen):

```ini
[testenv:docs]
description =
    Builds the API documentation from source code docstrings using Sphinx. The result can be
    viewed by loading 'docs/build/html/index.html'.
deps = ataraxis-automation=={version}
depends = uninstall
commands =
    sphinx-build -b html -d docs/build/doctrees docs/source docs/build/html -j auto -v
```

**Python + C++ extension projects** (with Doxygen):

```ini
[testenv:docs]
description =
    Builds the API documentation from source code docstrings using Doxygen, Breathe and Sphinx.
    The result can be viewed by loading 'docs/build/html/index.html'.
deps = ataraxis-automation=={version}
depends = uninstall
allowlist_externals = doxygen
commands =
    doxygen Doxyfile
    sphinx-build -b html -d docs/build/doctrees docs/source docs/build/html -j auto -v
```

**Parameterization:**
- Add `allowlist_externals = doxygen` and the `doxygen Doxyfile` command only for projects that
  have a `Doxyfile` at the project root.
- `depends = uninstall`: Ensures a clean environment state before building.

### build environment

**Standard Python projects:**

```ini
[testenv:build]
skip_install = true
description =
    Builds the project's source code distribution (sdist) and binary distribution (wheel).
deps = ataraxis-automation=={version}
allowlist_externals = docker
commands =
    python -m build . --sdist
    python -m build . --wheel
```

**C++ extension projects** (cibuildwheel):

```ini
[testenv:build]
skip_install = true
description =
    Builds the project's source code distribution (sdist) and binary distribution (wheel).
deps = ataraxis-automation=={version}
allowlist_externals = docker
commands =
    python -m build . --sdist
    cibuildwheel --output-dir dist --platform auto
```

### upload environment

```ini
# Note: use 'tox -e upload --replace-token' command to replace the token stored in the .pypirc
# file before uploading the project.
[testenv:upload]
skip_install = true
description =
    Uses twine to upload all files inside the project's 'dist' directory to PyPI.
deps = ataraxis-automation=={version}
allowlist_externals = distutils
commands =
    automation-cli acquire-pypi-token {posargs:}
    twine upload dist/* --skip-existing --config-file .pypirc
```

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
- `{posargs:}`: Allows passing additional flags at invocation time (e.g., `--prerelease` to
  enable prerelease package installation).

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
    Creates the project's development mamba environment using the requested python version and
    installs runtime and development project dependencies extracted from the pyproject.toml file.
commands =
    automation-cli create-environment --environment-name {env_abbr}_dev --python-version 3.14 {posargs:}
```

**Parameterization:**
- `--python-version`: Set to the latest Python version in the supported range.
- `{posargs:}`: Allows passing additional flags at invocation time (e.g., `--prerelease` to
  enable prerelease package installation).

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
    Provisions the project's development mamba environment by removing and (re)creating the
    environment.
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
    Exports the project's development mamba environment to the 'envs' project directory as a
    .yml file and as a spec.txt with revision history.
commands =
    automation-cli export-environment --environment-name {env_abbr}_dev
```

### import environment

```ini
[testenv:import]
skip_install = true
deps = ataraxis-automation=={version}
description =
    Creates or updates the project's development mamba environment using the .yml file stored in
    the 'envs' project directory.
commands =
    automation-cli import-environment --environment-name {env_abbr}_dev
```

---

## C++ docs-only pipeline

Template for pure C++ PlatformIO projects (libraries and firmware). These projects have only a
`docs` environment because they are not Python packages.

```ini
# This file provides configurations for tox-based project development and management automation
# tasks.

# Base tox configurations.
[tox]
requires =
    tox>=4,<5
    tox-uv>=1,<2
envlist = docs

# This forces tox to create a 'sterile' environment into which the project with all dependencies
# is installed prior to running the requested tasks, isolating the process from the rest of the
# system.
isolated_build = True

[testenv:docs]
skip_install = true
description =
    Builds the API documentation from source code docstrings using Doxygen, Breathe and Sphinx.
    The result can be viewed by loading 'docs/build/html/index.html'.
deps = ataraxis-automation=={version}
allowlist_externals = doxygen
commands =
    doxygen Doxyfile
    sphinx-build -b html -d docs/build/doctrees docs/source docs/build/html -j auto -v
```

**Key differences from the Python pipeline:**
- `envlist = docs` — only one environment.
- `skip_install = true` — no Python package to install.
- Always runs `doxygen Doxyfile` before `sphinx-build`.
- No lint, stubs, test, coverage, build, upload, or environment management environments.

---

## Reduced Python pipeline

Some Python application projects omit test and coverage environments from their envlist,
typically because the project is an application that integrates with hardware or external systems
and cannot be meaningfully unit-tested in isolation.

The tox.ini structure is identical to the full pipeline except:
- `envlist` omits `{pyXXX}-test` and `coverage`.
- The test and coverage environment definitions may be omitted entirely or left as unused
  definitions for future use.
- The `install` environment's `depends` list omits test and coverage dependencies.

---

## Command reference

These are the development-automation commands. Agents normally drive them via `tox -e <env>`; the
underlying `automation-cli` commands are documented for diagnostics and for answering user
questions.

### tox environments

| Environment                  | Purpose                                                                  |
|------------------------------|--------------------------------------------------------------------------|
| `lint`                       | Purges `.pyi` stubs, then runs `ruff format`, `ruff check --fix`, `mypy` |
| `stubs`                      | Regenerates the `py.typed` marker and `.pyi` stub files                  |
| `{py312,py313,py314}-test`   | Runs the test suite under each Python version, collecting coverage       |
| `coverage`                   | Combines per-version coverage and junit reports into xml + html          |
| `docs`                       | Builds API docs with Sphinx (and Doxygen for C++/hybrid projects)        |
| `build`                      | Builds the sdist and wheel distributions                                 |
| `upload`                     | Uploads the `dist/` files to PyPI via twine                              |
| `install`                    | Builds and installs the project into its development mamba environment   |
| `uninstall`                  | Uninstalls the project from its development mamba environment            |
| `create`                     | Creates the development mamba environment and installs dependencies      |
| `remove`                     | Removes (deletes) the development mamba environment                      |
| `provision`                  | Removes and (re)creates the development mamba environment                |
| `export`                     | Exports the mamba environment to `envs/` as `.yml` and `spec.txt`        |
| `import`                     | Creates or updates the mamba environment from the stored `.yml` file     |

`tox -e lint` purges `.pyi` stubs (via `automation-cli purge-stubs`) so they do not interfere with
mypy; `tox -e stubs` regenerates them afterward.

### automation-cli commands

| Command                  | Key options                                                            | Purpose                                                              |
|--------------------------|-----------------------------------------------------------------------|---------------------------------------------------------------------|
| `process-typed-markers`  | —                                                                     | Places the `py.typed` marker only at the library root               |
| `process-stubs`          | —                                                                     | Distributes generated stubs from `stubs/` into the source tree      |
| `purge-stubs`            | —                                                                     | Removes all `.pyi` stub files from the source tree                   |
| `acquire-pypi-token`     | `-rt`/`--replace-token`                                               | Ensures a valid PyPI token is stored in `.pypirc`                    |
| `install-project`        | `-e`/`--environment-name`, `-ed`/`--environment-directory`, `--prerelease` | Builds and installs the project into the mamba environment     |
| `uninstall-project`      | `-e`/`--environment-name`, `-ed`/`--environment-directory`           | Uninstalls the project from the mamba environment                   |
| `create-environment`     | `-e`/`--environment-name`, `-p`/`--python-version`, `-ed`/`--environment-directory`, `--prerelease` | Creates the mamba environment and installs dependencies |
| `remove-environment`     | `-e`/`--environment-name`, `-ed`/`--environment-directory`           | Removes (deletes) the mamba environment                             |
| `provision-environment`  | `-e`/`--environment-name`, `-p`/`--python-version`, `-ed`/`--environment-directory`, `--prerelease` | Removes and recreates the mamba environment           |
| `import-environment`     | `-e`/`--environment-name`, `-ed`/`--environment-directory`           | Creates or updates the mamba environment from the stored `.yml`     |
| `export-environment`     | `-e`/`--environment-name`, `-ed`/`--environment-directory`           | Exports the mamba environment to `envs/` as `.yml` and `spec.txt`   |
