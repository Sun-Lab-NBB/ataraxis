# Ataraxis

**Bridging AI Coding Assistants and Scientific Hardware**

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

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

---

## Features

### Hardware Discovery & Validation
- **MCP-based device enumeration**: AI agents can query connected cameras, microcontrollers, and
  motor controllers through structured tool interfaces.
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

---

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
│    │  Discovery Tools    │ ─────┼────▶│ Ataraxis Libraries      │          │
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

---

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
Transport layer for Arduino and Teensy microcontrollers, enabling bidirectional serial
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

---

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

### MCP Server Configuration

Add Ataraxis MCP servers to your Claude Code configuration (`~/.claude.json`):

```json
{
  "mcpServers": {
    "ataraxis-communication-interface": {
      "command": "axci-mcp",
      "args": []
    },
    "ataraxis-video-system": {
      "command": "python",
      "args": ["-m", "ataraxis_video_system.mcp_server"]
    }
  }
}
```

---

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

---

## Adoption Roadmap

1. **Install core libraries** and configure MCP servers
2. **Create lab-specific configuration schemas** using existing templates
3. **Encode recurring workflows as skills** that guide AI agents
4. **Iterate as hardware evolves** with AI-assisted development

The Sun Lab's implementation libraries (`sl-*`) serve as open-source templates for building
custom acquisition systems.

---

## Citation

If you use Ataraxis in your research, please cite:

```bibtex
@article{ataraxis2025,
  title={Ataraxis: Bridging AI Coding Assistants and Scientific Hardware},
  author={Kondratyev, Ivan and Sun, Weinan},
  journal={},
  year={2025},
  institution={Cornell University, Department of Neurobiology and Behavior}
}
```

---

## License

All Ataraxis libraries are released under the [GNU General Public License v3.0](LICENSE).

---

## Acknowledgments

We thank Anthropic for developing Claude Code and the Model Context Protocol. This work was
supported by Cornell University.

For questions or feedback, contact: ik278@cornell.edu, ws467@cornell.edu
