# Project metadata reference

Detailed specifications for all fields in the `[project]` table and related sections.

---

## Field specifications

### name

Lowercase, hyphenated format matching the GitHub repository name:

```toml
name = "ataraxis-automation"
name = "sl-shared-assets"
```

The import name uses underscores (`ataraxis_automation`), while the package name uses hyphens
(`ataraxis-automation`).

### version

Static semantic versioning in `X.Y.Z` format. Release candidates use `X.Y.ZrcN`:

```toml
version = "7.1.0"
version = "2.0.0rc9"
```

Sun Lab projects use static versioning (hard-coded in pyproject.toml). To access the version at
runtime, use `importlib.metadata`:

```python
from importlib.metadata import version
__version__ = version("package-name")
```

### description

A single sentence describing the project's purpose. The description MUST use bare form — no
language prefix ("A Python library that...") and no project name prefix ("project-name is...").
Start directly with an imperative verb (e.g., "Supports...", "Provides...", "Manages...").

This exact description MUST be used verbatim in every canonical description location that exists
for the project archetype. For Python projects, these are:
1. `pyproject.toml` `description` field
2. Top-level `__init__.py` module docstring (first line)
3. `docs/source/welcome.rst` first paragraph
4. `README.md` one-line description (immediately after the title)

```toml
description = "Supports tox-based development automation pipelines and provides agentic skills for Claude Code used by other Sun (NeuroAI) lab projects."
```

### readme

Always points to `README.md`:

```toml
readme = "README.md"
```

### license (PEP 639)

Uses an SPDX license expression string. The `license-files` field specifies the license file:

```toml
# GPL (most Sun Lab projects)
license = "GPL-3.0-or-later"
license-files = ["LICENSE"]

# Apache (ataraxis-automation)
license = "Apache-2.0"
license-files = ["LICENSE"]

# BSD (pirt)
license = "BSD-3-Clause"
license-files = ["LICENSE"]
```

Do NOT use the deprecated table format (`license = { file = "LICENSE" }`). When using SPDX
license expressions, do NOT include `License ::` trove classifiers — they are redundant and
deprecated under PEP 639.

### requires-python

Specifies the supported Python version range. The constraint style depends on the project type:

```toml
# Core libraries (ataraxis-*): Support multiple Python versions
requires-python = ">=3.12,<3.15"

# Applications (sl-*): Target a single Python version
requires-python = ">=3.14,<3.15"
```

### authors and maintainers

Authors lists contributors. Maintainers includes the active maintainer with an optional email:

```toml
authors = [
    { name = "Author Name" },
]
maintainers = [
    { name = "Maintainer Name", email = "maintainer@example.com" },
]
```

The email field in `maintainers` is optional. When present, use any valid email address.

For multi-author projects, list all contributors:

```toml
authors = [
    { name = "Author One" },
    { name = "Author Two" },
    { name = "Author Three" },
]
```

### keywords

Repository-specific tags as a single-line or multi-line array:

```toml
keywords = ["automation", "tox", "development-tools", "ataraxis", "sunlab"]
```

Include both domain-specific terms and organizational identifiers (`"ataraxis"`, `"sunlab"`).

---

## Classifiers

Group classifiers with category comments. The following categories are required in every project:

### Development status

All production Sun Lab projects use:

```toml
# Development status
"Development Status :: 5 - Production/Stable",
```

### Intended audience

```toml
# Core libraries (ataraxis-*)
"Intended Audience :: Developers",
"Topic :: Software Development",

# Scientific applications (sl-*)
"Intended Audience :: Science/Research",
"Topic :: Scientific/Engineering",
```

### Python versions

List each supported minor version individually:

```toml
# Core libraries
"Programming Language :: Python :: 3.12",
"Programming Language :: Python :: 3.13",
"Programming Language :: Python :: 3.14",

# Single-version applications
"Programming Language :: Python :: 3.14",
```

### Operating systems

```toml
# Multi-platform projects
"Operating System :: Microsoft :: Windows",
"Operating System :: POSIX :: Linux",
"Operating System :: MacOS :: MacOS X",

# Platform-independent projects
"Operating System :: OS Independent",

# Linux-only projects
"Operating System :: POSIX :: Linux",
```

### Typing

All Sun Lab projects are typed:

```toml
"Typing :: Typed"
```

### Additional classifiers

Projects with C/C++ extensions include the language classifier:

```toml
"Programming Language :: C++",
```

### Complete classifier template

```toml
classifiers = [
    # Development status
    "Development Status :: 5 - Production/Stable",
    # Intended audience and project topic
    "Intended Audience :: Developers",
    "Topic :: Software Development",
    # Supported Python Versions
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
    "Programming Language :: Python :: 3.14",
    # Supported Operating Systems
    "Operating System :: Microsoft :: Windows",
    "Operating System :: POSIX :: Linux",
    "Operating System :: MacOS :: MacOS X",
    # Typing
    "Typing :: Typed"
]
```

Do NOT include `License ::` classifiers. The license is declared via the `license` field using
a PEP 639 SPDX expression.

---

## Dependencies

### Runtime dependencies

Group dependencies by category with comments. Common categories:

```toml
dependencies = [
    # Automation Logic
    "click>=8,<9",
    "tomli>=2,<3",

    # Testing
    "pytest>=9,<10",
    "pytest-cov>=7,<8",
    "pytest-xdist>=3,<4",

    # Documentation
    "sphinx>=8,<9",
    "furo>=2024,<2027",

    # Linting and Stub Generation
    "mypy>=1,<2",
    "ruff>=0,<1",
]
```

### Ataraxis cross-dependencies

Projects depending on other Sun Lab libraries use the same range constraint pattern:

```toml
"ataraxis-time>=5,<6",
"ataraxis-base-utilities>=5,<6",
"ataraxis-data-structures>=5,<6",
"sl-shared-assets>=7,<8",
```

### Platform-specific dependencies

Use `sys_platform` markers for platform-conditional dependencies:

```toml
"intel-cmplr-lib-rt>=2025,<2026; sys_platform != 'darwin'",
"tbb4py>=2022,<2023; sys_platform != 'darwin'",
```

---

## Dependency groups (PEP 735)

Development dependencies use `[dependency-groups]` instead of `[project.optional-dependencies]`.
Dependency groups are NOT published to PyPI, which is semantically correct for development-only
tooling.

### dev group

The `dev` group contains all development-only dependencies. All Sun Lab projects include:

```toml
# Development dependencies (PEP 735). Not published to PyPI.
[dependency-groups]
dev = [
    # Tox
    "tox>=4,<5",
    "uv>=0,<1",
    "tox-uv>=1,<2",

    # Development Automation
    "ataraxis-automation>=7,<8",

    # Types
    "types-tqdm>=4,<5",
    "scipy-stubs>=1,<2",
]
```

Install via: `uv sync --group dev` or `pip install --group dev`

The type stub packages vary by project. Include stubs for any dependency that does not ship
inline types. Common type stubs:

| Stub package                           | For dependency   |
|----------------------------------------|------------------|
| `types-tqdm`                           | tqdm             |
| `types-paho-mqtt`                      | paho-mqtt        |
| `types-pyserial`                       | pyserial         |
| `types-pyyaml`                         | PyYAML           |
| `types-appdirs`                        | appdirs          |
| `types-tabulate`                       | tabulate         |
| `types-filelock`                       | filelock         |
| `scipy-stubs`                          | scipy            |
| `google-api-python-client-stubs`       | google-api       |

The `ataraxis-automation` project itself omits the `ataraxis-automation` dependency from its
dev group since it is self-referential.

Do NOT use `[project.optional-dependencies]` for development dependencies. That section is
reserved for user-facing optional feature extras (if any exist).

---

## URLs

### Standard URL fields

```toml
[project.urls]
Homepage = "https://github.com/Sun-Lab-NBB/project-name"
Documentation = "https://project-name-api-docs.netlify.app/"
```

`Homepage` is required and always points to the GitHub repository. `Documentation` is included
when hosted API documentation exists.

---

## Scripts

### Entry point format

```toml
[project.scripts]
command-name = "package_name.module:function"
```

### Naming conventions

- Use short, memorable command names
- Use hyphens for multi-word commands
- Prefix with project abbreviation for namespacing (e.g., `axci-id`, `axvs`, `ss2p`)

### Examples from Sun Lab projects

| Project                          | Command          | Entry point                                        |
|----------------------------------|------------------|----------------------------------------------------|
| ataraxis-automation              | `automation-cli` | `ataraxis_automation.cli:cli`                      |
| ataraxis-video-system            | `axvs`           | `ataraxis_video_system.cli:axvs_cli`               |
| ataraxis-communication-interface | `axci-id`        | `...microcontroller_interface:identify_interfaces` |
| sl-behavior                      | `sl-behavior`    | `sl_behavior.cli:cli`                              |
| sl-shared-assets                 | `sl-configure`   | `sl_shared_assets.interfaces.configure:configure`  |
| sl-suite2p                       | `ss2p`           | `sl_suite2p.interface.cli:ss2p`                    |

For Click-based CLIs, point to the Click group or command object directly.

