# Environment templates

Complete tox.ini environment templates for each Sun Lab project archetype. All templates use the
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
setenv = COVERAGE_FILE = reports{/}.coverage.{envname}
commands =
    pytest --import-mode=append --cov={package_name} --cov-config=pyproject.toml \
    --cov-report=xml --junitxml=reports/pytest.xml.{envname} -n logical --dist loadgroup
```

**Parameterization:**
- Python version matrix `{py312, py313, py314}`: Must match the `requires-python` range in
  `pyproject.toml`. Core libraries (`ataraxis-*`) test 3 versions; applications (`sl-*`) may
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

Some Python application projects (`sl-*`) omit test and coverage environments from their envlist,
typically because the project is an application that integrates with hardware or external systems
and cannot be meaningfully unit-tested in isolation.

The tox.ini structure is identical to the full pipeline except:
- `envlist` omits `{pyXXX}-test` and `coverage`.
- The test and coverage environment definitions may be omitted entirely or left as unused
  definitions for future use.
- The `install` environment's `depends` list omits test and coverage dependencies.
