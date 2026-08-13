---
name: pipeline
description: >-
  Orchestrates the ataraxis-communication-interface data acquisition and analysis pipeline end to end.
  Covers canonical phase ordering with handoff conditions, multi-controller planning with DataLogger
  topology, and decision trees for hardware, configuration, and processing setup. Use when planning a
  full data collection workflow, setting up multi-controller systems, or deciding between MCP and code.
user-invocable: false
---

# Pipeline

Orchestrates microcontroller data acquisition and analysis end to end, covering single and multi-controller setups,
phase ordering, handoff conditions, and decision guidance.

---

## Scope

**Covers:**
- Canonical pipeline phase ordering with handoff conditions
- Decision trees for hardware interface, keepalive interval, and MCP versus code
- Multi-controller planning: controller ID allocation, DataLogger topology, coordinated lifecycle
- Multi-controller log processing
- Quick-start references for common scenarios

**Does not cover:**
- Detailed tool usage for any individual phase (see phase-specific skills)
- MCP server connectivity (see `/communication-mcp-environment-setup`)
- Core and memory budget planning for a processing batch (see `/log-processing`, "Resource management")

---

## Pipeline phases

```text
Phase 0  Firmware flash      →  /microcontroller:firmware-module
Phase 1  Environment setup   →  /communication-mcp-environment-setup
Phase 2  Hardware discovery  →  /microcontroller-setup
Phase 3  Extraction config   →  /extraction-configuration
Phase 4  Recording session   →  /microcontroller-interface
Phase 5  Log processing      →  /log-processing
Phase 6  Results analysis    →  /log-processing-results
```

### Phase 0: Firmware

- **Skill:** `/microcontroller:firmware-module`
- **Actions:** Author or confirm the C++ firmware, set the controller ID and keepalive interval in `main.cpp`, flash the
  board
- **Handoff condition:** The board answers the identification query with its controller ID
- **Skip condition:** The board was flashed earlier and a previous discovery run already reported its ID
- **Note:** Firmware is a hard prerequisite for phase 2, not an optional preliminary. `list_microcontrollers_tool`
  prints `[No microcontroller]` for a port that it opened successfully but that returned no identification response.
  That is what an unflashed board, a board running non-ataraxis firmware, and a UART board queried at the wrong
  baudrate all look like. Treat that string as "check phase 0 and the baudrate", never as "no device attached".

### Phase 1: Environment setup

- **Skill:** `/communication-mcp-environment-setup`
- **Actions:** Verify MCP server connectivity, check `axci` command availability, verify Python version
- **Handoff condition:** MCP tools accessible
- **Skip condition:** MCP already verified in this session

### Phase 2: Hardware discovery

- **Skill:** `/microcontroller-setup`
- **Actions:** `list_microcontrollers_tool`, `check_mqtt_broker_tool`, inspect existing manifests
- **Handoff condition:** Microcontrollers identified, MQTT verified (if needed), device paths recorded
- **Decision point:** Single controller vs multi-controller (see multi-controller planning below)
- **Blocked by:** Phase 0. Return to `/microcontroller:firmware-module` for any port the discovery run reports as
  `[No microcontroller]`.

### Phase 3: Extraction configuration

- **Skill:** `/extraction-configuration`
- **Actions:** Read manifest, generate precursor config, ask user for event codes, write and validate config
- **Handoff condition:** Validated extraction config YAML file exists
- **Note:** This phase can be done before or after the recording *run*, but generating a precursor config from the
  manifest requires the MicroControllerInterface instances to have been constructed first (construction writes
  `microcontroller_manifest.yaml`). For a brand-new hardware setup, instantiate the interfaces first, or author the
  config by hand. For repeat experiments with the same hardware, reuse an existing config.

### Phase 4: Recording session

- **Skill:** `/microcontroller-interface`
- **Actions:** Write MicroControllerInterface/ModuleInterface code, choose `keepalive_interval`, run the session
- **Handoff condition:** The recording loop exited and the code is ready to shut the stack down
- **Important:** Recording ALWAYS requires Python code. There is no MCP recording session for `axci`.
- **Decision point:** Keepalive interval (see the keepalive interval decision below)

### Phase 5: Log processing

- **Skill:** `/log-processing`
- **Actions:** Assemble archives if the runtime left raw `.npy` entries, discover archives, prepare batch with config
  path, execute jobs, monitor progress
- **Handoff condition:** All jobs SUCCEEDED in ProcessingTracker
- **Precondition:** The recording's raw entries must already be assembled into per-source `.npz` archives.
  `/microcontroller-setup` owns that step. Skipping it leaves this phase with nothing to prepare, and it fails silently
  rather than loudly, since an unassembled directory simply discovers no source.
- **Resource planning:** Do not size jobs or budgets from this skill. `/log-processing`, "Resource management", names
  the assets that report per-job and per-session figures, and it is the only place those figures are authoritative.

### Phase 6: Results analysis

- **Skill:** `/log-processing-results`
- **Actions:** Verify output, analyze event distributions, interpret timing statistics
- **Caveat:** `/log-processing-results`, "Message loss is not measurable post-hoc", owns the rule governing how an
  inter-event gap may be reported.

---

## Decision trees

### Hardware interface

```text
Is the microcontroller connected via USB?
  YES → USB serial (baudrate setting ignored by USB devices)
  NO  → UART serial → Must specify correct baudrate for identification and communication
```

### Keepalive interval

`keepalive_interval` is a `MicroControllerInterface` constructor argument in milliseconds, and `0` disables the
mechanism, leaving no watchdog reset when the link drops. Pick the value during phase 4 planning, because the workable
band depends on the board and link speed fixed in phases 0 and 2, not on anything this phase controls. Read the
starting bands from `/microcontroller:firmware-module`, "Kernel constructor". They follow the link speed and CPU
frequency rather than the board name, so do not plan against a restatement of them here.

Both ends must be configured. The PC sends the keepalive command at this interval, and the firmware runs its own
watchdog off the interval its Kernel was built with. Setting the PC interval while the firmware has keepalive disabled
gives no protection, and setting it above the window the firmware derives from its own interval trips the watchdog
during normal operation. Keep the PC interval at or below the firmware's. See `/microcontroller:firmware-module` for the
firmware-side argument and the derived window, and `/microcontroller-interface` for the PC-side failure modes and the
timeout status event.

### MCP vs code

| Scenario                                | Recommendation                              |
|-----------------------------------------|---------------------------------------------|
| Write or flash microcontroller firmware | Code via `/microcontroller:firmware-module` |
| Discover connected microcontrollers     | MCP via `/microcontroller-setup`            |
| Verify MQTT broker connectivity         | MCP via `/microcontroller-setup`            |
| Create or inspect manifests             | MCP via `/microcontroller-setup`            |
| Assemble log archives                   | MCP via `/microcontroller-setup`            |
| Create and validate extraction config   | MCP via `/extraction-configuration`         |
| Run a recording session                 | Code via `/microcontroller-interface`       |
| Shut a runtime down                     | Code via `/microcontroller-interface`       |
| Process log archives                    | MCP via `/log-processing`                   |
| Analyze processing results              | MCP via `/log-processing-results`           |

The firmware and recording rows are code-only for structural reasons, not by preference. Firmware is C++, built and
flashed outside `axci` entirely, so no MCP tool can reach it. `axci` provides no MCP tool for running a recording
session, so all data acquisition requires Python code that creates MicroControllerInterface and ModuleInterface
instances.

Orchestration happens through the MCP tools or through the `axci` CLI a user runs by hand, and the library's
orchestration Python symbols are not an agent-facing surface even though they are exported. See `/cli-reference` for the
command-line side.

---

## Multi-controller planning

### Controller ID allocation

A controller's `controller_id` IS its source ID at the DataLogger level: it is the value MicroControllerInterface
registers as the `source_id`, and it names the controller's `{controller_id}_log.npz` archive (see `/log-input-format`).
This skill uses "source ID" for the shared DataLogger namespace and `controller_id` for the MicroControllerInterface
constructor.

| Range   | Assignment                         | Notes                                             |
|---------|------------------------------------|---------------------------------------------------|
| 101-150 | MicroControllerInterface instances | Advised production range, not enforced            |
| 1-255   | Valid range                        | Any np.uint8 value, must be unique per DataLogger |

Allocate controller IDs sequentially starting at 101 (e.g., 101, 102, 103 for a 3-controller setup). Source IDs must be
unique across **all** sources sharing a DataLogger, including sources from other libraries (e.g.,
ataraxis-video-system). The 101-150 range avoids collisions with other libraries' advised ranges.

When cameras record into the same DataLogger, invoke `/video:pipeline` for the camera side. It advises the 51-100 band
for VideoSystem instances, and it owns the rules for a directory holding both libraries' manifests and archives. One
value overlaps: the axvs interactive CLI is fixed at 111, which falls inside the 101-150 range above, so keep 111 free
on any logger a user may run `axvs run` against.

### DataLogger topology

A single shared DataLogger is the preferred topology for all use cases:

```text
DataLogger(instance_name="session")
  ├── MicroControllerInterface(controller_id=101, name="teensy_main")  → 101_log.npz
  └── MicroControllerInterface(controller_id=102, name="teensy_aux")   → 102_log.npz
```

All controllers share one log directory, all timestamps are correlated, one `assemble_log_archives` call consolidates
everything, and one processing batch covers all source IDs. Each MicroControllerInterface writes an entry to
`microcontroller_manifest.yaml` during initialization.

Use multiple DataLoggers only when a single logger cannot handle the message throughput.

### Coordinated lifecycle

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

`/microcontroller-interface` owns the code that implements this ordering, and `/microcontroller-setup` owns running the
assembly and the discovery that follows it, including the case where assembly must be run separately against an
already-stopped directory.

---

## Multi-controller log processing

All controllers sharing a DataLogger write to the same log directory and the same `microcontroller_manifest.yaml`.

1. `discover_microcontroller_data_tool` finds the manifest and names every confirmed source and controller name through
   its `breakdown`, with `include_items=True` listing the sources and `detailed=True` adding each source's modules
2. Create and validate extraction config covering all controllers (see `/extraction-configuration`)
3. `prepare_log_processing_batch_tool` creates one job per source ID with the config path
4. Process all source IDs in a single batch for efficiency
5. Output: multiple feather files per controller under `microcontroller_data/` subdirectory

To process every controller, list every controller's ID in the extraction config. `/log-processing` is authoritative for
the sourcing model that decides which requested IDs become jobs. Do not plan against a restatement of it here.

For multi-DataLogger setups, pass each DataLogger output directory to the preparation tool separately. Passing their
shared parent fails that directory outright rather than merely processing it inefficiently: a tree holding more than
one `microcontroller_manifest.yaml` spans several recordings or several DataLogger instances, and exactly one manifest
is supported per invocation. The preparation tool reports the failure under `failed_directories` and prepares the rest
of the batch, while the CLI and the library path raise `ValueError`. `discover_microcontroller_data_tool` does tolerate
the parent: run it there and pass its flat `log_directories` list to preparation, which is exactly the per-directory
split preparation needs.

---

## Quick-start scenarios

### Resuming an existing project root

Start here whenever the user points at a directory rather than at a task. `/log-processing` owns
`get_batch_status_overview_tool`, whose recursive `breakdown` of directories per status decides the route.

| The overview reports      | Route to                                                              |
|---------------------------|-----------------------------------------------------------------------|
| Nothing found             | `/microcontroller-setup` to assemble archives, then resume at phase 5 |
| Every directory succeeded | `/log-processing-results`                                             |
| Anything failed           | `/log-processing`, "Re-running failed jobs"                           |
| Anything still running    | `/log-processing` monitoring rather than a new batch                  |

### New hardware module, firmware and interface together

A hardware module is one design split across two plugins, and six values have to match on both sides. Work the two
skills in this order rather than finishing one side and transcribing it into the other.

1. `/microcontroller:firmware-module`: pick `module_type` and `module_id`, number the command enum from 1, assign custom
   event codes in the 51-250 range, lay out the `PACKED_STRUCT` parameter struct, and choose the C++ type and element
   count every `SendData()` call transmits
2. `/microcontroller-interface`: mirror all six in the `ModuleInterface` subclass. That means the same type and id, the
   same command codes, and `data_codes` and `error_codes` drawn from the same event codes. It also means a
   `send_parameters()` tuple matching the struct field for field, and `message.data_object` annotated against the
   payload prototype the firmware fixed
3. Confirm the payload prototype and the parameter struct both fit the target board before either side is written,
   because the reachable byte budgets fall below the Teensy figures on the Arduino Due and the Arduino Mega
4. Flash the board and discover it, which is phases 0 and 2 above
5. `/extraction-configuration`: declare the same custom event codes under that module's entry
6. `/log-processing-results`: decode the extracted `event` and `command` columns against the codes chosen in step 1

Any later change to a code, to the parameter struct, or to a payload prototype is a change to both sides. Re-run steps 1
and 2 together instead of editing one side alone.

### Existing data, new extraction

1. `/microcontroller-setup`: `discover_microcontroller_data_tool` to locate existing archives
2. `/extraction-configuration`: create new extraction config with different event codes
3. `/log-processing`: `clean_log_processing_output_tool` → re-process with new config
4. `/log-processing-results`: analyze new extraction

### Extraction on a machine without the MCP server

1. `/communication-mcp-environment-setup`: attempt to restore the server first
2. `/cli-reference`: if it cannot be restored, print the `axci config create` and `axci process` commands for the user
   to run by hand, and ask them to paste the output back
3. `/log-processing-results`: analyze the output once the user reports the run finished

---

## Related skills

| Skill                                  | Relationship                                             |
|----------------------------------------|----------------------------------------------------------|
| `/microcontroller:firmware-module`     | Phase 0: firmware the board must run to be discoverable  |
| `/communication-mcp-environment-setup` | Phase 1: environment verification                        |
| `/microcontroller-setup`               | Phase 2: hardware discovery and manifest management      |
| `/extraction-configuration`            | Phase 3: extraction config creation and validation       |
| `/microcontroller-interface`           | Phase 4: MicroControllerInterface code for recording     |
| `/log-input-format`                    | Reference: archive format for troubleshooting            |
| `/log-processing`                      | Phase 5: data extraction. Owns batch resource management |
| `/log-processing-results`              | Phase 6: output verification and event analysis          |
| `/cli-reference`                       | Reference: the full `axci` command surface, human-facing |
| `/video:pipeline`                      | Peer: the camera side of a shared-DataLogger recording   |

---

## Verification checklist

```text
Pipeline Orchestration, tool-settled (run `list_microcontrollers_tool`, `validate_extraction_config_tool`,
`discover_microcontroller_data_tool`, `get_batch_status_overview_tool`, `query_extracted_events_tool`):
- [ ] Firmware flashed and each board reports its controller ID (no [No microcontroller] ports in scope)
- [ ] Environment verified (MCP server connected)
- [ ] Microcontroller(s) discovered and device paths recorded
- [ ] Extraction configuration created and validated
- [ ] Post-runtime completed (archives assembled, one archive per expected source)
- [ ] Log processing completed (all source IDs processed, skipped_sources read and explained)
- [ ] Event analysis performed for all controllers

Pipeline Orchestration, reader-judged:
- [ ] Controller IDs allocated (unique per controller, 101-150 advised range)
- [ ] DataLogger topology decided (single vs multiple)
- [ ] Keepalive interval chosen for the board and link, or deliberately left at 0
- [ ] Recording session completed (all controllers started and stopped in order)
- [ ] Batch scoped per DataLogger output directory, never at their shared parent
```
