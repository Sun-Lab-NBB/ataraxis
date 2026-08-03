# Ataraxis

**Bridging AI Coding Assistants and Scientific Hardware**

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

Ataraxis is an open-source framework that enables AI coding assistants to interact with laboratory
hardware. It provides optimized hardware interface libraries, Model Context Protocol (MCP) servers
for structured device discovery, and domain-specific skills that encode expert workflows. AI agents
use these components to generate efficient data acquisition pipelines, configure systems, and
troubleshoot hardware issues.

**Core Insight:** AI assistance operates at *configuration time* while runtime data acquisition
remains *deterministic and AI-independent*. This separation ensures that network latency, API rate
limits, or model errors never disrupt a running experiment.

Authored by [Ivan Kondratyev](https://github.com/Inkaros).
Copyright: 2026, NeuroAI Lab, Cornell University.

___

## Features

### Hardware Discovery & Validation
- **MCP-based device enumeration**: AI agents can query connected cameras and microcontrollers
  through structured tool interfaces.
- **Pre-session diagnostics**: Validate hardware connectivity and configuration through natural
  language queries.
- **Real-time status checking**: Query device responsiveness, serial port status, and camera
  capabilities without manual debugging loops.

### Optimized Hardware Interfaces
- **High-speed camera acquisition**: Support for OpenCV and GeniCam cameras with real-time FFMPEG
  encoding (CPU/GPU).
- **Microcontroller communication**: Bidirectional serial communication with Arduino and Teensy
  boards at microsecond speeds.
- **Precision timing**: Microsecond-accurate timers using C++ chrono library bindings.
- **Inter-process data sharing**: Thread-safe shared memory arrays and scalable data logging.

### AI-Assisted Development
- **Code generation**: AI agents generate hardware interface code following established patterns.
- **Configuration management**: Interactive experiment configuration using task templates.
- **Domain-specific skills**: Reusable workflows for camera interfaces, microcontroller modules,
  and system health checks.
- **Cross-repository coordination**: Skills encode knowledge spanning multiple interdependent
  libraries.

### Deterministic Runtime
- **Static acquisition pipelines**: Experiments run independently of AI systems.
- **Validated configurations**: Pre-runtime parameter validation ensures reliable data collection.
- **Reproducible execution**: Configuration files capture complete experimental setups.

___

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Ataraxis Architecture                             │
├─────────────────────────────────┬───────────────────────────────────────────┤
│       Configuration Time        │           Runtime (No AI)                 │
├─────────────────────────────────┼───────────────────────────────────────────┤
│                                 │                                           │
│    ┌─────────────────────┐      │      ┌─────────────────────────┐          │
│    │  AI Agent (Claude)  │      │      │ Static Acquisition      │          │
│    └──────────┬──────────┘      │      │ Pipelines               │          │
│               │                 │      └────────────┬────────────┘          │
│               ▼                 │                   │                       │
│    ┌─────────────────────┐      │                   ▼                       │
│    │  Skills & MCP       │      │      ┌─────────────────────────┐          │
│    │  Discovery Tools    │ ─────┼────▶ │ Ataraxis Libraries      │          │
│    └──────────┬──────────┘      │      └────────────┬────────────┘          │
│               │                 │                   │                       │
│               ▼                 │                   ▼                       │
│    ┌─────────────────────┐      │      ┌─────────────────────────┐          │
│    │  Config Files &     │      │      │ Physical Hardware       │          │
│    │  Pipeline Code      │      │      └────────────┬────────────┘          │
│    └─────────────────────┘      │                   │                       │
│                                 │                   ▼                       │
│                                 │      ┌─────────────────────────┐          │
│                                 │      │ Session Data & Logs     │          │
│                                 │      └─────────────────────────┘          │
└─────────────────────────────────┴───────────────────────────────────────────┘
```

___

## Libraries

### Core Infrastructure

**[ataraxis-base-utilities](https://github.com/Sun-Lab-NBB/ataraxis-base-utilities)** (Python)
Shared utility assets providing a unified message/error processing framework (Console class) and
common utility functions for filesystem operations and parallel data processing.

**[ataraxis-automation](https://github.com/Sun-Lab-NBB/ataraxis-automation)** (Python)
Development automation pipelines using tox, providing CLI tools for environment management,
linting, typing, and documentation generation.

### Hardware Communication

**[ataraxis-communication-interface](https://github.com/Sun-Lab-NBB/ataraxis-communication-interface)** (Python)
Centralized interface for exchanging commands and data between Arduino/Teensy microcontrollers
and host computers. Includes MCP server for AI agent integration.

**[ataraxis-transport-layer-pc](https://github.com/Sun-Lab-NBB/ataraxis-transport-layer-pc)** (Python)
Transport layer implementation for host computers, providing bidirectional communication with
microcontrollers over USB/UART serial interfaces using COBS encoding and CRC verification.

**[ataraxis-transport-layer-mc](https://github.com/Sun-Lab-NBB/ataraxis-transport-layer-mc)** (C++)
Transport layer implementation for Arduino and Teensy microcontrollers, enabling bidirectional serial
communication with PC clients using COBS encoding and configurable CRC support.

**[ataraxis-micro-controller](https://github.com/Sun-Lab-NBB/ataraxis-micro-controller)** (C++)
Framework for integrating custom hardware modules with centralized PC control. Provides Kernel
and Communication classes with concurrent command execution.

### Data Acquisition & Processing

**[ataraxis-video-system](https://github.com/Sun-Lab-NBB/ataraxis-video-system)** (Python)
Camera interface library supporting OpenCV and GeniCam cameras with real-time FFMPEG video
encoding. Includes MCP server and CLI tools for camera management.

**[ataraxis-time](https://github.com/Sun-Lab-NBB/ataraxis-time)** (Python/C++)
High-precision thread-safe timers using C++ chrono bindings for microsecond accuracy. Includes
helper methods for time conversion and UTC timestamp handling.

**[ataraxis-data-structures](https://github.com/Sun-Lab-NBB/ataraxis-data-structures)** (Python)
Classes for storing, manipulating, and sharing data between processes. Includes SharedMemoryArray,
YamlConfig, and DataLogger for scalable multi-process data storage.

___

## Getting Started

### Installation

Core libraries are available via PyPI:

```bash
pip install ataraxis-video-system ataraxis-communication-interface ataraxis-time
pip install ataraxis-data-structures
```

C++ microcontroller libraries are available via PlatformIO:

```ini
lib_deps =
    Sun-Lab-NBB/ataraxis-micro-controller
    Sun-Lab-NBB/ataraxis-transport-layer-mc
```

___

## Claude Code Plugins

This repository serves as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin
marketplace. It distributes four plugins that together form the framework's agentic surface:
domain-specific *skills* that encode expert workflows and coding conventions, and *MCP servers* that
expose laboratory hardware to AI agents for structured discovery and control. Installing a plugin
makes its skills available to Claude Code and, for the plugins that bundle an MCP server, registers
that server automatically — no manual client configuration is required.

| Plugin            | MCP Server                         | Focus                                                                                             |
|-------------------|------------------------------------|---------------------------------------------------------------------------------------------------|
| `automation`      | —                                  | Development skills enforcing coding style, documentation, project structure, and git conventions. |
| `communication`   | `ataraxis-communication-interface` | Microcontroller communication, data extraction, and log processing workflows.                     |
| `video`           | `ataraxis-video-system`            | Camera acquisition, recording, and frame-timing log processing workflows.                         |
| `microcontroller` | —                                  | Firmware module implementation for custom microcontroller hardware.                               |

### Installation

Claude Code installs plugins through its built-in marketplace system. First, add the ataraxis
marketplace:

`/plugin marketplace add Sun-Lab-NBB/ataraxis`

Then install any combination of the four plugins:

`/plugin install automation@ataraxis`

`/plugin install communication@ataraxis`

`/plugin install video@ataraxis`

`/plugin install microcontroller@ataraxis`

Alternatively, run `/plugin`, open the **Discover** tab, and select the plugins to install
interactively.

***Note,*** installing the `communication` or `video` plugin automatically registers its bundled MCP
server (started via `axci mcp` and `axvs mcp`, respectively). No manual edits to `~/.claude.json` are
required. When a plugin is enabled mid-session, run `/reload-plugins` to connect its MCP server.

Each plugin can be installed at a different scope, depending on the intended use:
- **user** (default): available across all projects for the current user.
- **project**: shared with all developers via version control (stored in `.claude/settings.json`).
- **local**: project-specific and gitignored (stored in `.claude/settings.local.json`).

To select a scope during installation, use the CLI form:
`claude plugin install automation@ataraxis --scope project`.

### Skill Invocation

Most skills are deliberately **not** user-invocable. They encode background conventions and workflow
knowledge that AI coding agents pick up and apply automatically whenever a task matches the skill's
description — there is nothing to type. The per-plugin tables below enumerate these skills for
reference.

A small subset of skills is user-invocable: they perform discrete, on-demand actions and can be
typed directly as slash commands. All of them are provided by the `automation` plugin.

| Command                 | Description                                                            |
|-------------------------|------------------------------------------------------------------------|
| `/explore-codebase`     | Performs in-depth codebase exploration at the start of a session       |
| `/explore-dependencies` | Explores installed ataraxis library APIs for dependency awareness      |
| `/audit-project`        | Orchestrates the four audits and merges their findings into one report |
| `/audit-facts`          | Audits documentation and in-source docstrings for factual accuracy     |
| `/audit-style`          | Audits files against applicable style skill checklists for compliance  |
| `/audit-correctness`    | Audits source code for active and latent bugs the tests leave uncaught |
| `/audit-performance`    | Audits source code for optimization and dtype predictability findings  |
| `/commit`               | Drafts style-compliant git commit messages                             |
| `/pr`                   | Drafts a style-compliant pull request summary for the active branch    |
| `/release`              | Drafts style-compliant release notes summarizing merged pull requests  |

### automation

Shared development-convention skills used across Ataraxis and derived projects. This plugin does not
provide an MCP server.

| Skill                  | Description                                                            |
|------------------------|------------------------------------------------------------------------|
| `explore-codebase`     | Performs in-depth codebase exploration at the start of a session       |
| `explore-dependencies` | Explores installed ataraxis library APIs for dependency awareness      |
| `python-style`         | Applies ataraxis framework Python coding conventions                   |
| `cpp-style`            | Applies ataraxis framework C++ coding conventions                      |
| `csharp-style`         | Applies ataraxis framework C# coding conventions                       |
| `readme-style`         | Applies ataraxis framework README conventions                          |
| `pyproject-style`      | Applies ataraxis framework pyproject.toml conventions                  |
| `api-docs`             | Applies ataraxis framework API documentation conventions               |
| `project-layout`       | Applies ataraxis framework project directory structure conventions     |
| `tox-config`           | Applies ataraxis framework tox.ini conventions                         |
| `platformio-config`    | Applies ataraxis framework platformio.ini and library.json conventions |
| `audit-project`        | Orchestrates the four audits and merges their findings into one report |
| `audit-facts`          | Audits documentation and in-source docstrings for factual accuracy     |
| `audit-style`          | Audits files against applicable style skill checklists for compliance  |
| `audit-correctness`    | Audits source code for active and latent bugs the tests leave uncaught |
| `audit-performance`    | Audits source code for optimization and dtype predictability findings  |
| `commit`               | Drafts style-compliant git commit messages                             |
| `pr`                   | Drafts a style-compliant pull request summary for the active branch    |
| `release`              | Drafts style-compliant release notes summarizing merged pull requests  |
| `skill-design`         | Generates, updates, and verifies skill files and CLAUDE.md             |

### communication

Microcontroller communication and data-processing skills for
[ataraxis-communication-interface](https://github.com/Sun-Lab-NBB/ataraxis-communication-interface).
Bundles the `ataraxis-communication-interface` MCP server for hardware discovery, manifest
management, and batch log processing.

| Skill                                 | Description                                                            |
|---------------------------------------|------------------------------------------------------------------------|
| `microcontroller-interface`           | Guides MicroControllerInterface, ModuleInterface, and MQTT setup       |
| `microcontroller-setup`               | Discovers microcontrollers and manages manifests via MCP tools         |
| `extraction-configuration`            | Creates and validates ExtractionConfig for the log processing pipeline |
| `log-input-format`                    | Documents the NPZ log archive input format and source ID semantics     |
| `log-processing`                      | Orchestrates batch log processing via the MCP server                   |
| `log-processing-results`              | Interprets extracted event data and timing statistics                  |
| `pipeline`                            | Orchestrates the end-to-end data acquisition and analysis pipeline     |
| `communication-mcp-environment-setup` | Diagnoses and resolves MCP server connectivity issues                  |

### video

Camera acquisition and frame-timing skills for
[ataraxis-video-system](https://github.com/Sun-Lab-NBB/ataraxis-video-system). Bundles the
`ataraxis-video-system` MCP server for camera discovery, interactive session testing, and batch log
processing.

| Skill                         | Description                                                          |
|-------------------------------|----------------------------------------------------------------------|
| `camera-interface`            | Guides creation and configuration of VideoSystem instances           |
| `camera-setup`                | Discovers cameras and configures GenICam nodes via MCP tools         |
| `log-input-format`            | Documents the NPZ log archive input format and source ID semantics   |
| `log-processing`              | Orchestrates batch frame-timestamp log processing via the MCP server |
| `log-processing-results`      | Interprets frame timing data, frame drops, and acquisition quality   |
| `pipeline`                    | Orchestrates the end-to-end recording and analysis pipeline          |
| `post-recording`              | Verifies recording outputs and prepares them for log processing      |
| `video-mcp-environment-setup` | Diagnoses and resolves MCP server connectivity issues                |

### microcontroller

Firmware development skills for
[ataraxis-micro-controller](https://github.com/Sun-Lab-NBB/ataraxis-micro-controller). This plugin
does not provide an MCP server.

| Skill             | Description                                                          |
|-------------------|----------------------------------------------------------------------|
| `firmware-module` | Guides creation of custom hardware Module subclasses in C++ firmware |

### Compatibility

These plugins are designed for [Claude Code](https://docs.anthropic.com/en/docs/claude-code), the official CLI tool from
Anthropic. The plugin marketplace system is a Claude Code feature and is not currently available in other Claude
distributions such as Claude Desktop or the Claude web interface. The bundled MCP servers can be registered with any
MCP-capable client, and the skill files themselves are plain Markdown that can be referenced manually in any context 
that supports custom instructions or system prompts.

___

## Example Workflows

### Pre-Session Hardware Validation

```
User: Is the system ready for today's imaging session?

[AI invokes MCP tools: list_microcontrollers, list_cameras, check_mqtt_broker]

AI: Pre-flight check complete:
    Microcontrollers:
      ✓ /dev/ttyACM0 -> Teensy 4.1 [ID: 101] (actor)
      ✓ /dev/ttyACM1 -> Teensy 4.1 [ID: 152] (sensor)
    Cameras:
      ✓ FXO546 (19322813) 2064x1544@120fps (face)
      ✓ FXO547 (19322814) 2064x1544@60fps (body)
    Communication:
      ✓ MQTT broker at localhost:1883 connected

    All hardware validated. Ready to start acquisition.
```

### Hardware Interface Code Generation

```
User: I need to add a piezo buzzer for audio cues

[AI invokes skill: /microcontroller-interface]

AI: Which Teensy should control it? The ACTOR handles outputs.

User: ACTOR controller, pin 5

[AI generates C++ firmware module and Python bindings]

AI: Created buzzer module with pulse, toggle on, and toggle off commands.
    Upload the firmware and update the dependency version.
```

___

## Adoption Roadmap

1. **Install core libraries** and Claude Code plugins
2. **Create lab-specific configuration schemas** using existing templates
3. **Encode recurring workflows as skills** that guide AI agents
4. **Iterate as hardware evolves** with AI-assisted development

The Ataraxis libraries provide open-source building blocks for assembling custom acquisition
systems. For a complete platform built on Ataraxis, see
[Sollertia](https://github.com/Sun-Lab-NBB/sollertia) — a platform for AI-assisted data acquisition
and management.

___

## Citation

If you use Ataraxis in your research, please cite:

```bibtex
@article{ataraxis2025,
  title={Ataraxis: Bridging AI Coding Assistants and Scientific Hardware},
  author={Kondratyev, Ivan and Sun, Weinan},
  journal={},
  year={2026},
  institution={Cornell University, Department of Neurobiology and Behavior}
}
```

___

## License

All Ataraxis libraries are released under the [Apache License 2.0](LICENSE).

___

## Acknowledgments

We thank Anthropic for developing Claude Code and the Model Context Protocol. This work was
supported by Cornell University.

For questions or feedback, contact: ik278@cornell.edu, ws467@cornell.edu
