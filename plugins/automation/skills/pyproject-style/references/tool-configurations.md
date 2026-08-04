# Tool configurations reference

Detailed specifications for all `[tool.*]` sections in pyproject.toml files.

---

## Hatch build targets

### sdist exclusions

Exclude CI and packaging directories from source distributions:

```toml
# Specifies files that should not be included in the source-code distribution.
[tool.hatch.build.targets.sdist]
exclude = [".github", "recipe"]
```

### Wheel packages

Point to the source package directory. The package name uses underscores:

```toml
# Specifies the library structure.
[tool.hatch.build.targets.wheel]
packages = ["src/package_name"]
```

Additional directories may be included when they are part of the distributed package:

```toml
packages = ["src/package_name", "notebooks"]
packages = ["src/ataraxis_video_system", "examples"]
```

---

## Ruff configuration

### Universal settings

These settings are identical across all projects:

```toml
# Ruff Configuration.
[tool.ruff]
line-length = 120         # Deviates from the commonly used '80' standard.
indent-width = 4
target-version = "py312"  # Targets the lowest supported version of python
src = ["src"]             # The name of the root source code directory

# Excludes 'service' .py files, such as the sphinx configuration file, from linting.
extend-exclude = ["conf.py"]

# Checks for all potential violations and uses the exclusions below to target-disable specific ones.
lint.select = ["ALL"]
```

Set `target-version` to the **lowest** supported Python version for the project (e.g., `"py312"` for `>=3.12,<3.15`,
`"py314"` for `>=3.14,<3.15`).

### Ruff format

```toml
# Additional formatting configurations
[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false  # Like black, honors magic trailing commas
line-ending = "auto"               # Automatically detects and standardizes line-ending character
```

### Ruff lint configuration

```toml
# Docstrings and comments' line length
[tool.ruff.lint.pycodestyle]
max-doc-length = 120  # Maximum documentation line length, the same as code

# Docstrings style
[tool.ruff.lint.pydocstyle]
convention = "google"
```

### The two ignore corpora

Ruff configuration carries two standing ignore lists, and they are governed by opposite rules. Name them in full
wherever either one is referenced, because the bare word "corpus" cannot be resolved to one of them by a reader who
arrives at a single sentence.

| Name                               | Location                                        | Governs                   |
|------------------------------------|-------------------------------------------------|---------------------------|
| the universal `lint.ignore` corpus | `lint.ignore` in `[tool.ruff]`                  | Library code under `src/` |
| the shared test corpus             | the `"tests/**/*.py"` key of `per-file-ignores` | Test code under `tests/`  |

Each opens with a shared block, the entries every project carries, and closes with a project-specific section below a
blank line. The `per-file-ignores."**/__init__.py"` key is a third ignore list rather than a corpus. It waives two
import rules and takes no project-specific section.

Test code is linted, and it is held to the wider of the two lists. Tests assert, reach into private members, inline
expected values, and omit annotations and docstrings, all of which library code is forbidden to do. The shared test
corpus waives exactly the rules that report those patterns, so a rule may be required in one corpus and forbidden in the
other. `SLF001` is the clearest case: it stays out of the shared block of the universal `lint.ignore` corpus and is a
member of the shared test corpus.

### Ruff per-file ignores

```toml
# Additional, file-specific 'ignore' directives
[tool.ruff.lint.per-file-ignores]
"**/__init__.py" = [
    "F401", # Imported but unused
    "F403", # Wildcard imports
]
```

When a `tests/` directory exists, add a second per-file-ignores key for test files. Spell the glob as `tests/**/*.py`,
which matches both direct children of `tests/` and files nested in its subdirectories. The shared test corpus below is
present in every project, followed by any project-specific entries:

```toml
"tests/**/*.py" = [
    # Shared test corpus. These entries are present in every Ataraxis framework and Sollertia platform project.
    "S101",    # Tests assert by design
    "PLR2004", # Expected values are clearer inline than as named constants
    "INP001",  # The test directory is not an importable package
    "SLF001",  # Tests exercise private module members directly
    "PT006",   # Parametrize argument names are declared as a single comma-separated string
    "ARG001",  # Pytest injects fixtures that individual tests may not reference
    "T201",    # Diagnostic prints are how a test reports intermediate state
    "D",       # Test names and inline comments carry the intent that docstrings carry in library code
    "ANN",     # Test functions and mock helpers do not need annotations

    # Project-specific
    "S106",    # Token arguments in tests carry fake values
]
```

`tests/**/*.py` is the spelling, anchored at the project root. The `**/tests/**/*.py` variant reaches the same test
files and additionally waives the whole corpus for any `tests` directory nested anywhere under `src/`, so library code
placed there is linted as test code. Rewrite the prefixed form to the anchored one wherever it appears.

`D` and `ANN` are family prefixes rather than individual codes. Test files trip a wide and shifting set of members of
both families, so the prefixes keep the list stable as tests are added.

These entries take effect only when the lint task passes the test directory to `ruff check`. A lint task that checks
`./src` alone leaves the whole key inert. See `/tox-config` for the lint task definition.

Projects extend this key with their own entries below the shared block. When judging a test file, read the project's own
`"tests/**/*.py"` key as well, because every code it lists is waived for that project's test files.

### Ruff isort

```toml
[tool.ruff.lint.isort]
case-sensitive = true
combine-as-imports = true          # Combines multiple "as" imports for the same package
force-wrap-aliases = true          # Wraps "as" imports so that each uses a separate line
force-sort-within-sections = true  # Forces "as" and "from" imports for the same package to be close
length-sort = true                 # Places shorter imports first
```

### The universal `lint.ignore` corpus

Every project carries the same shared block of the universal `lint.ignore` corpus, followed by a project-specific
section. A category comment heads each section and a blank line separates them, matching the layout used for dependency
arrays:

```toml
lint.ignore = [
    # Universal lint.ignore corpus. Present in every Ataraxis framework and Sollertia platform project.
    "W291",    # Conflicts with docstring formatting
    "ANN401",  # While suboptimal, ANY is a helpful type when used with caution
    "BLE001",  # Broad exception handling is necessary at library boundaries
    "C901",    # Sometimes complex functions are necessary
    "PLR0911", # Sometimes complex return graphs are necessary
    "PLR0912", # Sometimes complex functions are necessary
    "PLR0913", # Sometimes complex functions are necessary
    "PLR0915", # Sometimes functions with many statements are necessary
    "PLR0917", # PLR0913 already covers argument counts, and call sites pass keyword arguments
    "COM812",  # Conflicts with the formatter
    "ISC001",  # Conflicts with the formatter
    "PT001",   # https://github.com/astral-sh/ruff/issues/8796#issuecomment-1825907715
    "PT023",   # https://github.com/astral-sh/ruff/issues/8796#issuecomment-1825907715
    "D107",    # __init__ is documented inside the main class docstring where applicable
    "D205",    # Bugs out for file descriptions
    "PLW0603", # While global statement usage is not ideal, it streamlines certain development patterns
    "TID252",  # Conflicts with the library import management strategy, which requires parent-relative imports
    "CPY001",  # The license is declared in the LICENSE file and the license and license-files metadata fields

    # Project-specific
    "S602",    # Subprocess calls with shell=True are necessary to drive mamba and uv
]
```

The universal `lint.ignore` corpus stays complete in every project, including where a given rule never fires. Because
`lint.select = ["ALL"]` adopts each new ruff release automatically, a complete list keeps a newly stabilized rule from
breaking the lint task over a question the framework has already settled. The same completeness applies to the shared
test corpus in every project that carries a test directory.

### Project-specific Ruff ignores

Add these ignores only when the project requires them. Place them in the project-specific section of `lint.ignore`,
below the shared block:

| Ignore   | Reason                                                      |
|----------|-------------------------------------------------------------|
| `S602`   | Subprocess calls with `shell=True` are needed               |
| `S607`   | Partial executable paths are needed                         |
| `SLF001` | Private member access is needed outside tests               |
| `FBT001` | Positional boolean arguments are needed                     |
| `FBT002` | Boolean default values are needed                           |
| `FBT003` | Boolean positional values in calls are needed               |
| `SIM115` | File operations outside a context manager are needed        |
| `T201`   | Print statements are needed                                 |
| `W293`   | Whitespace in UI formatting is needed                       |
| `TRY301` | Raising inside a helper is needed to build a full traceback |

Keep the security rules `S602` and `S607`, together with `SLF001`, out of the shared block of the universal
`lint.ignore` corpus. Each reports a genuine risk in a project that has no reason to trip it, and each is added to the
project-specific section only by a project whose library code needs it.

`SLF001` carries opposite status in the two corpora, so read the requirement for the one being judged. It stays out of
the shared block of the universal `lint.ignore` corpus, where it enforces the rule that private members stay inside the
module that defines them. It is a required member of the shared test corpus, where tests are the sole sanctioned
exception to that rule. A project that lists `SLF001` in the shared test corpus and omits it from the shared block of
the universal `lint.ignore` corpus is compliant with both rules at once.

### Ruff unused arguments

Some projects include this optional section. It is the last of the `[tool.ruff.lint.*]` sub-tables, placed below
`[tool.ruff.lint.isort]`:

```toml
[tool.ruff.lint.flake8-unused-arguments]
ignore-variadic-names = true       # Ignores unused *args and **kwargs
```

---

## MyPy configuration

### Full strict mode (core libraries)

Used by all `ataraxis-*` projects:

```toml
# MyPy configuration section.
[tool.mypy]
# Strict mode settings (--strict), plus the unused-config warning that --strict does not enable
warn_unused_configs = true
disallow_any_generics = true
disallow_subclassing_any = true
disallow_untyped_calls = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
disallow_untyped_decorators = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_return_any = true
no_implicit_reexport = true
strict_equality = true

# Extra checks (equivalent to --extra-checks)
extra_checks = true

# Pretty output (equivalent to --pretty)
pretty = true

exclude = [
    "project-name-\\d+",  # The temporary folder created when building the sdist
    "venv.*/",
    "build/",
    "dist/",
    "docs/",
    "stubs/",             # stubgen output target
    "recipe/",            # grayskull output target
    "tests/",
]
```

Replace `project-name` in the first exclude entry with the actual project name (hyphenated).

### Minimal mode (applications)

Used by application projects:

```toml
# MyPy configuration section.
[tool.mypy]
disallow_untyped_defs = true # Enforces function annotation
warn_unused_ignores = true   # Warns against using 'type: ignore' for packages with type stubs

exclude = [
    "project-name-\\d+",  # The temporary folder created when building the sdist
    "venv.*/",
    "build/",
    "dist/",
    "docs/",
    "stubs/",             # stubgen output target
    "recipe/",            # grayskull output target
    "tests/",
]
```

### MyPy exclusion list

The exclusion list is the same for both tiers, and every one of its entries is mandatory.

---

## Pytest configuration

Projects with a `tests/` directory declare the import mode so that a bare `pytest` invocation resolves test modules the
same way the `test` tox task does:

```toml
# Pytest configuration. The import mode is declared here so that a bare 'pytest' invocation resolves test modules the
# same way the '{py}-test' tox task does.
[tool.pytest.ini_options]
addopts = "--import-mode=importlib"
```

`importlib` mode imports each test module under its own name without inserting its directory into `sys.path`, which is
what lets identically named modules coexist in sibling test directories under the `tests/**/*.py` layout.

A project whose tests spawn subprocesses that unpickle callables defined in test modules adds `pythonpath` so those
workers resolve the `tests` package:

```toml
pythonpath = ["."]
```

The src layout keeps this entry from shadowing the installed distribution the suite measures coverage against. Add it
only for a project that spawns such workers, and state the reason in the comment above the section.

---

## Coverage configuration

The test suite MUST cover 100% of the measured statements. The `[tool.coverage.report]` section declares that gate, and
the `[tool.coverage.run]` section lists the files that stay outside the measured corpus. These settings are identical
across all projects, apart from the project-specific `omit` entries:

```toml
# Lists the source files excluded from coverage measurement in full. Interface modules, such as the CLI, are covered
# by the tests written for the functions they wrap.
[tool.coverage.run]
omit = [
    "*/package_name/cli.py",
]

# This is used by the 'test' tox tasks to aggregate coverage data produced during pytest runtimes.
[tool.coverage.paths]

# Maps coverage measured in site-packages to source files in src. The second pattern matches the POSIX virtual
# environment layout and the third matches the Windows layout, which omits the 'python*' path component. Both are
# required for the 'coverage' task to merge the data measured by each 'test' task into a single record per source file.
# Without the merge, every statement is judged against a single python version, so any statement that only executes on
# some of the tested versions is misreported as uncovered.
source = [
    "src/",
    ".tox/*/lib/python*/site-packages/",
    ".tox/*/Lib/site-packages/",
]

# Same as above, specifies the output directory for the coverage .html report
[tool.coverage.html]
directory = "reports/coverage_html"

# Specifies the coverage gate and additional ignore directives
[tool.coverage.report]
fail_under = 100     # Requires the test suite to cover every measured statement
show_missing = true  # Lists the statements missed by the test suite, so a failed check names the lines to cover
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "def __del__",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "pass",
]
```

`fail_under` applies to every command that renders a report, so `pytest --cov`, `coverage report`, `coverage xml`, and
`coverage html` all fail once the measured total drops below 100. See `/tox-config` for the `coverage` environment
command sequence that renders the artifacts and applies the gate.

The `source` list in `[tool.coverage.paths]` carries one entry per virtual environment layout the project is tested on.
POSIX hosts place installed packages under `lib/python*/site-packages/` and Windows hosts place them under
`Lib/site-packages/`, so both patterns stay in the list for every project regardless of the host it is currently
developed on. See `/tox-config` for the `coverage` environment that consumes this mapping.

### Choosing between omit and pragma

Two mechanisms remove code from the coverage requirement. Choose between them by the size of the target.

| Target                                                   | Mechanism                  | Declared in            |
|----------------------------------------------------------|----------------------------|------------------------|
| A whole module that the test suite never exercises       | `[tool.coverage.run] omit` | `pyproject.toml`       |
| A statement or branch inside an otherwise-covered module | `# pragma: no cover`       | The source line itself |

Whole-module exclusion belongs in `omit`. The canonical case is a user-facing interface module, where every member wraps
a function the suite already covers, so listing the file once is clearer and cheaper to maintain than annotating each of
its members. The modules that qualify are:

- `cli.py`, and any other module that defines Click commands.
- The MCP server module and its `*_tools.py` tool modules.
- `__main__.py`, and similar process entry points.

Write each `omit` pattern with a leading wildcard so that it matches both the `src/` tree and the `site-packages` copy
that the test environments measure on any host, for example `"*/package_name/cli.py"`. A module listed in `omit` carries
no `# pragma: no cover` comments, because the whole file already sits outside the measured corpus.

Every other module stays in the measured corpus, and the individual statements the suite cannot reach carry a targeted
`# pragma: no cover`. See `/python-style` for the statements that qualify.

### Additional coverage settings

Projects with multiprocessing or parallel test execution add these keys to the same `[tool.coverage.run]` section that
carries the `omit` list:

```toml
[tool.coverage.run]
parallel = true
concurrency = ["multiprocessing", "thread"]
```

A project may also measure branch coverage, which reports a conditional or loop whose alternate path no test takes:

```toml
[tool.coverage.run]
branch = true
```

`branch = true` raises what `fail_under = 100` demands, since a partial branch counts as a gap once the key is present.
Add it to a project whose suite already passes with it, rather than to a project that would need new tests written to
restore the gate.

---

## scikit-build-core configuration (C-extension projects)

For projects using scikit-build-core (e.g., ataraxis-time):

```toml
[tool.scikit-build]
sdist.exclude = [".github", "recipe"]
minimum-version = "0.9"
build-dir = "build/{wheel_tag}"
```

### cibuildwheel configuration

C-extension projects also include cibuildwheel settings for multi-platform binary wheel builds:

```toml
[tool.cibuildwheel]
build-verbosity = 1
skip = ["*t-*"]               # Skip free-threaded wheels
build-frontend = "build[uv]"
test-command = "pytest {project}/tests -n logical --dist loadgroup"
test-requires = ["pytest", "pytest-xdist"]
```

Platform-specific architecture settings use `[tool.cibuildwheel.linux]`, `[tool.cibuildwheel.windows]`, and
`[tool.cibuildwheel.macos]` sub-tables.
