---
name: api-docs
description: >-
  Applies API documentation conventions when creating or modifying Sphinx documentation files. Covers conf.py
  configuration, RST file structure, Doxygen/Breathe integration for C++ projects, tox build environments, and Makefile
  wrappers. Use when creating documentation for a new project, modifying existing docs/ files, adding new modules to API
  documentation, or when the user asks about documentation conventions.
user-invocable: false
---

# API documentation

Applies conventions for generating API documentation HTML pages from source code docstrings using Sphinx, autodoc,
Napoleon, and optionally Breathe/Doxygen for C++ code.

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
- MCP server modules or shared asset libraries. These serve AI agents rather than end-users, so they MUST NOT be
  included in API documentation
- Writing Python docstrings or type annotations (see `/python-style`)
- README file conventions (see `/readme-style`)
- pyproject.toml dependency sections (see `/pyproject-style`)
- General project structure or architecture (see `/explore-codebase`)

---

## Documentation architecture

### Documentation archetypes

Projects follow one of three documentation archetypes based on language composition:

| Archetype   | Language     | Extensions                                      | Doxygen |
|-------------|--------------|-------------------------------------------------|---------|
| Python-only | Pure Python  | autodoc, napoleon, click, typehints, furo theme | No      |
| C++-only    | Pure C++     | breathe, furo theme                             | Yes     |
| Hybrid      | Python + C++ | All Python extensions + breathe                 | Yes     |

### Directory structure

For the full project-level directory tree, invoke `/project-layout`. The documentation-specific layout within each
project is:

```text
project-root/
├── docs/
│   ├── Makefile              # Minimal Sphinx wrapper (builds usually run via tox)
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

**Hybrid projects** use all Python extensions plus `breathe`, inserted after `sphinx_autodoc_typehints`:

```python
extensions = [
    'sphinx.ext.autodoc',
    'sphinx.ext.napoleon',
    'sphinx_click',
    'sphinx_autodoc_typehints',
    'breathe',
]
```

**Ordering constraint:** `sphinx_click` MUST appear before `sphinx_autodoc_typehints`. `sphinx_autodoc_typehints`
imports the `sphinx.ext.autodoc.mock` submodule at load time, which shadows the `mock` function that `sphinx_click`
needs. Loading `sphinx_click` first ensures it captures the correct function binding before the submodule shadowing
occurs.

### Documentation dependencies

All documentation dependencies are bundled as runtime dependencies of `ataraxis-automation`. Downstream projects include
`ataraxis-automation` in their `dev` optional dependencies, which transitively provides all documentation tools. You
MUST NOT add Sphinx or documentation dependencies directly to downstream project pyproject.toml files.

---

## Workflow

### Creating documentation from scratch

1. **Determine archetype**: Identify whether the project is Python-only, C++-only, or hybrid based on the source code
   language composition.

2. **Create directory structure**: Create `docs/source/` directory and the Makefile wrappers. Read
   [rst-templates.md](references/rst-templates.md) for Makefile and make.bat templates.

3. **Generate conf.py**: Read [conf-py-templates.md](references/conf-py-templates.md) and use the appropriate archetype
   template. Replace all placeholder values with project-specific information.

4. **Generate RST files**: Read [rst-templates.md](references/rst-templates.md) and create `index.rst`, `welcome.rst`,
   and `api.rst` using the templates. Populate `api.rst` with the correct directives for the project's modules.

5. **Configure Doxygen** (C++ and hybrid only): Read [doxygen-reference.md](references/doxygen-reference.md) and create
   a `Doxyfile` at the project root listing all C++ source files.

6. **Configure tox build**: Add the `[testenv:docs]` section to `tox.ini` following the appropriate pattern for the
   archetype.

7. **Verify**: Run through the verification checklist below.

### Modifying existing documentation

1. **Read existing files**: Read `docs/source/conf.py`, `docs/source/api.rst`, and any other files you intend to modify
   before making changes.

2. **Identify archetype**: Check the extensions list in `conf.py` to determine the documentation archetype.

3. **Apply changes**: Follow the conventions in this skill and the reference files. Common modifications include:
   - Adding new Python modules to `api.rst` via `automodule` directives
   - Adding new Click CLI groups to `api.rst` via `click` directives
   - Adding new C++ source files to `api.rst` via `doxygenfile` directives (and to `Doxyfile`)
   - Updating project metadata in `conf.py` and `welcome.rst`

4. **Verify**: Run through the verification checklist below.

---

## Key conventions

### conf.py rules

- Version extraction MUST use the `importlib_metadata` backport for Python and hybrid projects. C++-only projects
  hardcode the version string.
- The import MUST be written as `import importlib_metadata` and the call MUST stay module-qualified, as
  `importlib_metadata.version()`. Sphinx reads every module-level name in `conf.py` that matches a configuration key,
  and `version` is such a key, so the `from importlib_metadata import version` form would make Sphinx adopt the function
  object as the project version.
- `importlib_metadata` reaches `conf.py` through `ataraxis-automation`, which bundles it alongside every other
  documentation dependency. A downstream project declares it only when its own runtime code imports it. A `conf.py` that
  reads the backport while nothing in the project's dependency tree requires it still builds, until an unrelated bump
  drops the transitive copy. Treat the undeclared runtime import as the defect.
- Copyright format: `'YEAR, Sun (NeuroAI) lab'` where YEAR is the current year.
- `author` is the only author key Sphinx defines and it holds a string, so a multi-author project joins the names into
  that one string. A plural `authors` key and a list value both parse without error and reach no template, leaving the
  rendered pages on Sphinx's `Author name not set` default.
- Do NOT add IDE-specific suppression comments anywhere in `conf.py`, which ruff excludes via `extend-exclude` and mypy
  excludes as part of `docs/`. See `/python-style` for the framework-wide policy on IDE directives.
- The `templates_path` and `exclude_patterns` fields are included but left at defaults (`['_templates']` and `[]`
  respectively) for Python-only and hybrid archetypes. The C++-only archetype omits both fields entirely (it uses a
  minimal breathe-only conf.py).
- Napoleon is configured for Google-style docstrings only (`napoleon_numpy_docstring = False`).
- All Napoleon and `sphinx_autodoc_typehints` settings MUST match the templates exactly. The extension is enabled by
  listing it in `extensions`, so the templates carry no `sphinx_autodoc_typehints` boolean. Sphinx accepts unknown names
  in `conf.py` without complaint, so such a line reads as configuration while doing nothing. See
  [conf-py-templates.md](references/conf-py-templates.md) for the full settings.
- The HTML theme is always `furo`.

### api.rst rules

- Each documented module or file gets its own RST section with an `=`-underlined heading.
- Python modules use `automodule` with `:members:`, `:undoc-members:`, and `:show-inheritance:`.
- Click CLI groups use `click` with `:prog:` set to the CLI entry point name and `:nested: full`.
- C++ files use `doxygenfile` with `:project:` set to the project name.
- Section headings MUST be descriptive names, not module paths (e.g., "Precision Timer" not
  "ataraxis_time.precision_timer.timer_class").
- You MUST NOT add `automodule` directives for MCP server modules or shared asset modules.

### welcome.rst rules

Read [rst-templates.md](references/rst-templates.md) for the welcome.rst title, paragraph, and link-target rules.

### Wrap width

Break a hand-written RST line only where it would otherwise pass 120 characters, and fill each line to that limit before
breaking. Prose wrapped at a narrower width reads as a rigid block and re-wraps badly at any other viewport. The test is
mechanical: a wrapped line that ends before column 100 while its next word would still fit under 120 is re-flowed. A
line ending early because the sentence or the paragraph ends, or because it holds a title underline, a directive, an
option field, or a link target, is already correct.

### Prose punctuation and positive description

Documentation prose in `welcome.rst`, `index.rst`, and other RST source follows the framework prose conventions. Prose
uses only the full stop and the comma to separate clauses. Do not use a semicolon or an em-dash (`--`, `—`, or `–`) as a
separator, and use a colon only where it is lexically appropriate. A single hyphen stays available as a list marker, in
tables, and in compound words. State what the subject does and what is currently true. Do not frame it by what it is not
or what it used to be, and keep a "not Y" contrast only when it is load-bearing because it corrects a counter-intuitive
assumption, giving its reason. Sentences over 40 words are difficult for humans to parse and must be broken into smaller
sentences at natural clause boundaries. Every hand-written sentence in `welcome.rst` and `index.rst` must also be free
of typos and grammatical errors, while a defect in a generated page is fixed in the source docstring under the owning
language style skill.

### Accessibility

The Sphinx build renders a public HTML site, so hand-written RST follows the same accessibility rules the framework
applies to every rendered page.

- An `image` or `figure` directive carries an `:alt:` option naming what the image shows, not the file it loads.
- Link text names the page or document it opens. Text such as "click here" or a bare URL fails that test.

### Content restraint

Sphinx generates the entire API reference from the docstrings in the source, so hand-written RST is the smallest surface
in the project and must stay that way. Every sentence you write by hand is a sentence no tool keeps in sync with the
code.

**No hand-written API prose**: Do not describe a class, a function, a parameter, or a return value in RST. The
`automodule`, `click`, and `doxygenfile` directives already emit that content from the authoritative source. When an API
description reads poorly on the rendered page, correct the docstring in the source rather than adding prose around the
directive.

**The cover test**: Before keeping a hand-written sentence, cover it and try to reconstruct it from the project
description and the generated page it introduces. A sentence you are able to reconstruct carries no information, so
delete it.

**Fixed-size front matter**: `welcome.rst` carries the three paragraphs the
[rst-templates.md](references/rst-templates.md) rules prescribe and stops. Do not add a features list, a quickstart, an
architecture overview, or a motivation section, because each duplicates the README and drifts from it. `index.rst`
carries the toctree.

**No change narration**: RST source describes the current API. Version history and migration notes belong to the release
notes.

**No ratchet**: Editing an RST page is not a reason to lengthen it, so a change that leaves the page's claims intact
leaves the page exactly as it stands. When the API changes, rewrite the affected sentences and delete the ones the
change made redundant, so the page ends no longer than it started unless the new API is genuinely harder to introduce.

### tox.ini docs environment

`/tox-config` owns the `[testenv:docs]` definition and every field it carries. Invoke that skill and read the "docs
environment" section of its `environment-templates.md` reference for the Python-only and Doxygen-enabled templates, and
the "C++ docs-only pipeline" section of the same reference for the complete tox.ini of a pure C++ project. Do not copy
those templates into documentation notes, because a local copy drifts from the authoritative one.

This skill owns only the archetype mapping onto those templates:
- Python-only projects build with Sphinx alone and carry no `doxygen` command.
- C++-only and hybrid projects run `doxygen Doxyfile` before the Sphinx build, so their `docs` environment MUST also
  declare `allowlist_externals = doxygen`. Without that field tox refuses to execute the external Doxygen binary and the
  build fails before Sphinx ever reads the XML. See [doxygen-reference.md](references/doxygen-reference.md) for the
  Doxyfile that command consumes.

### Hosting convention

Documentation is hosted on Netlify with standardized URLs:

```text
https://PROJECT_NAME-api-docs.netlify.app/
```

This URL is declared in `pyproject.toml` under `[project.urls]` as the `Documentation` key.

### Makefile and make.bat

These are standard Sphinx wrapper files. They are rarely used directly since builds are invoked via tox (`tox -e docs`).
Use the exact templates from [rst-templates.md](references/rst-templates.md) when creating new projects.

---

## Related skills

| Skill               | Relationship                                                                 |
|---------------------|------------------------------------------------------------------------------|
| `/python-style`     | Defines docstring and type annotation conventions consumed by autodoc        |
| `/cpp-style`        | Defines Doxygen comment conventions consumed by Breathe                      |
| `/readme-style`     | Defines README conventions, and the README links to hosted API docs          |
| `/pyproject-style`  | Defines pyproject.toml conventions including documentation URL               |
| `/project-layout`   | Provides full project directory trees, while this skill owns docs/ internals |
| `/tox-config`       | Owns the tox.ini docs environment this skill maps archetypes onto            |
| `/commit`           | Should be invoked after completing documentation changes                     |
| `/explore-codebase` | Provides project context needed to identify modules for api.rst              |

---

## Proactive behavior

You should proactively offer to invoke this skill when:
- Creating a new project that needs API documentation
- Adding new Python modules or Click CLI commands that should appear in API docs
- Adding C++ source files to a hybrid or C++ project
- The user asks about Sphinx, autodoc, Breathe, Doxygen, or documentation generation

---

## Verification checklist

You MUST verify your work against this checklist before submitting any documentation changes.

```text
API Documentation Compliance:

Settled by `tox -e docs`. A build surfaces each of these as an error or a warning, so run the
command rather than hand-checking them. They stay listed for reviews performed without a build.
- [ ] conf.py uses correct extension ordering
- [ ] Breathe configuration present and correct (C++/hybrid)
- [ ] index.rst includes welcome.rst and has toctree with api
- [ ] Doxyfile present at project root (C++/hybrid only)
- [ ] Doxyfile outputs to docs/source/doxygen with XML generation enabled
- [ ] C++ and hybrid tox.ini docs env runs doxygen Doxyfile first and declares allowlist_externals = doxygen

Settled by reading. No command inspects these, so this checklist is their only enforcement. Walk
every one against the files you wrote.
- [ ] Documentation archetype correctly identified (Python-only, C++-only, or hybrid)
- [ ] Directory structure matches convention (docs/source/ with conf.py, index.rst, welcome.rst, api.rst)
- [ ] conf.py uses correct extension stack for the archetype
- [ ] Version extracted via module-qualified importlib_metadata (Python/hybrid) or hardcoded (C++-only)
- [ ] importlib_metadata reaches conf.py through ataraxis-automation, and is declared by the project only when
      its own runtime code imports it
- [ ] Copyright uses current year and 'Sun (NeuroAI) lab' format
- [ ] Author declared as the singular `author` key holding one string (no plural `authors` key, no list value)
- [ ] Napoleon configured for Google-style only (numpy disabled)
- [ ] All sphinx_autodoc_typehints settings present and correct (Python/hybrid)
- [ ] templates_path = ['_templates'] and exclude_patterns = [] present (Python/hybrid), both fields absent (C++-only)
- [ ] html_theme set to 'furo'
- [ ] welcome.rst follows template with correct project name and description
- [ ] welcome.rst includes the attribution and GitHub repository links
- [ ] api.rst uses automodule with :members:, :undoc-members:, :show-inheritance: (Python)
- [ ] api.rst uses click directive with :prog: and :nested: full (Click CLIs)
- [ ] api.rst uses doxygenfile with :project: (C++ files)
- [ ] api.rst section headings are descriptive, not module paths
- [ ] No MCP server or shared asset modules included in api.rst
- [ ] tox.ini [testenv:docs] taken from /tox-config, not copied or paraphrased into this skill's notes
- [ ] Makefile and make.bat present in docs/
- [ ] No Sphinx dependencies added directly to downstream pyproject.toml
- [ ] Documentation URL follows https://PROJECT-api-docs.netlify.app/ convention
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons, hyphen bullets, and code
      syntax exempt)
- [ ] Prose states what the component does, not what it is not or used to be (contrast only when load-bearing)
- [ ] Sentences in hand-written RST prose stay under 40 words
- [ ] Hand-written RST lines under 120 characters, with wrapped prose filled to that limit rather than broken at a
      narrower width
- [ ] Hand-written RST prose free of typos and grammar errors
- [ ] Hand-written RST image directives carry an :alt: option naming what the image shows, and link text names the
      page it opens
- [ ] No hand-written RST describing a class, function, parameter, or return value
- [ ] Poor API prose corrected in the source docstring rather than worked around in RST
- [ ] welcome.rst carries only its three prescribed paragraphs (no features, quickstart, or overview sections)
- [ ] Each retained hand-written sentence survives the cover test (unable to be reconstructed from the project
      description and the generated page)
- [ ] No version history or migration notes in RST source
- [ ] Edits leave hand-written RST no longer than it started unless the new API is genuinely harder to introduce
```
