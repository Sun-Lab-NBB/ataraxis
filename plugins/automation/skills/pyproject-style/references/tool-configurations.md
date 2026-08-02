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
indent-width = 4          # Same as black, indents are 4 spaces
target-version = "py312"  # Targets the lowest supported version of python
src = ["src"]             # The name of the root source code directory

# Excludes 'service' .py files, such as the sphinx configuration file, from linting.
extend-exclude = ["conf.py"]

# Checks for all potential violations and uses the exclusions below to target-disable specific ones.
lint.select = ["ALL"]
```

Set `target-version` to the **lowest** supported Python version for the project (e.g., `"py312"`
for `>=3.12,<3.15`, `"py314"` for `>=3.14,<3.15`).

### Ruff format

```toml
# Additional formatting configurations
[tool.ruff.format]
quote-style = "double"             # Uses double quotes for strings
indent-style = "space"             # Uses space for indents
skip-magic-trailing-comma = false  # Like black, ignores trailing commas
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

### Ruff per-file ignores

```toml
# Additional, file-specific 'ignore' directives
[tool.ruff.lint.per-file-ignores]
"**/__init__.py" = [
    "F401", # Imported but unused
    "F403", # Wildcard imports
]
```

When a `tests/` directory exists, add a second per-file-ignores key for test files relaxing test-specific lint
rules. Spell the glob `**/tests/**/*.py` or `tests/**/*.py` to match the project's layout. The commonly relaxed rules
are `S101`, `SLF001`, `PLR2004`, `ANN`/`ANN001`/`ANN201`, `D`/`D202`, `INP001`, `PT006`, and `ARG001`; the exact set
varies per project. Each ignore carries an explanatory inline comment, as elsewhere.

### Ruff isort

```toml
[tool.ruff.lint.isort]
case-sensitive = true              # Takes case into account when sorting imports
combine-as-imports = true          # Combines multiple "as" imports for the same package
force-wrap-aliases = true          # Wraps "as" imports so that each uses a separate line
force-sort-within-sections = true  # Forces "as" and "from" imports for the same package to be close
length-sort = true                 # Places shorter imports first
```

### Universal Ruff ignores

These ignores are present in all projects:

```toml
lint.ignore = [
    "W291",    # Conflicts with docstring formatting
    "C901",    # Sometimes complex functions are necessary
    "PLR0913", # Sometimes complex functions are necessary
    "PLR0912", # Sometimes complex functions are necessary
    "PLR0915", # Sometimes complex type definitions are necessary
    "PLR0911", # Sometimes complex return graphs are necessary
    "COM812",  # Conflicts with the formatter
    "ISC001",  # Conflicts with the formatter
    "PT001",   # https://github.com/astral-sh/ruff/issues/8796#issuecomment-182590771
    "PT023",   # https://github.com/astral-sh/ruff/issues/8796#issuecomment-1825907715
    "D107",    # __init__ is documented inside the main class docstring where applicable
    "D205",    # Bugs out for file descriptions
    "PLW0603", # While global statement usage is not ideal, it streamlines certain development patterns
]
```

### Project-specific Ruff ignores

Add these ignores only when the project requires them:

| Ignore   | Reason                                | Projects using it                          |
|----------|---------------------------------------|--------------------------------------------|
| `ANN401` | `Any` type is needed                  | ataraxis-automation                        |
| `BLE001` | Blind exception catching is necessary | ataraxis-automation, ataraxis-video-system |
| `S602`   | Subprocess calls are needed           | ataraxis-automation, ataraxis-video-system |
| `S607`   | Partial executable paths are needed   | ataraxis-automation                        |
| `FBT001` | Positional boolean arguments (UI)     | ataraxis-communication-interface           |
| `FBT002` | Boolean default values (UI)           | ataraxis-communication-interface           |
| `FBT003` | Boolean positional in function calls  | application projects                       |
| `SLF001` | Private member access is necessary    | ataraxis-communication-interface           |
| `SIM115` | Non-context-manager file operations   | application projects                       |
| `TID252` | Relative imports required             | ataraxis-video-system                      |
| `T201`   | Print statements are needed           | application projects                       |
| `W293`   | Whitespace in UI formatting           | application projects                       |

### Ruff unused arguments

Some projects include this optional section:

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
# Strict mode settings (equivalent to --strict)
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
    "project-name-\\d+",  # Ignores temporary folder created by setuptools when building the sdist
    "venv.*/",             # Ignores virtual environments
    "build/",              # Ignores the sdist directory
    "dist/",               # Ignores the wheel directory
    "docs/",               # Ignores the sphinx / doxygen directory
    "stubs/",              # Ignores stubs directory (stubgen output target)
    "recipe/",             # Ignores recipe directory (grayskull output target)
    "tests/",              # Ignores the test directory.
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
    "project-name-\\d+",  # Ignores temporary folder created by setuptools when building the sdist
    "venv.*/",             # Ignores virtual environments
    "build/",              # Ignores the sdist directory
    "dist/",               # Ignores the wheel directory
    "docs/",               # Ignores the sphinx / doxygen directory
    "stubs/",              # Ignores stubs directory (stubgen output target)
    "recipe/",             # Ignores recipe directory (grayskull output target)
    "tests/",              # Ignores the test directory.
]
```

### MyPy exclusion list

The exclusion list is the same for both tiers. All entries are mandatory:

| Entry               | Purpose                                      |
|---------------------|----------------------------------------------|
| `project-name-\\d+` | Temporary folder created during sdist builds |
| `venv.*/`           | Virtual environments                         |
| `build/`            | Build output directory                       |
| `dist/`             | Distribution output directory                |
| `docs/`             | Sphinx / Doxygen documentation directory     |
| `stubs/`            | stubgen output target directory              |
| `recipe/`           | grayskull output target directory            |
| `tests/`            | Test directory (excluded from type checking) |

---

## Coverage configuration

The test suite MUST cover 100% of the measured statements. The `[tool.coverage.report]` section
declares that gate, and the `[tool.coverage.run]` section lists the files that stay outside the
measured corpus. These settings are identical across all projects, apart from the project-specific
`omit` entries:

```toml
# Lists the source files excluded from coverage measurement in full. Interface modules, such as the CLI, are covered
# by the tests written for the functions they wrap.
[tool.coverage.run]
omit = [
    "*/package_name/cli.py",
]

# This is used by the 'test' tox tasks to aggregate coverage data produced during pytest runtimes.
[tool.coverage.paths]

# Maps coverage measured in site-packages to source files in src
source = ["src/", ".tox/*/lib/python*/site-packages/"]

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

`fail_under` applies to every command that renders a report, so `pytest --cov`, `coverage report`,
`coverage xml`, and `coverage html` all fail once the measured total drops below 100. The `coverage`
tox environment passes `--fail-under=0` to its artifact-writing commands so that the html and xml
reports are always written, and applies the gate through a trailing `coverage report` call. See
`/tox-config` for that command sequence.

### Choosing between omit and pragma

Two mechanisms remove code from the coverage requirement. Choose between them by the size of the
target.

| Target                                                   | Mechanism                  | Declared in            |
|-----------------------------------------------------------|----------------------------|------------------------|
| A whole module that the test suite never exercises       | `[tool.coverage.run] omit` | `pyproject.toml`       |
| A statement or branch inside an otherwise-covered module | `# pragma: no cover`       | The source line itself |

Whole-module exclusion belongs in `omit`. The canonical case is a user-facing interface module,
where every member wraps a function the suite already covers, so listing the file once is clearer
and cheaper to maintain than annotating each of its members. The modules that qualify are:

- `cli.py`, and any other module that defines Click commands.
- The MCP server module and its `*_tools.py` tool modules.
- `__main__.py`, and similar process entry points.

Write each `omit` pattern with a leading wildcard so that it matches both the `src/` tree and the
`.tox/*/lib/python*/site-packages/` copy that the test environments measure, for example
`"*/package_name/cli.py"`. A module listed in `omit` carries no `# pragma: no cover` comments,
because the whole file already sits outside the measured corpus.

Every other module stays in the measured corpus, and the individual statements the suite cannot
reach carry a targeted `# pragma: no cover`. See `/python-style` for the statements that qualify.

### Additional coverage settings

Projects with multiprocessing or parallel test execution add these keys to the same
`[tool.coverage.run]` section that carries the `omit` list:

```toml
[tool.coverage.run]
parallel = true
concurrency = ["multiprocessing", "thread"]
```

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

Platform-specific architecture settings use `[tool.cibuildwheel.linux]`,
`[tool.cibuildwheel.windows]`, and `[tool.cibuildwheel.macos]` sub-tables.
