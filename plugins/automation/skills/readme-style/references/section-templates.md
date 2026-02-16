# Section templates

Exact templates for every README section. Load this reference when creating or updating README
files.

---

## Title and one-line description

```markdown
# project-name

Supports tox-based development automation pipelines used by other Sun (NeuroAI) lab projects.
```

The title must match the repository and package name (lowercase, hyphenated). The one-line
description MUST be the bare project description — the same sentence used in all other canonical
description locations for the project archetype. No language prefix ("A Python library that...")
and no project name prefix ("project-name is...").

The canonical description locations vary by archetype:

| Archetype              | Canonical locations                                             |
|------------------------|-----------------------------------------------------------------|
| Python-only            | `pyproject.toml`, `__init__.py`, `welcome.rst`, `README.md`    |
| Python + C++ extension | `pyproject.toml`, `__init__.py`, `welcome.rst`, `README.md`    |
| C++ PlatformIO library | `library.json`, `welcome.rst`, `README.md`                     |
| C++ PlatformIO firmware| `welcome.rst`, `README.md`                                     |
| C# Unity               | `README.md`                                                    |

---

## Badges

### Python libraries

Python libraries use the following 8 badges in this exact order. A blank line must separate
the one-line description from the first badge.

```markdown
![PyPI - Version](https://img.shields.io/pypi/v/PACKAGE-NAME)
![PyPI - Python Version](https://img.shields.io/pypi/pyversions/PACKAGE-NAME)
[![uv](https://tinyurl.com/uvbadge)](https://github.com/astral-sh/uv)
[![Ruff](https://tinyurl.com/ruffbadge)](https://github.com/astral-sh/ruff)
![type-checked: mypy](https://img.shields.io/badge/type--checked-mypy-blue?style=flat-square&logo=python)
![PyPI - License](https://img.shields.io/pypi/l/PACKAGE-NAME)
![PyPI - Status](https://img.shields.io/pypi/status/PACKAGE-NAME)
![PyPI - Wheel](https://img.shields.io/pypi/wheel/PACKAGE-NAME)
```

Replace `PACKAGE-NAME` with the actual PyPI package name (e.g., `sl-shared-assets`,
`ataraxis-time`).

### C++ / PlatformIO libraries

```markdown
[![PlatformIO Registry](https://badges.registry.platformio.org/packages/ORG/library/PACKAGE.svg)](https://registry.platformio.org/libraries/ORG/PACKAGE)
![C++](https://img.shields.io/badge/C%2B%2B-blue?logo=cplusplus&logoColor=white&labelColor=grey)
![Arduino](https://img.shields.io/badge/Arduino-blue?logo=Arduino&logoColor=white&labelColor=grey)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
```

### Other project types

For MATLAB projects, use `matlab` and `license` badges. For Unity/C# projects, use `C#`, `Unity`,
and license badges. Match the badge URLs from existing Sun Lab projects of the same type.

---

## Horizontal rule after badges

A triple underscore horizontal rule separates the header block (title, description, badges) from
the body content. This must immediately follow the last badge line:

```markdown
![PyPI - Wheel](https://img.shields.io/pypi/wheel/PACKAGE-NAME)

___
```

---

## Detailed description

An expanded explanation of the library's purpose, typically 2-4 sentences. Placed immediately
after the horizontal rule under a `## Detailed Description` heading:

```markdown
___

## Detailed Description

This library provides the shared automation pipeline for all Sun Lab Python projects. It abstracts
project environment manipulation and facilitates development tasks such as linting, typing,
testing, documentation, and building through a unified CLI interface.
```

---

## Features

Optional. When present, use a bulleted list of key capabilities. The **last bullet** must state
the license type:

```markdown
## Features

- Automated environment creation and management via mamba and uv.
- Unified CLI for linting, type checking, testing, and documentation builds.
- Cross-platform support for Windows, Linux, and macOS.
- Apache 2.0 License.
```

---

## Table of contents

Link to all H2 sections using lowercase Markdown anchors. Always spell "Acknowledgments" (not
"Acknowledgements"). Nest H3 subsections when present:

```markdown
## Table of Contents

- [Dependencies](#dependencies)
- [Installation](#installation)
- [Usage](#usage)
  - [CLI Commands](#cli-commands)
  - [MCP Server](#mcp-server)
- [API Documentation](#api-documentation)
- [Developers](#developers)
- [Versioning](#versioning)
- [Authors](#authors)
- [License](#license)
- [Acknowledgments](#acknowledgments)
```

Include only sections that exist in the README. Omit nested entries for subsections that are not
present.

---

## Dependencies

For Python libraries where all dependencies are automatically installed:

```markdown
## Dependencies

For users, all library dependencies are installed automatically by all supported installation
methods. For developers, see the [Developers](#developers) section for information on installing
additional development dependencies.
```

For libraries with external (non-pip) dependencies, list them before the standard text:

```markdown
## Dependencies

This library requires [FFmpeg](https://ffmpeg.org/) to be installed on the system for video
encoding and decoding functionality.

For users, all other library dependencies are installed automatically by all supported
installation methods. For developers, see the [Developers](#developers) section for information
on installing additional development dependencies.
```

---

## Installation

Python libraries use two subsections: Source and pip.

### Source subsection

```markdown
## Installation

### Source

***Note,*** installation from source is ***highly discouraged*** for anyone who is not an active
project developer. If possible, use the pip installation method described below.

1. Download this repository to your local machine using your preferred method, such as
   [git clone](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository).
2. `cd` into the root directory of the project using your CLI of choice.
3. Run `pip install .` to install the project and its dependencies.
```

### pip subsection

````markdown
### pip

Use the following command to install the library and all of its dependencies via
[pip](https://pip.pypa.io/en/stable/):

```
pip install PACKAGE-NAME
```
````

Replace `PACKAGE-NAME` with the actual PyPI package name.

---

## Usage

The Usage section structure depends on the library type. Common subsections include:

- **API usage examples**: Show minimal working code with expected output
- **CLI Commands**: Document each command (see below)
- **MCP Server**: Document AI agent integration (see below)

Keep examples minimal and link to full documentation for advanced usage.

---

## CLI commands

For libraries with CLI interfaces, document commands using a brief overview followed by a
command table:

```markdown
### CLI Commands

This library provides the `COMMAND` CLI that exposes the following commands:

| Command     | Description                                      |
|-------------|--------------------------------------------------|
| `discover`  | Discovers all compatible cameras on the system   |
| `record`    | Starts a video recording session                 |
| `benchmark` | Runs camera performance benchmarks               |

Use `COMMAND --help` or `COMMAND SUBCOMMAND --help` for detailed usage information.
```

For complex CLIs with many commands, add subsections for each command or command group after the
overview table.

---

## MCP server

Libraries that provide MCP servers document this functionality under Usage. Always use the section
title "MCP Server" (or "MCP Servers" when the library exposes more than one server). Do not use
"MCP Server (Agentic Integration)" or other variants.

````markdown
### MCP Server

This library provides an MCP server that exposes BRIEF DESCRIPTION for AI agent integration.

#### Starting the Server

Start the MCP server using the CLI:

```
COMMAND mcp
```

#### Available Tools

| Tool                  | Description                                    |
|-----------------------|------------------------------------------------|
| `list_cameras`        | Discovers all compatible cameras on the system |
| `start_video_session` | Starts a video capture session                 |
| `stop_video_session`  | Stops the active video session                 |

#### Claude Desktop Configuration

Add the following to the Claude Desktop configuration file:

```json
{
  "mcpServers": {
    "PACKAGE-NAME": {
      "command": "COMMAND",
      "args": ["mcp"]
    }
  }
}
```
````

Always use a table for the Available Tools section (not a bullet list). Replace `COMMAND` with
the actual CLI command and `PACKAGE-NAME` with the package name.

---

## API documentation

```markdown
## API Documentation

See the [API documentation](https://PACKAGE-NAME-api-docs.netlify.app/) for the detailed
description of the methods and classes exposed by components of this library.
```

Replace the URL with the actual documentation URL. Sun Lab documentation links follow the pattern
`https://PACKAGE-NAME-api-docs.netlify.app/`.

---

## Developers

Optional. When present, Python libraries use the following standard template. Adapt the tox
environments table to match the actual tox configuration of the project.

````markdown
## Developers

This section provides installation, dependency, and build-system instructions for the developers
that want to modify the source code of this library.

### Installing the Project

The first step is to create a [mamba](https://mamba.readthedocs.io/en/latest/) environment using
one of the environment files stored in the `envs` directory of this project. Alternatively, use
an existing `mamba` environment that has the minimum version of Python required by this library
installed.

***Note,*** this project was developed and tested using the `mamba` Python environment manager.
Use other environment managers at your own risk.

After creating or activating the environment, `cd` into the root directory of the project and
install the project and all of its development dependencies by running:

```
pip install -e .[dev]
```

***Note,*** this is different from the user installation instructions.

### Additional Dependencies

Some development tools used by this project are not installable via pip and have to be installed
separately:

1. Install [tox](https://tox.wiki/en/stable/) by running `pip install tox`.
2. Install project-specific linters and formatters by running `tox -e lint`.

### Development Automation

This project uses `tox` for development automation. The following tox environments are available:

| Environment | Description                         |
|-------------|-------------------------------------|
| `lint`      | Runs ruff linting and mypy typing   |
| `test`      | Runs the test suite via pytest      |
| `docs`      | Builds the documentation via Sphinx |
| `build`     | Builds the distribution packages    |

Run any environment using `tox -e ENVIRONMENT`. For example, `tox -e lint`.

### Automation Troubleshooting

Many `tox` automation commands cache intermediate results to speed up repeated execution. On rare
occasions, invalid cache can break the automation pipeline. If you encounter inexplicable errors
when running one of the `tox` commands, clear the cache by running `tox -e clean` and try again.
````

---

## Standard ending sections

The final four sections appear in every README in this exact order.

### Versioning

```markdown
## Versioning

This project uses [semantic versioning](https://semver.org/). See the
[tags on this repository](https://github.com/Sun-Lab-NBB/PROJECT-NAME/tags) for the available
project releases.
```

### Authors

```markdown
## Authors

- Author Name ([GitHubHandle](https://github.com/handle))
```

List all contributors. Include GitHub profile links where available.

### License

```markdown
## License

This project is licensed under the Apache 2.0 License: see the [LICENSE](LICENSE) file for
details.
```

### Acknowledgments

For Python libraries:

```markdown
## Acknowledgments

- All Sun lab [members](https://neuroai.github.io/sunlab/people) for providing the inspiration
  and comments during the development of this library.
- The creators of all other dependencies and projects listed in the
  [pyproject.toml](pyproject.toml) file.
```

For C++ / PlatformIO libraries, replace `pyproject.toml` with `platformio.ini`:

```markdown
- The creators of all other dependencies and projects listed in the
  [platformio.ini](platformio.ini) file.
```

Additional project-specific acknowledgments may be added between the two standard bullets.
