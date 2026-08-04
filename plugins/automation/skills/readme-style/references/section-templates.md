# Section templates

Exact templates for every README section. Load this reference when creating or updating README
files. Every template's prose is filled to the 120 character limit, under the wrap-width rule the
skill file defines.

---

## Title and one-line description

```markdown
# project-name

Supports tox-based development automation pipelines used by other ataraxis framework projects.
```

The title must match the repository and package name (lowercase, hyphenated). The one-line
description MUST be the bare project description, the same sentence used in all other canonical
description locations for the project archetype. No language prefix ("A Python library that...")
and no project name prefix ("project-name is...").

This template covers library READMEs. An umbrella README titles itself with the framework name in
its display casing and follows the title with a bold tagline, so it has no package name to match and
no other canonical location to stay in sync with. See the umbrella README order in the skill file.

The canonical description locations vary by archetype:

| Archetype               | Canonical locations                                         |
|-------------------------|-------------------------------------------------------------|
| Python-only             | `pyproject.toml`, `__init__.py`, `welcome.rst`, `README.md` |
| Python + C++ extension  | `pyproject.toml`, `__init__.py`, `welcome.rst`, `README.md` |
| C++ PlatformIO library  | `library.json`, `welcome.rst`, `README.md`                  |
| C++ PlatformIO firmware | `welcome.rst`, `README.md`                                  |
| C# Unity                | `README.md`                                                 |

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

Replace `PACKAGE-NAME` with the actual PyPI package name (e.g., `ataraxis-time`).

### C++ / PlatformIO libraries

```markdown
[![PlatformIO Registry](https://badges.registry.platformio.org/packages/ORG/library/PACKAGE.svg)](https://registry.platformio.org/libraries/ORG/PACKAGE)
![C++](https://img.shields.io/badge/C%2B%2B-blue?logo=cplusplus&logoColor=white&labelColor=grey)
![Arduino](https://img.shields.io/badge/Arduino-blue?logo=Arduino&logoColor=white&labelColor=grey)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
```

### Umbrella / marketplace repositories

An umbrella repository publishes no package, so it carries the license badge alone:

```markdown
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
```

### Other project types

For MATLAB projects, use `matlab` and `license` badges. For Unity/C# projects, use `C#`, `Unity`,
and license badges. Match the badge URLs from existing projects of the same type.

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
after the horizontal rule under a `## Detailed Description` heading.

For Ataraxis ecosystem libraries, end the detailed description with a sentence linking to the
main project: "This library is part of the
[Ataraxis](https://github.com/Sun-Lab-NBB/ataraxis) framework for AI-assisted scientific
hardware control."

```markdown
___

## Detailed Description

This library provides the shared automation pipeline for all Python projects. It abstracts project environment
manipulation and facilitates development tasks such as linting, typing, testing, documentation, and building through
a unified CLI interface. This library is part of the [Ataraxis](https://github.com/Sun-Lab-NBB/ataraxis) framework
for AI-assisted scientific hardware control.
```

---

## Features

Optional. When present, use a bulleted list of key capabilities. The **first bullet** states the
supported platforms/OS ("Supports Windows, Linux, and macOS." for PC libraries, "Supports all
recent Arduino and Teensy architectures and platforms." for microcontroller libraries). The
**last bullet** must state the license type:

```markdown
## Features

- Supports Windows, Linux, and macOS.
- Unified CLI for linting, type checking, testing, and documentation builds.
- Automated environment creation and management via mamba and uv.
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

For users, all library dependencies are installed automatically by all supported installation methods. For developers,
see the [Developers](#developers) section for information on installing additional development dependencies.
```

For libraries with external (non-pip) dependencies, list them before the standard text:

```markdown
## Dependencies

This library requires [FFmpeg](https://ffmpeg.org/) to be installed on the system for video encoding and decoding
functionality.

For users, all other library dependencies are installed automatically by all supported installation methods. For
developers, see the [Developers](#developers) section for information on installing additional development
dependencies.
```

---

## Installation

Python libraries use two subsections: Source and pip. Short shell commands (such as
`pip install PACKAGE-NAME`, `pip install .`, `tox -e lint`) should be inlined with backticks
rather than placed in fenced code blocks. Reserve fenced code blocks for multi-line code samples
such as Python usage examples.

### Source subsection

```markdown
## Installation

### Source

***Note,*** installation from source is ***highly discouraged*** for anyone who is not an active project developer.

1. Download this repository to the local machine using the preferred method, such as git-cloning. Use one of the
   [stable releases](https://github.com/Sun-Lab-NBB/PROJECT-NAME/tags) that include precompiled binary and source code
   distribution (sdist) wheels.
2. If the downloaded distribution is stored as a compressed archive, unpack it using the appropriate decompression
   tool.
3. `cd` to the root directory of the prepared project distribution.
4. Run `pip install .` to install the project and its dependencies.
```

Replace `PROJECT-NAME` with the actual repository name.

### pip subsection

````markdown
### pip

Use the following command to install the library and all of its dependencies via [pip](https://pip.pypa.io/en/stable/):
`pip install PACKAGE-NAME`
````

Replace `PACKAGE-NAME` with the actual PyPI package name.

### C++ / PlatformIO libraries

C++ PlatformIO libraries use a Source and a `### Platformio` subsection (not `### pip`). The
Source subsection moves the distribution's `src` contents into the consuming project's directory
and adds the relevant `#include` directives. The Platformio subsection declares the dependency via
`lib_deps`:

```markdown
## Installation

### Source

***Note,*** installation from source is ***highly discouraged*** for anyone who is not an active project developer.

1. Download this repository to the local machine using the preferred method, such as git-cloning. Use one of the
   [stable releases](https://github.com/Sun-Lab-NBB/PROJECT-NAME/tags).
2. Unpack the downloaded tarball and move all 'src' contents into the appropriate destination ('include,' 'src,' or
   'libs') directory of the project that needs to use this library.
3. Add the project's `#include` directives at the top of the main.cpp file and each consuming header file.

### Platformio

1. Navigate to the project's platformio.ini file and add the following line to the target environment specification:
   `lib_deps = inkaros/PACKAGE-NAME@^MAJOR.0.0`.
2. Add the project's `#include` directives at the top of the main.cpp file and each consuming header file.
```

Replace `PROJECT-NAME` with the repository name, `PACKAGE-NAME` with the PlatformIO registry
package name, and `MAJOR` with the current major version.

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

```bash
COMMAND mcp
```

#### Available Tools

| Tool                  | Description                                    |
|-----------------------|------------------------------------------------|
| `list_cameras`        | Discovers all compatible cameras on the system |
| `start_video_session` | Starts a video capture session                 |
| `stop_video_session`  | Stops the active video session                 |

#### Client Registration

MCP server registration and Claude Code skill assets for this library are distributed through the
[ataraxis](https://github.com/Sun-Lab-NBB/ataraxis) marketplace as part of the **PLUGIN-NAME** plugin. Install the
plugin from the marketplace to automatically register the MCP server with compatible clients and make all associated
skills available.
````

Always use a table for the Available Tools section (not a bullet list). Replace `COMMAND` with
the actual CLI command and `PLUGIN-NAME` with the name of the ataraxis marketplace plugin that
distributes the assets for this library.

---

## API documentation

```markdown
## API Documentation

See the [API documentation](https://PACKAGE-NAME-api-docs.netlify.app/) for the detailed description of the methods and
classes exposed by components of this library.
```

Replace the URL with the actual documentation URL. Documentation links follow the pattern
`https://PACKAGE-NAME-api-docs.netlify.app/`.

---

## AI-Assisted Development as an H2 section

The AI-Assisted Development section is an H3 under Developers by default, where it tells a
contributor which plugin carries the project's agent assets. A project whose primary interface is an
agent, meaning the documented path to using it runs through its MCP server and skills rather than
through its Python API, promotes the section to H2 and places it immediately after API Documentation.
Promotion moves the section to where a user reads rather than where a contributor reads, and the
title stays `AI-Assisted Development` at either level.

A promoted section carries what its readers need to start, which is the plugin name, the tools or
skills the plugin exposes, and the client registration step. Keep the tool listing to a table and
link the API documentation for everything past the first working result, exactly as the Usage section
does.

---

## Developers

Optional. When present, Python libraries use the following standard template. Adapt the tox
environments table and Additional Dependencies section to match the actual tox configuration
and requirements of the project, whose environment set `/tox-config` owns.

````markdown
## Developers

This section provides installation, dependency, and build-system instructions for the developers that want to modify
the source code of this library.

### Installing the Project

***Note,*** this installation method requires **mamba version 2.3.2 or above**. Currently, all automation pipelines
require that mamba is installed through the [miniforge3](https://github.com/conda-forge/miniforge) installer.

1. Download this repository to the local machine using the preferred method, such as git-cloning.
2. If the downloaded distribution is stored as a compressed archive, unpack it using the appropriate decompression
   tool.
3. `cd` to the root directory of the prepared project distribution.
4. Install the core development dependencies into the ***base*** mamba environment via the
   `mamba install tox uv tox-uv` command.
5. Use the `tox -e create` command to create the project-specific development environment followed by `tox -e install`
   command to install the project into that environment as a library.

### Additional Dependencies

In addition to installing the project and all user dependencies, install the following dependencies:

1. [Python](https://www.python.org/downloads/) distributions, one for each version supported by the developed project.
   Currently, this library supports the three latest stable versions. It is recommended to use a tool like
   [pyenv](https://github.com/pyenv/pyenv) to install and manage the required versions.

### Development Automation

This project uses `tox` for development automation. The following tox environments are available:

| Environment          | Description                                                  |
|----------------------|--------------------------------------------------------------|
| `lint`               | Runs ruff formatting, ruff linting, and mypy type checking   |
| `stubs`              | Generates py.typed marker and .pyi stub files                |
| `{py312,...}-test`   | Runs the test suite via pytest for each supported Python     |
| `coverage`           | Aggregates test coverage and applies the 100% coverage gate  |
| `docs`               | Builds the API documentation via Sphinx                      |
| `build`              | Builds sdist and wheel distributions                         |
| `upload`             | Uploads distributions to PyPI via twine                      |
| `deploy`             | Uploads the built documentation to the Netlify site          |
| `install`            | Builds and installs the project into its mamba environment   |
| `uninstall`          | Uninstalls the project from its mamba environment            |
| `create`             | Creates the project's mamba development environment          |
| `remove`             | Removes the project's mamba development environment          |
| `provision`          | Recreates the mamba environment from scratch                 |
| `export`             | Exports the mamba environment as a .yml file                 |
| `import`             | Creates or updates the mamba environment from a .yml file    |

Run any environment using `tox -e ENVIRONMENT`. For example, `tox -e lint`.

***Note,*** all pull requests for this project have to successfully complete the `tox` task before being merged. To
expedite the task's runtime, use the `tox --parallel` command to run some tasks in parallel.

### AI-Assisted Development

Claude Code skills and other AI development assets for this project are distributed through the
[ataraxis](https://github.com/Sun-Lab-NBB/ataraxis) marketplace as part of the **automation** plugin. Install the
plugin from the marketplace to make all associated skills and development tools available to compatible AI coding
agents.

### Automation Troubleshooting

Many packages used in `tox` automation pipelines (uv, mypy, ruff) and `tox` itself may experience runtime failures. In
most cases, this is related to their caching behavior. If an unintelligible error is encountered with any of the
automation components, deleting the corresponding cache directories (`.tox`, `.ruff_cache`, `.mypy_cache`, etc.)
manually or via a CLI command typically resolves the issue.
````

### C++ / PlatformIO variant

C++ PlatformIO projects differ from the Python template above:

- **Installing the Project**: install the [PlatformIO](https://platformio.org/install/integration)
  IDE/plugin instead of mamba/tox, then run `pio project init` and (for IDEs lacking native
  PlatformIO support) `pio project metadata`.
- **Additional Dependencies**: Tox and Doxygen (both on the system path), used to build the API
  documentation.
- **Development Automation**: driven by the PlatformIO CLI, with tox reserved for tasks not covered
  by PlatformIO, such as API documentation generation.
- **PR gate**: the `tox`, `pio check`, and `pio test` tasks must all pass before merging.

---

## Standard ending sections

The final four sections close every library README in this exact order. An umbrella README closes with
License and Acknowledgments alone, because the indexed repositories tag their own releases and the
header attribution block already names the authors.

### Versioning

```markdown
## Versioning

This project uses [semantic versioning](https://semver.org/). See the
[tags on this repository](https://github.com/Sun-Lab-NBB/PROJECT-NAME/tags) for the available project releases.
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

This project is licensed under the Apache 2.0 License: see the [LICENSE](LICENSE) file for details.
```

### Acknowledgments

For Python libraries:

```markdown
## Acknowledgments

- All individuals who contributed to the development of this library, directly or indirectly.
- The creators of all other dependencies and projects listed in the [pyproject.toml](pyproject.toml) file.
```

For C++ / PlatformIO libraries, replace `pyproject.toml` with `platformio.ini`:

```markdown
- The creators of all other dependencies and projects listed in the [platformio.ini](platformio.ini) file.
```

Additional project-specific acknowledgments may be added between the two standard bullets. A library
Acknowledgments section credits dependency developers and contributors, so it names no institution and
links to no institutional page. An umbrella Acknowledgments section credits the projects the framework
builds on and ends with a contact address.
