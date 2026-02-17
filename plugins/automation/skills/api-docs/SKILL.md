---
name: api-docs
description: >-
  Applies Sun Lab API documentation conventions when creating or modifying Sphinx documentation
  files. Covers conf.py configuration, RST file structure, Doxygen/Breathe integration for C++
  projects, tox build environments, and Makefile wrappers. Use when creating documentation for a
  new project, modifying existing docs/ files, adding new modules to API documentation, or when
  the user asks about documentation conventions.
user-invocable: true
---

# API documentation

Applies Sun Lab conventions for generating API documentation HTML pages from source code
docstrings using Sphinx, autodoc, Napoleon, and optionally Breathe/Doxygen for C++ code.

---

## Scope

**Covers:**
- Sphinx configuration (conf.py) for Python-only, C++-only, and hybrid projects
- RST file structure (index.rst, welcome.rst, api.rst)
- Doxygen configuration and Breathe integration for C++ code
- tox.ini `[testenv:docs]` build environment setup
- Makefile and make.bat wrappers
- Documentation hosting URL conventions
- Documentation dependency management via ataraxis-automation

**Does not cover:**
- MCP server modules or shared asset libraries — these are consumed by AI agents, not
  end-users, and MUST NOT be included in API documentation
- Writing Python docstrings or type annotations (see `/python-style`)
- README file conventions (see `/readme-style`)
- pyproject.toml dependency sections (see `/pyproject-style`)
- General project structure or architecture (see `/explore-codebase`)

---

## Documentation architecture

### Project archetypes

Sun Lab projects follow one of three documentation archetypes based on language composition:

| Archetype   | Language     | Extensions                                                | Doxygen |
|-------------|--------------|-----------------------------------------------------------|---------|
| Python-only | Pure Python  | autodoc, napoleon, click, typehints, furo theme            | No      |
| C++-only    | Pure C++     | breathe, furo theme                                        | Yes     |
| Hybrid      | Python + C++ | All Python extensions + breathe                            | Yes     |

### Directory structure

For the full project-level directory tree, invoke `/project-layout`. The documentation-specific
layout within each project is:

```text
project-root/
├── docs/
│   ├── Makefile              # Unix Sphinx wrapper (delegates to tox)
│   ├── make.bat              # Windows Sphinx wrapper
│   └── source/
│       ├── conf.py           # Sphinx configuration
│       ├── index.rst         # Main page with toctree
│       ├── welcome.rst       # Landing page content
│       └── api.rst           # API reference directives
├── Doxyfile                  # (C++ and hybrid projects only)
└── tox.ini                   # Contains [testenv:docs] build environment
```

C++ and hybrid projects additionally generate:

```text
docs/source/doxygen/
└── xml/                      # Doxygen-generated XML consumed by Breathe
```

### Extension stack

**Python-only projects** use these extensions in this exact order:

```python
extensions = [
    'sphinx.ext.autodoc',
    'sphinx.ext.napoleon',
    'sphinx_click',
    'sphinx_autodoc_typehints',
]
```

**C++-only projects** use:

```python
extensions = [
    'breathe',
]
```

**Hybrid projects** use all Python extensions plus `breathe`, inserted after
`sphinx_autodoc_typehints`:

```python
extensions = [
    'sphinx.ext.autodoc',
    'sphinx.ext.napoleon',
    'sphinx_click',
    'sphinx_autodoc_typehints',
    'breathe',
]
```

**Ordering constraint:** `sphinx_click` MUST appear before `sphinx_autodoc_typehints`.
`sphinx_autodoc_typehints` imports the `sphinx.ext.autodoc.mock` submodule at load time, which
shadows the `mock` function that `sphinx_click` needs. Loading `sphinx_click` first ensures it
captures the correct function binding before the submodule shadowing occurs.

### Documentation dependencies

All documentation dependencies are bundled as runtime dependencies of `ataraxis-automation`.
Downstream projects include `ataraxis-automation` in their `dev` optional dependencies, which
transitively provides all documentation tools. You MUST NOT add Sphinx or documentation
dependencies directly to downstream project pyproject.toml files.

---

## Workflow

### Creating documentation from scratch

1. **Determine archetype**: Identify whether the project is Python-only, C++-only, or hybrid
   based on the source code language composition.

2. **Create directory structure**: Create `docs/source/` directory and the Makefile wrappers.
   Read [rst-templates.md](references/rst-templates.md) for Makefile and make.bat templates.

3. **Generate conf.py**: Read [conf-py-templates.md](references/conf-py-templates.md) and use
   the appropriate archetype template. Replace all placeholder values with project-specific
   information.

4. **Generate RST files**: Read [rst-templates.md](references/rst-templates.md) and create
   `index.rst`, `welcome.rst`, and `api.rst` using the templates. Populate `api.rst` with
   the correct directives for the project's modules.

5. **Configure Doxygen** (C++ and hybrid only): Read
   [doxygen-reference.md](references/doxygen-reference.md) and create a `Doxyfile` at the
   project root listing all C++ source files.

6. **Configure tox build**: Add the `[testenv:docs]` section to `tox.ini` following the
   appropriate pattern for the archetype.

7. **Verify**: Run through the verification checklist below.

### Modifying existing documentation

1. **Read existing files**: Read `docs/source/conf.py`, `docs/source/api.rst`, and any other
   files you intend to modify before making changes.

2. **Identify archetype**: Check the extensions list in `conf.py` to determine the project
   archetype.

3. **Apply changes**: Follow the conventions in this skill and the reference files. Common
   modifications include:
   - Adding new Python modules to `api.rst` via `automodule` directives
   - Adding new Click CLI groups to `api.rst` via `click` directives
   - Adding new C++ source files to `api.rst` via `doxygenfile` directives (and to `Doxyfile`)
   - Updating project metadata in `conf.py` and `welcome.rst`

4. **Verify**: Run through the verification checklist below.

---

## Key conventions

### conf.py rules

- Version extraction MUST use `importlib_metadata.version()` for Python and hybrid projects.
  C++-only projects hardcode the version string.
- Copyright format: `'YEAR, Sun (NeuroAI) lab'` where YEAR is the current year.
- The `templates_path` and `exclude_patterns` fields are included but left at defaults
  (`['_templates']` and `[]` respectively).
- Napoleon is configured for Google-style docstrings only (`napoleon_numpy_docstring = False`).
- All Napoleon and `sphinx_autodoc_typehints` settings MUST match the templates exactly. See
  [conf-py-templates.md](references/conf-py-templates.md) for the full settings.
- The HTML theme is always `furo`. Furo provides built-in light/dark mode toggling with no
  additional extensions required.

### api.rst rules

- Each documented module or file gets its own RST section with an `=`-underlined heading.
- Python modules use `automodule` with `:members:`, `:undoc-members:`, and
  `:show-inheritance:`.
- Click CLI groups use `click` with `:prog:` set to the CLI entry point name and
  `:nested: full`.
- C++ files use `doxygenfile` with `:project:` set to the project name.
- Section headings MUST be descriptive names, not module paths (e.g., "Precision Timer" not
  "ataraxis_time.precision_timer.timer_class").
- You MUST NOT add `automodule` directives for MCP server modules or shared asset modules.
  These components are designed for consumption by AI agents, not end-users, and do not belong
  in API documentation.

### welcome.rst rules

- Title format: `Welcome to PROJECT_NAME API documentation page` with matching `=` underline.
- First paragraph: the bare project description — the same sentence used in all other canonical
  description locations for the project archetype (e.g., `pyproject.toml`, `__init__.py`,
  `README.md`, or `library.json`). No language or project name prefix.
- Second paragraph: standard Ataraxis project attribution with Sun (NeuroAI) lab link.
- Third paragraph: standard disclaimer that the site contains API docs only, with link to
  GitHub repository.
- Footer: explicit RST link targets for the GitHub repo and Sun lab URLs.

### tox.ini docs environment

For the complete tox.ini conventions and all other environment definitions, invoke `/tox-config`.
`/tox-config` is the authoritative source; if any pattern here conflicts, `/tox-config` takes
precedence. The docs-specific patterns are summarized here for convenience.

**Python-only projects** (no external Doxygen dependency):

```ini
[testenv:docs]
description =
    Builds the API documentation from source code docstrings using Sphinx. The result can be
    viewed by loading 'docs/build/html/index.html'.
depends = uninstall
commands =
    sphinx-build -b html -d docs/build/doctrees docs/source docs/build/html -j auto -v
```

**C++ and hybrid projects** add Doxygen before Sphinx:

```ini
[testenv:docs]
description =
    Builds the API documentation from source code docstrings using Doxygen, Breathe and Sphinx.
    The result can be viewed by loading 'docs/build/html/index.html'.
deps = ataraxis-automation==VERSION
depends = uninstall
allowlist_externals = doxygen
commands =
    doxygen Doxyfile
    sphinx-build -b html -d docs/build/doctrees docs/source docs/build/html -j auto -v
```

**Rules:**
- The `depends = uninstall` ensures a clean environment state before building.
- Downstream projects pin `deps = ataraxis-automation==VERSION` to the current release. The
  `ataraxis-automation` project itself omits this field since it IS ataraxis-automation.
- The Sphinx build command MUST use `-j auto` for parallel building and `-v` for verbose output.
- C++ and hybrid projects MUST add `allowlist_externals = doxygen` and run `doxygen Doxyfile`
  before the Sphinx build command.
- C++-only projects use `skip_install = true` since there is no Python package to install. See
  [doxygen-reference.md](references/doxygen-reference.md) for the full tox pattern.

### Hosting convention

Documentation is hosted on Netlify with standardized URLs:

```text
https://PROJECT_NAME-api-docs.netlify.app/
```

This URL is declared in `pyproject.toml` under `[project.urls]` as the `Documentation` key.

### Makefile and make.bat

These are standard Sphinx wrapper files. They are rarely used directly since builds are invoked
via tox (`tox -e docs`). Use the exact templates from
[rst-templates.md](references/rst-templates.md) when creating new projects.

---

## Related skills

| Skill               | Relationship                                                           |
|---------------------|------------------------------------------------------------------------|
| `/python-style`     | Defines docstring and type annotation conventions consumed by autodoc  |
| `/cpp-style`        | Defines Doxygen comment conventions consumed by Breathe                |
| `/readme-style`     | Defines README conventions; README links to hosted API docs            |
| `/pyproject-style`  | Defines pyproject.toml conventions including documentation URL         |
| `/project-layout`   | Provides full project directory trees; this skill owns docs/ internals |
| `/tox-config`       | Owns full tox.ini conventions; this skill summarizes docs env patterns |
| `/commit`           | Should be invoked after completing documentation changes               |
| `/explore-codebase` | Provides project context needed to identify modules for api.rst        |

---

## Proactive behavior

You should proactively offer to invoke this skill when:
- Creating a new Sun Lab project that needs API documentation
- Adding new Python modules or Click CLI commands that should appear in API docs
- Adding C++ source files to a hybrid or C++ project
- The user asks about Sphinx, autodoc, Breathe, Doxygen, or documentation generation

---

## Verification checklist

You MUST verify your work against this checklist before submitting any documentation changes.

```text
API Documentation Compliance:
- [ ] Documentation archetype correctly identified (Python-only, C++-only, or hybrid)
- [ ] Directory structure matches convention (docs/source/ with conf.py, index.rst, welcome.rst, api.rst)
- [ ] conf.py uses correct extension stack for the archetype
- [ ] conf.py uses correct extension ordering
- [ ] Version extracted via importlib_metadata (Python/hybrid) or hardcoded (C++-only)
- [ ] Copyright uses current year and 'Sun (NeuroAI) lab' format
- [ ] Napoleon configured for Google-style only (numpy disabled)
- [ ] All sphinx_autodoc_typehints settings present and correct (Python/hybrid)
- [ ] Breathe configuration present and correct (C++/hybrid)
- [ ] html_theme set to 'furo'
- [ ] index.rst includes welcome.rst and has toctree with api
- [ ] welcome.rst follows template with correct project name and description
- [ ] welcome.rst includes Sun lab and GitHub repository links
- [ ] api.rst uses automodule with :members:, :undoc-members:, :show-inheritance: (Python)
- [ ] api.rst uses click directive with :prog: and :nested: full (Click CLIs)
- [ ] api.rst uses doxygenfile with :project: (C++ files)
- [ ] api.rst section headings are descriptive, not module paths
- [ ] No MCP server or shared asset modules included in api.rst
- [ ] Doxyfile present at project root (C++/hybrid only)
- [ ] Doxyfile outputs to docs/source/doxygen with XML generation enabled
- [ ] tox.ini [testenv:docs] follows correct pattern for archetype
- [ ] tox.ini docs command uses -j auto -v flags
- [ ] Makefile and make.bat present in docs/
- [ ] No Sphinx dependencies added directly to downstream pyproject.toml
- [ ] Documentation URL follows https://PROJECT-api-docs.netlify.app/ convention
```
