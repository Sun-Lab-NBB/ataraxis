# conf.py templates

Complete Sphinx configuration templates for each documentation archetype. Replace all
`<PLACEHOLDER>` values with project-specific information before use.

---

## Python-only archetype

Use this template for projects containing only Python source code.

```python
# Configuration file for the Sphinx documentation builder.
import importlib_metadata

# -- Project information -----------------------------------------------------
project = '<PROJECT_NAME>'
# noinspection PyShadowingBuiltins
copyright = '<YEAR>, Sun (NeuroAI) lab'
author = '<AUTHOR>'
# Extracts the project version from the metadata .toml file.
release = importlib_metadata.version("<PROJECT_NAME>")

# -- General configuration ---------------------------------------------------
extensions = [
    'sphinx.ext.autodoc',        # To build documentation from python source code docstrings.
    'sphinx.ext.napoleon',       # To read google-style docstrings (works with autodoc module).
    'sphinx_click',              # Must load before sphinx_autodoc_typehints to avoid mock import shadowing.
    'sphinx_autodoc_typehints',  # To parse typehints into documentation
]

templates_path = ['_templates']
exclude_patterns = []

# Google-style docstring parsing configuration for napoleon extension
napoleon_google_docstring = True
napoleon_numpy_docstring = False
napoleon_include_init_with_doc = False
napoleon_include_private_with_doc = False
napoleon_include_special_with_doc = False
napoleon_use_admonition_for_examples = False
napoleon_use_admonition_for_notes = False
napoleon_use_admonition_for_references = False
napoleon_use_ivar = False
napoleon_use_param = True
napoleon_use_rtype = True

# Additional sphinx-typehints configuration
sphinx_autodoc_typehints = True
always_document_param_types = False
typehints_document_rtype = True
typehints_use_rtype = True
typehints_defaults = 'comma'
simplify_optional_unions = True
typehints_formatter = None
typehints_use_signature = False
typehints_use_signature_return = False

# -- Options for HTML output -------------------------------------------------
html_theme = 'furo'
```

### Placeholders

| Placeholder      | Description                          | Example               |
|------------------|--------------------------------------|-----------------------|
| `<PROJECT_NAME>` | Project name matching pyproject.toml | `ataraxis-automation` |
| `<YEAR>`         | Current copyright year               | `2026`                |
| `<AUTHOR>`       | Author name(s)                       | `Ivan Kondratyev`     |

---

## C++-only archetype

Use this template for projects containing only C++ source code. Note that version is hardcoded
(no importlib_metadata) and `skip_install = true` is set in the tox docs environment.

```python
# Configuration file for the Sphinx documentation builder.

# -- Project information -----------------------------------------------------
project = '<PROJECT_NAME>'
# noinspection PyShadowingBuiltins
copyright = '<YEAR>, Sun (NeuroAI) lab'
author = '<AUTHOR>'
release = '<VERSION>'

# -- General configuration ---------------------------------------------------
extensions = [
    'breathe',             # To read doxygen-generated xml files (to parse C++ documentation).
]

# Breathe configuration
breathe_projects = {"<PROJECT_NAME>": "./doxygen/xml"}
breathe_default_project = "<PROJECT_NAME>"

# -- Options for HTML output -------------------------------------------------
html_theme = 'furo'
```

### Optional: preprocessor macros

If the C++ project uses macros that Doxygen needs to expand, add Breathe preprocessor
configuration after `breathe_default_project`:

```python
breathe_doxygen_config_options = {
    'ENABLE_PREPROCESSING': 'YES',
    'MACRO_EXPANSION': 'YES',
    'EXPAND_ONLY_PREDEF': 'NO',
    'PREDEFINED': 'MACRO_NAME='
}
```

Replace `MACRO_NAME=` with the actual macro definitions needed. Multiple macros can be
space-separated.

### Placeholders

| Placeholder      | Description                          | Example                      |
|------------------|--------------------------------------|------------------------------|
| `<PROJECT_NAME>` | Project name matching pyproject.toml | `ataraxis-micro-controller`  |
| `<YEAR>`         | Current copyright year               | `2026`                       |
| `<AUTHOR>`       | Author name(s)                       | `Ivan Kondratyev`            |
| `<VERSION>`      | Hardcoded version string             | `2.0.0`                      |

---

## Hybrid archetype

Use this template for projects containing both Python and C++ source code. Combines the full
Python configuration with Breathe integration.

```python
# Configuration file for the Sphinx documentation builder.
import importlib_metadata

# -- Project information -----------------------------------------------------
project = '<PROJECT_NAME>'
# noinspection PyShadowingBuiltins
copyright = '<YEAR>, Sun (NeuroAI) lab'
author = '<AUTHOR>'
# Extracts the project version from the .toml file.
release = importlib_metadata.version("<PROJECT_NAME>")

# -- General configuration ---------------------------------------------------
extensions = [
    'sphinx.ext.autodoc',        # To build documentation from python source code docstrings.
    'sphinx.ext.napoleon',       # To read google-style docstrings (works with autodoc module).
    'sphinx_click',              # Must load before sphinx_autodoc_typehints to avoid mock import shadowing.
    'sphinx_autodoc_typehints',  # To parse typehints into documentation
    'breathe',                   # To read doxygen-generated xml files (to parse C++ documentation).
]

templates_path = ['_templates']
exclude_patterns = []

# Breathe configuration
breathe_projects = {"<PROJECT_NAME>": "./doxygen/xml"}
breathe_default_project = "<PROJECT_NAME>"

# Google-style docstring parsing configuration for napoleon extension
napoleon_google_docstring = True
napoleon_numpy_docstring = False
napoleon_include_init_with_doc = False
napoleon_include_private_with_doc = False
napoleon_include_special_with_doc = False
napoleon_use_admonition_for_examples = False
napoleon_use_admonition_for_notes = False
napoleon_use_admonition_for_references = False
napoleon_use_ivar = False
napoleon_use_param = True
napoleon_use_rtype = True

# Additional sphinx-typehints configuration
sphinx_autodoc_typehints = True
always_document_param_types = False
typehints_document_rtype = True
typehints_use_rtype = True
typehints_defaults = 'comma'
simplify_optional_unions = True
typehints_formatter = None
typehints_use_signature = False
typehints_use_signature_return = False

# -- Options for HTML output -------------------------------------------------
html_theme = 'furo'
```

### Placeholders

| Placeholder      | Description                          | Example           |
|------------------|--------------------------------------|-------------------|
| `<PROJECT_NAME>` | Project name matching pyproject.toml | `ataraxis-time`   |
| `<YEAR>`         | Current copyright year               | `2026`            |
| `<AUTHOR>`       | Author name(s)                       | `Ivan Kondratyev` |
