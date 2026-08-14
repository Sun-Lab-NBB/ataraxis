# RST and build file templates

Complete templates for all documentation source files and build wrappers. Replace all `<PLACEHOLDER>` values with
project-specific information before use. Every template's prose is filled to the 120 character limit, under the
wrap-width rule the skill file defines.

---

## index.rst

The main documentation page. This file is identical across all projects regardless of archetype.

```rst
.. Main documentation page file, determines the overall layout of the static documentation .html page (after it is
   rendered with Sphinx) and allows linking additional sub-pages. Use it to build the skeleton of the documentation
   website.

.. Includes the Welcome page. This is the page that will be displayed whenever the user navigates to the documentation
   website.
.. include:: welcome.rst

.. Adds the left-hand-side navigation panel to the documentation website. Uses the API file to generate content list.
.. toctree::
   :maxdepth: 2
   :caption: Contents:
   :hidden:

   api

Index
==================
* :ref:`genindex`
```

This file requires no customization.

---

## welcome.rst

The landing page content. Replace `<PROJECT_NAME>` and `<PROJECT_DESCRIPTION>` with project-specific values.

```rst
Welcome to <PROJECT_NAME> API documentation page
================================================

<PROJECT_DESCRIPTION>

This library is part of the `Ataraxis <https://github.com/Sun-Lab-NBB/ataraxis>`_ framework for AI-assisted scientific
hardware control, developed in the `Sun (NeuroAI) lab <https://neuroai.github.io/sunlab/>`_ at Cornell University.

This website contains the API documentation for the classes and methods offered by this library, together with the
reference for every command exposed by its command-line interface where the project declares one. See the project
GitHub repository for installation instructions and library usage examples:
`<PROJECT_NAME> GitHub repository <https://github.com/Sun-Lab-NBB/<PROJECT_NAME>>`_.

.. _`Ataraxis`: https://github.com/Sun-Lab-NBB/ataraxis
.. _`<PROJECT_NAME> GitHub repository`: https://github.com/Sun-Lab-NBB/<PROJECT_NAME>
.. _`Sun (NeuroAI) lab`: https://neuroai.github.io/sunlab/
```

**Rules:**
- The title reads `Welcome to <PROJECT_NAME> API documentation page`, and the title underline (`=`) MUST match the exact
  character length of the title text.
- The first paragraph is `<PROJECT_DESCRIPTION>`, the bare project description, which is the same sentence used in all
  other canonical description locations for the project archetype (e.g., `pyproject.toml`, `__init__.py`, `README.md`,
  or `library.json`). No language prefix ("A Python library that...") and no project name prefix ("project-name is...").
- For Ataraxis projects, the second paragraph MUST include the standard Ataraxis attribution sentence with links to the
  Ataraxis repository and the Sun (NeuroAI) lab. For a non-Ataraxis project, the paragraph carries the interoperation
  context below alone, and the page omits the paragraph where the project has no such component.
- The second paragraph also names, and links to, the component the project is built to interoperate with, such as the
  companion library implementing the other end of its protocol or the hardware it drives. The test is whether the named
  component sits on the other side of an interface this project implements, and a component that passes that test
  belongs on the page. A predecessor also belongs, meaning a library this project reimplements or supersedes, because
  it tells a reader arriving from that library what changed. Context that fails both tests, such as an adjacent
  project, a funding source, or an unrelated comparison with another library, is omitted.
- The third paragraph is the standard disclaimer that the site carries API documentation only, with a link to the
  project GitHub repository.
- The footer declares an explicit RST link target for every inline link the page uses.
- The GitHub URL uses the `Sun-Lab-NBB` organization.

### Placeholders

| Placeholder             | Description                          | Example                             |
|-------------------------|--------------------------------------|-------------------------------------|
| `<PROJECT_NAME>`        | Project name matching pyproject.toml | `ataraxis-time`                     |
| `<PROJECT_DESCRIPTION>` | Bare project description, imperative | `Provides high-precision timers...` |

---

## api.rst

The API reference page. Structure varies by archetype and project contents. The comment header and directive patterns
are shown below.

### Comment header

Every api.rst starts with a comment describing its purpose:

**Python-only:**

```rst
.. This file provides the instructions for how to display the API documentation generated using sphinx autodoc
   extension. Use it to declare Python documentation sub-directories via appropriate modules (automodule, etc.).
```

**C++-only:**

```rst
.. This file provides the instructions for how to display the API documentation generated using doxygen-breathe-sphinx
   pipeline. Use it to declare C++ documentation sub-directories via appropriate doxygen directives.
```

**Hybrid:**

```rst
.. This file provides the instructions for how to display the API documentation generated using sphinx autodoc
   extension. Use it to declare Python and C++ extension documentation sub-directories via appropriate modules
   (automodule, doxygenfile and sphinx-click).
```

### Python module directive

```rst
Section Name
============

.. automodule:: package_name.module_name
   :members:
   :undoc-members:
   :show-inheritance:
```

### Module constant directive

Use this where a package re-exports a module-level constant. The `automodule` directive above discovers module-level
data through the source of the module it documents, so it skips a constant the package re-exports, and the constant
never reaches the rendered page. Name the DEFINING module rather than the re-exporting package, because autodoc reads
the attribute docstring from that module's source and otherwise falls back to the docstring of the value's own type.
Precede the block with a comment stating both reasons:

```rst
.. Documents the package constants explicitly, since the automodule directive above discovers module-level data through
   the source of the module it documents and therefore skips a constant this package re-exports. The directive names
   the defining module rather than the package, because autodoc reads the attribute docstring from that module's
   source and falls back to the docstring of the value's own type when it is pointed at the re-exporting package.
.. autodata:: package_name.submodule.dataclasses.CONSTANT_NAME
```

### Click CLI directive

```rst
Section Name
============

.. click:: package_name.cli_module:cli_group
   :prog: cli-entry-point
   :nested: full
```

### C++ file directive

```rst
Section Name
============

.. doxygenfile:: source_file.cpp
   :project: project-name
```

Or for header files:

```rst
Section Name
============

.. doxygenfile:: header_file.h
   :project: project-name
```

### Section ordering

Order sections by logical grouping:

1. Primary Python modules (core functionality)
2. Click CLI commands
3. Helper/utility modules
4. C++ extension files (hybrid projects)

---

## Makefile

The Unix/Linux Sphinx build wrapper. This file is identical across all projects.

```makefile
# Minimal makefile for Sphinx documentation. Generally not used as sphinx is interfaced with using 'tox'.

# You can set these variables from the command line, and also
# from the environment for the first two.
SPHINXOPTS    ?=
SPHINXBUILD   ?= sphinx-build
SOURCEDIR     = source
BUILDDIR      = build

# Put it first so that "make" without argument is like "make help".
help:
	@$(SPHINXBUILD) -M help "$(SOURCEDIR)" "$(BUILDDIR)" $(SPHINXOPTS) $(O)

.PHONY: help Makefile

# Catch-all target: route all unknown targets to Sphinx using the new
# "make mode" option.  $(O) is meant as a shortcut for $(SPHINXOPTS).
%: Makefile
	@$(SPHINXBUILD) -M $@ "$(SOURCEDIR)" "$(BUILDDIR)" $(SPHINXOPTS) $(O)
```

**Important:** Makefile indentation MUST use tabs, not spaces.

---

## make.bat

The Windows Sphinx build wrapper. This file is identical across all projects.

```bat
@ECHO OFF

pushd %~dp0

REM Command file for Sphinx documentation

if "%SPHINXBUILD%" == "" (
	set SPHINXBUILD=sphinx-build
)
set SOURCEDIR=source
set BUILDDIR=build

%SPHINXBUILD% >NUL 2>NUL
if errorlevel 9009 (
	echo.
	echo.The 'sphinx-build' command was not found. Make sure you have Sphinx
	echo.installed, then set the SPHINXBUILD environment variable to point
	echo.to the full path of the 'sphinx-build' executable. Alternatively you
	echo.may add the Sphinx directory to PATH.
	echo.
	echo.If you don't have Sphinx installed, grab it from
	echo.https://www.sphinx-doc.org/
	exit /b 1
)

if "%1" == "" goto help

%SPHINXBUILD% -M %1 %SOURCEDIR% %BUILDDIR% %SPHINXOPTS% %O%
goto end

:help
%SPHINXBUILD% -M help %SOURCEDIR% %BUILDDIR% %SPHINXOPTS% %O%

:end
popd
```
