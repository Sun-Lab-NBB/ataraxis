# RST and build file templates

Complete templates for all documentation source files and build wrappers. Replace all
`<PLACEHOLDER>` values with project-specific information before use.

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

The landing page content. Replace `<PROJECT_NAME>` and `<PROJECT_DESCRIPTION>` with
project-specific values.

```rst
Welcome to <PROJECT_NAME> API documentation page
=================================================

<PROJECT_DESCRIPTION>

This library is part of the broader 'Ataraxis' science-automation project, developed in the
`Sun (NeuroAI) lab <https://neuroai.github.io/sunlab/>`_ at Cornell University.

This website only contains the API documentation for the classes and methods offered by this library. See the project
GitHub repository for installation instructions and library usage examples:
`<PROJECT_NAME> GitHub repository <https://github.com/Sun-Lab-NBB/<PROJECT_NAME>>`_.

.. _`<PROJECT_NAME> GitHub repository`: https://github.com/Sun-Lab-NBB/<PROJECT_NAME>
.. _`Sun (NeuroAI) lab`: https://neuroai.github.io/sunlab/
```

**Rules:**
- The title underline (`=`) MUST match the exact character length of the title text.
- `<PROJECT_DESCRIPTION>` is the bare project description — the same sentence used in all other
  canonical description locations for the project archetype (e.g., `pyproject.toml`, `__init__.py`,
  `README.md`, or `library.json`). No language prefix ("A Python library that...") and no project
  name prefix ("project-name is...").
- The GitHub URL uses the `Sun-Lab-NBB` organization.

### Placeholders

| Placeholder             | Description                              | Example                             |
|-------------------------|------------------------------------------|-------------------------------------|
| `<PROJECT_NAME>`        | Project name matching pyproject.toml     | `ataraxis-time`                     |
| `<PROJECT_DESCRIPTION>` | Bare project description, imperative     | `Provides high-precision timers...` |

---

## api.rst

The API reference page. Structure varies by archetype and project contents. The comment header
and directive patterns are shown below.

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
