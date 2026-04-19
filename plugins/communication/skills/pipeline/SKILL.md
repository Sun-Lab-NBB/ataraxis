---
name: pipeline
description: >-
  End-to-end orchestration guide for the ataraxis-communication-interface data acquisition and analysis
  pipeline. Covers canonical phase ordering with handoff conditions, multi-controller planning with
  DataLogger topology, and decision trees for hardware, configuration, and processing setup. Use when
  planning a full data collection workflow, setting up multi-controller systems, or deciding between
  MCP and code.
user-invocable: true
---

# Pipeline

End-to-end orchestration reference for microcontroller data acquisition and analysis. Covers single and
multi-controller setups, phase ordering, handoff conditions, and decision guidance.

---

## Scope

**Covers:**
- Canonical pipeline phase ordering with handoff conditions
- Decision trees for hardware interface and processing configuration
- Multi-controller planning: controller ID allocation, DataLogger topology, coordinated lifecycle
- Multi-controller log processing
- MCP vs code decision guidance
- Quick-start references for common scenarios

**Does not cover:**
- Detailed tool usage for any individual phase (see phase-specific skills)
- MCP server connectivity (see `/communication-mcp-environment-setup`)

**Handoff rules:** This skill dispatches to phase-specific skills at each stage. Always invoke the relevant
skill for detailed tool usage, parameter reference, and troubleshooting.

---

## Pipeline phases

```text
Environment    Hardware       Extraction     Recording     Log            Results
Setup       →  Discovery   →  Config      →             →  Processing  →  Analysis
    |              |              |              |              |              |
/mcp-env-    /mc-setup     /extraction-  /mc-interface  /log-          /log-processing
 setup                      configuration               processing     -results
```

### Phase 1: Environment setup

- **Skill:** `/communication-mcp-environment-setup`
- **Actions:** Verify MCP server connectivity, check `axci` command availability, verify Python version
- **Handoff condition:** MCP tools accessible
- **Skip condition:** MCP already verified in this session

### Phase 2: Hardware discovery

- **Skill:** `/microcontroller-setup`
- **Actions:** `list_microcontrollers`, `check_mqtt_broker`, inspect existing manifests
- **Handoff condition:** Microcontrollers identified, MQTT verified (if needed), device paths recorded
- **Decision point:** Single controller vs multi-controller (see multi-controller planning below)

### Phase 3: Extraction configuration

- **Skill:** `/extraction-configuration`
- **Actions:** Read manifest, generate precursor config, ask user for event codes, write and validate config
- **Handoff condition:** Validated extraction config YAML file exists
- **Note:** This phase can be done before or after recording. For repeat experiments with the same hardware,
  reuse an existing config.

### Phase 4: Recording session

- **Skill:** `/microcontroller-interface`
- **Actions:** Write MicroControllerInterface/ModuleInterface code, run recording session
- **Handoff condition:** Session stopped, DataLogger flushed, archives assembled
- **Important:** Recording ALWAYS requires Python code. There is no MCP recording session for AXCI.

### Phase 5: Log processing

- **Skill:** `/log-processing`
- **Actions:** Discover archives, prepare batch with config path, execute jobs, monitor progress
- **Handoff condition:** All jobs SUCCEEDED in ProcessingTracker

### Phase 6: Results analysis

- **Skill:** `/log-processing-results`
- **Actions:** Verify output, analyze event distributions, interpret timing statistics

---

## Decision trees

### Hardware interface

```text
Is the microcontroller connected via USB?
  YES → USB serial (baudrate setting ignored by USB devices)
  NO  → UART serial → Must specify correct baudrate for identification and communication
```

### MCP vs code

| Scenario                                          | Recommendation                             |
|---------------------------------------------------|--------------------------------------------|
| Discover connected microcontrollers               | MCP via `/microcontroller-setup`           |
| Verify MQTT broker connectivity                   | MCP via `/microcontroller-setup`           |
| Create or inspect manifests                       | MCP via `/microcontroller-setup`           |
| Assemble log archives                             | MCP via `/microcontroller-setup`           |
| Create and validate extraction config             | MCP via `/extraction-configuration`        |
| Run a recording session                           | Code via `/microcontroller-interface`      |
| Process log archives                              | MCP via `/log-processing`                  |
| Analyze processing results                        | MCP via `/log-processing-results`          |

AXCI does not provide MCP tools for running recording sessions. All data acquisition requires writing
Python code that creates MicroControllerInterface and ModuleInterface instances.

---

## Multi-controller planning

### Controller ID allocation

| Range   | Assignment                         | Notes                                             |
|---------|------------------------------------|---------------------------------------------------|
| 101-150 | MicroControllerInterface instances | Advised production range; not enforced            |
| 1-255   | Valid range                        | Any np.uint8 value; must be unique per DataLogger |

Allocate controller IDs sequentially starting at 101 (e.g., 101, 102, 103 for a 3-controller setup).
Source IDs must be unique across **all** sources sharing a DataLogger, including sources from other
libraries (e.g., ataraxis-video-system). The 101-150 range avoids collisions with other libraries'
advised ranges.

### DataLogger topology

A single shared DataLogger is the preferred topology for all use cases:

```text
DataLogger(instance_name="session")
  ├── MicroControllerInterface(controller_id=101, name="teensy_main")  → 101_log.npz
  └── MicroControllerInterface(controller_id=102, name="teensy_aux")   → 102_log.npz
```

All controllers share one log directory, all timestamps are correlated, one `assemble_log_archives` call
consolidates everything, and one processing batch covers all source IDs. Each MicroControllerInterface
writes an entry to `microcontroller_manifest.yaml` during initialization.

Multiple DataLoggers should only be used if a single logger cannot handle the message throughput. When
it does occur, each DataLogger creates a separate output directory that must be assembled and processed
independently.

### Coordinated lifecycle

The ordering of initialization and shutdown is critical for multi-controller setups:

```text
Startup (in order):
  1. DataLogger → __init__() → start()
  2. MicroControllerInterface(s) → __init__() → start()

Shutdown (reverse order):
  3. MicroControllerInterface(s) → stop()
  4. DataLogger → stop()
  5. assemble_log_archives() for each DataLogger output directory
```

- DataLogger must be started BEFORE any MicroControllerInterface that references it
- MicroControllerInterface must be stopped BEFORE its DataLogger
- Assembly must run AFTER `DataLogger.stop()`

### Multi-controller code structure

```python
from pathlib import Path

import numpy as np
from ataraxis_data_structures import DataLogger, assemble_log_archives

from ataraxis_communication_interface import MicroControllerInterface

session_directory = Path("/path/to/session")

# Starts the shared DataLogger first.
logger = DataLogger(output_directory=session_directory, instance_name="session")
logger.start()

# Initializes and starts each controller with a unique ID and descriptive name.
controllers: list[MicroControllerInterface] = []
controller_configs = [
    (101, "/dev/ttyACM0", "teensy_main", main_modules),
    (102, "/dev/ttyACM1", "teensy_aux", aux_modules),
]
for ctrl_id, port, name, modules in controller_configs:
    controller = MicroControllerInterface(
        controller_id=np.uint8(ctrl_id),
        data_logger=logger,
        module_interfaces=modules,
        buffer_size=512,
        port=port,
        name=name,
    )
    controller.start()
    controllers.append(controller)

# ... recording ...

# Shuts down in reverse order.
for controller in controllers:
    controller.stop()
logger.stop()

# Assembles archives after the DataLogger has fully stopped.
assemble_log_archives(log_directory=logger.output_directory, remove_sources=True)
```

---

## Multi-controller log processing

All controllers sharing a DataLogger write to the same log directory and the same
`microcontroller_manifest.yaml`. This simplifies batch processing:

1. `discover_microcontroller_data_tool` finds the manifest and identifies all confirmed sources
   with their controller names and module listings
2. Create and validate extraction config covering all controllers (see `/extraction-configuration`)
3. `prepare_log_processing_batch_tool` creates one job per source ID with the config path
4. Process all source IDs in a single batch for efficiency
5. Output: multiple feather files per controller under `microcontroller_data/` subdirectory

For multi-DataLogger setups, process each DataLogger output directory as a separate batch.

---

## Quick-start scenarios

### Single microcontroller, first session

1. `/communication-mcp-environment-setup` — verify MCP connectivity (if first session)
2. `/microcontroller-setup` — `list_microcontrollers` → record device path and controller ID
3. `/microcontroller-interface` — write MicroControllerInterface code
4. Run the recording session
5. `/microcontroller-setup` — `assemble_log_archives_tool` if needed
6. `/extraction-configuration` — create and validate extraction config
7. `/log-processing` — extract microcontroller data
8. `/log-processing-results` — analyze event distributions and timing

### Multi-controller, behavioral experiment

1. `/microcontroller-setup` — discover all controllers
2. `/pipeline` — plan controller IDs and DataLogger topology
3. `/microcontroller-interface` — write multi-controller code following coordinated lifecycle
4. Run the recording session
5. `/extraction-configuration` — create config covering all controllers
6. `/log-processing` — batch process all source IDs together
7. `/log-processing-results` — analyze results per controller

### Existing data, new extraction

1. `/microcontroller-setup` — `discover_microcontroller_data_tool` to locate existing archives
2. `/extraction-configuration` — create new extraction config with different event codes
3. `/log-processing` — `clean_log_processing_output_tool` → re-process with new config
4. `/log-processing-results` — analyze new extraction

---

## Related skills

| Skill                        | Role                                                          |
|------------------------------|---------------------------------------------------------------|
| `/communication-mcp-environment-setup`     | Phase 1: environment verification                             |
| `/microcontroller-setup`     | Phase 2: hardware discovery and manifest management           |
| `/extraction-configuration`  | Phase 3: extraction config creation and validation            |
| `/microcontroller-interface` | Phase 4: MicroControllerInterface code for recording          |
| `/log-input-format`          | Reference: archive format for troubleshooting                 |
| `/log-processing`            | Phase 5: data extraction                                      |
| `/log-processing-results`    | Phase 6: output verification and event analysis               |

---

## Verification checklist

```text
Pipeline Orchestration:
- [ ] Environment verified (MCP server connected)
- [ ] Microcontroller(s) discovered and device paths recorded
- [ ] Controller IDs allocated (unique per controller, 101-150 advised range)
- [ ] DataLogger topology decided (single vs multiple)
- [ ] Extraction configuration created and validated
- [ ] Recording session completed (all controllers started and stopped in order)
- [ ] Log archives assembled (assemble_log_archives)
- [ ] Log processing completed (all source IDs processed)
- [ ] Event analysis performed for all controllers
```
