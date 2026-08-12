---
name: microcontroller-setup
description: >-
  Guides use of ataraxis-communication-interface MCP tools for microcontroller discovery, MQTT broker
  verification, manifest management, log archive assembly, and recording discovery. Use when discovering
  connected microcontrollers, testing MQTT connectivity, managing manifests, or assembling log archives.
user-invocable: false
---

# Microcontroller setup

Guides the use of the ataraxis-communication-interface MCP tools for hardware discovery, MQTT verification, manifest
management, and log archive assembly. This skill covers all MCP tool interactions, for writing code that integrates
MicroControllerInterface into an acquisition system, use `/microcontroller-interface` instead.

---

## Scope

**Covers:**
- Discovering connected microcontrollers via serial ports
- Testing MQTT broker connectivity
- Reading, writing, and inspecting microcontroller manifests
- Assembling raw log entries into archives after a recording
- Discovering microcontroller recordings across directory trees

**Does not cover:**
- Writing MicroControllerInterface or ModuleInterface code (see `/microcontroller-interface`)
- Extraction configuration management (see `/extraction-configuration`)
- Log processing workflow (see `/log-processing`)
- MCP server connectivity issues (see `/communication-mcp-environment-setup`)

---

## MCP tool reference

The ataraxis-communication-interface MCP server exposes 19 tools. This skill covers the 6 tools most relevant to
hardware setup and data management. Log processing and analysis tools are documented in `/log-processing` and
`/log-processing-results`. Configuration tools are documented in `/extraction-configuration`.

### Hardware discovery

| Tool                         | Purpose                                                          |
|------------------------------|------------------------------------------------------------------|
| `list_microcontrollers_tool` | Discovers serial ports and identifies connected microcontrollers |
| `check_mqtt_broker_tool`     | Tests MQTT broker reachability at a specified host and port      |

**`list_microcontrollers_tool` parameters:**

| Parameter  | Type  | Default  | Description                                                     |
|------------|-------|----------|-----------------------------------------------------------------|
| `baudrate` | `int` | `115200` | Baudrate for identification (UART only, ignored by USB devices) |

For a UART connection the identification baudrate must match the speed the firmware was built with, and 115200 is not a
universal default across boards. Confirm the flashed board's configured speed with the user before concluding a port
holds no microcontroller. `/microcontroller-interface` carries the per-board values and the firmware build files that
fix them.

Output format:
```text
Evaluated 3 serial port(s) at baudrate 115200:
1: /dev/ttyACM0 -> Teensy 4.1 [Microcontroller ID: 101]
2: /dev/ttyACM1 -> Arduino Mega [No microcontroller]
3: /dev/ttyUSB0 -> USB-SERIAL CH340 [Connection Failed: SerialException: could not open port /dev/ttyUSB0]
```

Each line shows the device path, description, and one of three statuses:
- **Microcontroller ID: N**: Identified as running ataraxis-micro-controller with the given controller ID
- **No microcontroller**: Port responds but is not running ataraxis-micro-controller firmware
- **Connection Failed**: Could not establish communication (timeout, permission error, etc.)

Each line is rendered from `evaluate_port(port, baudrate)`, which the `ataraxis_communication_interface.microcontroller`
sub-package exports. The three statuses are its three return shapes, so import it when code must branch on the outcome
instead of parsing this string:

| `evaluate_port` return  | Rendered status         | Meaning                                                    |
|-------------------------|-------------------------|------------------------------------------------------------|
| `(controller_id, None)` | `Microcontroller ID: N` | The controller answered the identification command         |
| `(-1, None)`            | `No microcontroller`    | The port opened but nothing answered within the ID timeout |
| `(-1, "Type: message")` | `Connection Failed`     | Opening or querying the port raised. `Type` is the class   |

The function never propagates an exception, so one unreachable port never aborts the sweep over the others.

Use `check_mqtt_broker_tool` to verify the broker is reachable before writing code that depends on MQTT connectivity.
`/microcontroller-interface` owns `MQTTCommunication`, the class that carries that connectivity into the runtime.

**`check_mqtt_broker_tool` parameters:**

| Parameter | Type  | Default       | Description                               |
|-----------|-------|---------------|-------------------------------------------|
| `host`    | `str` | `"127.0.0.1"` | IP address or hostname of the MQTT broker |
| `port`    | `int` | `1883`        | Socket port used by the MQTT broker       |

### Manifest management

| Tool                                  | Purpose                                            |
|---------------------------------------|----------------------------------------------------|
| `read_microcontroller_manifest_tool`  | Reads manifest YAML and returns controller entries |
| `write_microcontroller_manifest_tool` | Registers a controller source in the manifest      |
| `discover_microcontroller_data_tool`  | Recursively discovers recordings with manifests    |

**`read_microcontroller_manifest_tool` parameters:**

| Parameter       | Type  | Default    | Description                                             |
|-----------------|-------|------------|---------------------------------------------------------|
| `manifest_path` | `str` | (required) | Absolute path to the microcontroller_manifest.yaml file |

**Return structure:**
```text
manifest_path:      Path to the manifest file
controllers[]:      List of registered controller entries:
  id:               Controller ID (integer)
  name:             Human-readable controller name
  modules[]:        List of hardware module entries:
    module_type:    Module type code (integer)
    module_id:      Module instance ID (integer)
    name:           Human-readable module name
total_controllers:  Number of registered controllers
```

**Note:** this tool reports `id` as an integer, while `discover_microcontroller_data_tool` reports the same value as the
string `source_id`. Every batch tool takes `source_ids` as strings, so wrap a manifest `id` in `str()` before passing it
on. `/log-input-format` carries the full source ID type rule.

**`write_microcontroller_manifest_tool` parameters:**

| Parameter         | Type         | Default    | Description                                                           |
|-------------------|--------------|------------|-----------------------------------------------------------------------|
| `log_directory`   | `str`        | (required) | Absolute path to the DataLogger output directory                      |
| `controller_id`   | `int`        | (required) | Controller ID to register                                             |
| `controller_name` | `str`        | (required) | Human-readable name for the controller                                |
| `modules`         | `list[dict]` | (required) | Module descriptors: each must have `module_type`, `module_id`, `name` |

**Important:** You MUST know the controller ID, name, and module details. Do not guess these values. Each module
dictionary must have keys: `module_type` (int), `module_id` (int), `name` (str).

Creates a new manifest if none exists. Otherwise replaces the entry already registered under the same `controller_id`,
or appends a new entry when the manifest carries none.

Re-registering the same `controller_id` overwrites that controller's entry rather than duplicating it, so a corrected
call replaces a wrong entry in place. `discover_microcontroller_data_tool` also collapses repeated controller IDs, so a
legacy manifest that still holds several rows for one ID reports a single source.

**`discover_microcontroller_data_tool` parameters:**

| Parameter        | Type  | Default    | Description                                   |
|------------------|-------|------------|-----------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to the root directory to search |

**Return structure:**
```text
sources[]:              Flat list of confirmed source entries:
  recording_root:       Path to the recording root directory
  source_id:            Source ID string (controller ID)
  name:                 Controller name from manifest
  log_archive:          Absolute path to the .npz archive
  log_directory:        Absolute path to the DataLogger output directory
  modules[]:            Module entries from manifest (module_type, module_id, name)
log_directories:        Flat list of log directory paths (pass directly to batch tools)
total_sources:          Number of confirmed source entries
total_log_directories:  Number of log directories with archives
```

**Important:** This tool requires `microcontroller_manifest.yaml` files in DataLogger output directories. These
manifests are written automatically by `MicroControllerInterface.__init__()`. For legacy sessions without manifests, use
`write_microcontroller_manifest_tool` to retroactively tag log directories before running discovery. It also returns
only those sources whose log archive (`{source_id}_log.npz`) already exists in that directory, a manifest entry with no
archive beside it is skipped, so assemble archives with `assemble_log_archives_tool` before running discovery on a
legacy directory.

### Archive assembly

| Tool                         | Purpose                                                           |
|------------------------------|-------------------------------------------------------------------|
| `assemble_log_archives_tool` | Consolidates raw .npy log entries into .npz archives by source ID |

**Parameters:**

| Parameter          | Type   | Default    | Description                                                 |
|--------------------|--------|------------|-------------------------------------------------------------|
| `log_directory`    | `str`  | (required) | Absolute path to DataLogger output directory. **Ask user.** |
| `remove_sources`   | `bool` | `true`     | Delete original .npy files after assembly                   |
| `verify_integrity` | `bool` | `false`    | Verify archive integrity before removing sources            |

The defaults permanently delete the raw .npy files (`remove_sources=true`) without verifying the archive first
(`verify_integrity=false`). For irreplaceable or legacy recordings, pass `verify_integrity=true` (and optionally
`remove_sources=false`) to keep the raw entries until you have confirmed the archives are valid.

**Return structure:**
```text
status:         "assembled"
directory:      Path to the log directory
archives:       List of archive filenames present in the directory after assembly (e.g., ["101_log.npz"])
source_ids:     List of extracted source ID strings
archive_count:  Number of archives present after assembly
```

The tool returns `{"error": "<message>"}` in place of that dictionary on three conditions, so check for an `error` key
before reading `status`:

| Condition                          | Message                            |
|------------------------------------|------------------------------------|
| `log_directory` does not exist     | `Directory not found: <path>`      |
| `log_directory` is not a directory | `Not a directory: <path>`          |
| The assembly call itself raised    | `Archive assembly failed: <error>` |

**Note:** `archives`, `source_ids`, and `archive_count` are recomputed by scanning the directory AFTER assembly runs, so
they report what is present rather than what this call produced. A directory that already holds .npz archives and no
.npy entries returns `status: "assembled"` with a full archive list and no indication that nothing was assembled. To
confirm a real change, list the .npy entries in the directory before calling the tool.

---

## Workflows

### Microcontroller discovery

1. Call `list_microcontrollers_tool` (adjust `baudrate` if using non-default UART settings)
2. Record the device paths and controller IDs for identified microcontrollers
3. If no microcontrollers appear:
   - Check physical USB connections
   - Verify the microcontroller is running ataraxis-micro-controller firmware
   - Check for serial port permissions (`/dev/ttyACM*` may require user group membership)
   - Try a different baudrate if using UART communication

### MQTT broker verification

1. Call `check_mqtt_broker_tool` with the broker's host and port
2. If unreachable:
   - Verify the MQTT broker service is running (e.g., `systemctl status mosquitto`)
   - Check firewall rules allow connections on the specified port
   - Verify the host address is correct
3. The "not reachable" message covers every socket-level failure, including a refused connection, a timeout, and a host
   name that could not be resolved. Verify the host string as well as the broker service before concluding the broker is
   down.

### Manifest inspection and retroactive tagging

**Inspect an existing manifest:**
1. Call `read_microcontroller_manifest_tool` with the manifest path
2. Review the controller and module entries

**Retroactively tag a legacy session:**
1. Call `write_microcontroller_manifest_tool` with the log directory, controller ID, controller name, and module list
   (re-registering the same `controller_id` replaces that entry rather than duplicating it)
2. Verify by calling `read_microcontroller_manifest_tool` on the created manifest
3. Run `discover_microcontroller_data_tool` to confirm the session is now discoverable

**Warning:** the manifest bounds the set of jobs a recording can hold, and `prepare_log_processing_batch_tool` DELETES
every tracker entry that falls outside it. Editing or regenerating a manifest so it stops registering a controller, then
re-preparing that directory, silently erases the completed-job history of the dropped controllers. Correct entries in
place with `write_microcontroller_manifest_tool`, which replaces the entry under the same `controller_id`. Never
hand-edit or rewrite a manifest to drop a controller whose jobs already ran.

### Post-session archive assembly

Assembly is a step of its own that runs once per recording, not something a shutdown call performs. axci exposes no tool
that starts or stops a runtime, and neither `MicroControllerInterface.stop()` nor `DataLogger.stop()` assembles
anything. One log directory holds every source the recording wrote, so a system recording alongside a microcontroller
writes into the same directory and is assembled by the same pass.

1. Confirm with the user that the runtime script stopped every interface and then the DataLogger
2. Check the log directory for `.npy` entries, and for `.npz` archives (see the hazard below)
3. Call `assemble_log_archives_tool` with the directory
4. Call `discover_microcontroller_data_tool` to prove the archives exist and every expected source is present
5. Proceed to `/extraction-configuration` for config setup, or `/log-processing` if a config exists

**A directory holding both `.npy` entries and `.npz` archives is a half-assembled recording, and assembling it again is
unsafe.** Archiving overwrites an existing archive of the same source, and neither `assemble_log_archives_tool` nor the
library function beneath it guards against this, so the destructive call succeeds silently and reports `status:
"assembled"`. Check for both extensions first. If both are present, have the user back up the existing archives and
remove them from the log directory before retrying.

Nothing downstream errors when this transition is skipped. `discover_microcontroller_data_tool` returns no source for an
unassembled directory, and a directory reached without discovery has each of its sources recorded under prepare's
`skipped_sources` while the call still reports `success: True`. A skipped assembly reads as an empty recording, never as
a failure.

---

## Bridge to code integration

`/microcontroller-interface` owns the mapping from a discovered device path, controller ID, and baudrate to the
`MicroControllerInterface` constructor arguments, and it owns controller ID allocation.

---

## Troubleshooting

| Symptom                                                | Likely Cause                          | Resolution                                                                                   |
|--------------------------------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| `list_microcontrollers_tool` → "No valid serial ports" | No USB devices connected              | Check physical connections and USB cables                                                    |
| Port shows "No microcontroller"                        | Firmware not loaded or wrong baudrate | Verify firmware, then retry at the board's own rate (see `/microcontroller:firmware-module`) |
| Port shows "Connection Failed"                         | Permission denied or port in use      | Check serial port permissions. Close conflicting programs                                    |
| MQTT broker unreachable                                | Broker not running                    | Start the broker service                                                                     |
| Assembly returns an `error` key                        | Missing path, or assembly raised      | Read the `error` message. See the assemble error table                                       |
| Assembly returns "assembled" but no new .npz           | Directory held no .npy entries        | Confirm the DataLogger wrote here, then stopped and flushed                                  |
| Discovery finds no sources                             | Missing manifest files                | Use `write_microcontroller_manifest_tool` to tag sessions                                    |
| Discovery errors with "Unable to search"               | Root or a subdirectory is unreadable  | Fix directory permissions or remount. Not a manifest gap                                     |
| MCP tools unavailable                                  | Server not running                    | Use `/communication-mcp-environment-setup` to diagnose                                       |

---

## CLI reference (human-facing, do not invoke)

> **CLI reference, for answering user questions only.** The `axci` command-line interface is a
> **human-facing** tool. **Agents must never invoke `axci` commands**, every agent-driven operation has an
> equivalent MCP tool (noted in the table). This section exists solely so the agent can answer user
> questions about the CLI.
>
> `/cli-reference` is the canonical reference for the whole `axci` command surface. The table below is the
> discovery subset. Invoke `/cli-reference` for anything it does not answer.

| Command     | Key options                                                     | Purpose                                             | MCP equivalent               |
|-------------|-----------------------------------------------------------------|-----------------------------------------------------|------------------------------|
| `axci id`   | `-b`/`--baudrate` (default 115200)                              | Discovers connected Arduino/Teensy microcontrollers | `list_microcontrollers_tool` |
| `axci mqtt` | `-h`/`--host` (default 127.0.0.1), `-p`/`--port` (default 1883) | Checks whether an MQTT broker is reachable          | `check_mqtt_broker_tool`     |

**Note:** `axci mqtt` binds `-h` to `--host`, and the CLI registers no `-h` help alias on any command, so `-h` consumes
the next token as a host instead of printing help. `--help` is the only help form. Tell a user who wants the option list
to run `axci mqtt --help`.

---

## Related skills

| Skill                                  | Relationship                                                                    |
|----------------------------------------|---------------------------------------------------------------------------------|
| `/microcontroller-interface`           | Covers writing MicroControllerInterface code after testing via MCP              |
| `/microcontroller:firmware-module`     | Firmware-side counterpart: C++ Module code that discovered controllers must run |
| `/extraction-configuration`            | Downstream: configure extraction parameters before processing                   |
| `/log-input-format`                    | Reference: documents the archive format produced by this workflow               |
| `/log-processing`                      | Downstream: processes archives assembled by this skill                          |
| `/log-processing-results`              | Downstream: analyzes output from processed archives                             |
| `/cli-reference`                       | Reference: the full `axci` command surface, human-facing                        |
| `/pipeline`                            | Context: end-to-end orchestration and multi-controller planning                 |
| `/communication-mcp-environment-setup` | Prerequisite: MCP server connectivity for all tool interactions                 |

---

## Verification checklist

```text
Microcontroller Setup:
- [ ] Discovered microcontrollers via list_microcontrollers_tool (recorded device paths and IDs)
- [ ] Verified MQTT broker connectivity if needed (check_mqtt_broker_tool)
- [ ] Confirmed microcontroller manifests exist in DataLogger output directories
- [ ] Assembled log archives if needed (assemble_log_archives_tool)
- [ ] Verified recordings discoverable via discover_microcontroller_data_tool
- [ ] Noted controller IDs for extraction configuration
```
