---
name: microcontroller-setup
description: >-
  Guides use of ataraxis-communication-interface MCP tools for microcontroller discovery, MQTT broker
  verification, manifest management, log archive assembly, and recording discovery. Use when discovering
  connected microcontrollers, testing MQTT connectivity, managing manifests, or assembling log archives.
user-invocable: true
---

# Microcontroller setup

Guides the use of the ataraxis-communication-interface MCP tools for hardware discovery, MQTT verification,
manifest management, and log archive assembly. This skill covers all MCP tool interactions; for writing code
that integrates MicroControllerInterface into an acquisition system, use `/microcontroller-interface` instead.

---

## Scope

**Covers:**
- Discovering connected microcontrollers via serial ports
- Testing MQTT broker connectivity
- Reading, writing, and inspecting microcontroller manifests
- Assembling raw log entries into archives
- Discovering microcontroller recordings across directory trees

**Does not cover:**
- Writing MicroControllerInterface or ModuleInterface code (see `/microcontroller-interface`)
- Extraction configuration management (see `/extraction-configuration`)
- Log processing workflow (see `/log-processing`)
- MCP server connectivity issues (see `/mcp-environment-setup`)

---

## MCP tool reference

The ataraxis-communication-interface MCP server exposes 19 tools. This skill covers the 6 tools most
relevant to hardware setup and data management. Log processing and analysis tools are documented in
`/log-processing` and `/log-processing-results`. Configuration tools are documented in
`/extraction-configuration`.

### Hardware discovery

| Tool                     | Purpose                                                          |
|--------------------------|------------------------------------------------------------------|
| `list_microcontrollers`  | Discovers serial ports and identifies connected microcontrollers |
| `check_mqtt_broker`      | Tests MQTT broker reachability at a specified host and port      |

**`list_microcontrollers` parameters:**

| Parameter  | Type  | Default    | Description                                                     |
|------------|-------|------------|-----------------------------------------------------------------|
| `baudrate` | `int` | `115200`   | Baudrate for identification (UART only; ignored by USB devices) |

Output format:
```text
Evaluated 3 serial port(s) at baudrate 115200:
1: /dev/ttyACM0 -> Teensy 4.1 [Microcontroller ID: 101]
2: /dev/ttyACM1 -> Arduino Mega [No microcontroller]
3: /dev/ttyUSB0 -> USB-SERIAL CH340 [Connection Failed: timeout]
```

Each line shows the device path, description, and one of three statuses:
- **Microcontroller ID: N** — Identified as running ataraxis-micro-controller with the given controller ID
- **No microcontroller** — Port responds but is not running ataraxis-micro-controller firmware
- **Connection Failed** — Could not establish communication (timeout, permission error, etc.)

`MQTTCommunication` extends the serial microcontroller communication by connecting remote producers and
consumers to the microcontroller ecosystem over TCP. It is designed for tight integration with
`MicroControllerInterface` — for example, allowing a separate process or machine to send commands to or
receive data from microcontrollers via MQTT topics. It can be used standalone, but the library was
designed with this integrated usage in mind. Use `check_mqtt_broker` to verify the broker is reachable
before writing code that depends on MQTT connectivity.

**`check_mqtt_broker` parameters:**

| Parameter | Type  | Default       | Description                                       |
|-----------|-------|---------------|---------------------------------------------------|
| `host`    | `str` | `"127.0.0.1"` | IP address or hostname of the MQTT broker         |
| `port`    | `int` | `1883`        | Socket port used by the MQTT broker               |

Returns a message indicating whether the broker is reachable.

### Manifest management

| Tool                                  | Purpose                                            |
|---------------------------------------|----------------------------------------------------|
| `read_microcontroller_manifest_tool`  | Reads manifest YAML and returns controller entries |
| `write_microcontroller_manifest_tool` | Registers a controller source in the manifest      |
| `discover_microcontroller_data_tool`  | Recursively discovers recordings with manifests    |

**`read_microcontroller_manifest_tool` parameters:**

| Parameter       | Type  | Default    | Description                                              |
|-----------------|-------|------------|----------------------------------------------------------|
| `manifest_path` | `str` | (required) | Absolute path to the microcontroller_manifest.yaml file  |

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

**`write_microcontroller_manifest_tool` parameters:**

| Parameter         | Type         | Default    | Description                                                           |
|-------------------|--------------|------------|-----------------------------------------------------------------------|
| `log_directory`   | `str`        | (required) | Absolute path to the DataLogger output directory                      |
| `controller_id`   | `int`        | (required) | Controller ID to register                                             |
| `controller_name` | `str`        | (required) | Human-readable name for the controller                                |
| `modules`         | `list[dict]` | (required) | Module descriptors: each must have `module_type`, `module_id`, `name` |

**Important:** The agent MUST know the controller ID, name, and module details. Do not guess these values.
Each module dictionary must have keys: `module_type` (int), `module_id` (int), `name` (str).

Creates a new manifest if none exists; appends to the existing manifest otherwise.

**`discover_microcontroller_data_tool` parameters:**

| Parameter        | Type  | Default    | Description                                          |
|------------------|-------|------------|------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to the root directory to search        |

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

**Important:** This tool requires `microcontroller_manifest.yaml` files in DataLogger output directories.
These manifests are written automatically by `MicroControllerInterface.__init__()`. For legacy sessions
without manifests, use `write_microcontroller_manifest_tool` to retroactively tag log directories before
running discovery.

### Archive assembly

| Tool                         | Purpose                                                           |
|------------------------------|-------------------------------------------------------------------|
| `assemble_log_archives_tool` | Consolidates raw .npy log entries into .npz archives by source ID |

**Parameters:**

| Parameter          | Type   | Default    | Description                                                  |
|--------------------|--------|------------|--------------------------------------------------------------|
| `log_directory`    | `str`  | (required) | Absolute path to DataLogger output directory. **Ask user.**  |
| `remove_sources`   | `bool` | `true`     | Delete original .npy files after assembly                    |
| `verify_integrity` | `bool` | `false`    | Verify archive integrity before removing sources             |

**Return structure:**
```text
status:         "assembled"
directory:      Path to the log directory
archives:       List of created archive filenames (e.g., ["101_log.npz", "102_log.npz"])
source_ids:     List of extracted source ID strings
archive_count:  Number of archives created
```

---

## Workflows

### Microcontroller discovery

Run this to identify which microcontrollers are connected and their IDs:

1. Call `list_microcontrollers` (adjust `baudrate` if using non-default UART settings)
2. Record the device paths and controller IDs for identified microcontrollers
3. If no microcontrollers appear:
   - Check physical USB connections
   - Verify the microcontroller is running ataraxis-micro-controller firmware
   - Check for serial port permissions (`/dev/ttyACM*` may require user group membership)
   - Try a different baudrate if using UART communication

### MQTT broker verification

1. Call `check_mqtt_broker` with the broker's host and port
2. If unreachable:
   - Verify the MQTT broker service is running (e.g., `systemctl status mosquitto`)
   - Check firewall rules allow connections on the specified port
   - Verify the host address is correct

### Manifest inspection and retroactive tagging

**Inspect an existing manifest:**
1. Call `read_microcontroller_manifest_tool` with the manifest path
2. Review the controller and module entries

**Retroactively tag a legacy session:**
1. Call `write_microcontroller_manifest_tool` with the log directory, controller ID, controller name,
   and module list
2. Verify by calling `read_microcontroller_manifest_tool` on the created manifest
3. Run `discover_microcontroller_data_tool` to confirm the session is now discoverable

### Post-session archive assembly

After a recording session, assemble raw logs into archives:

1. Ask the user for the DataLogger output directory path
2. Call `assemble_log_archives_tool` with the directory
3. Verify with `discover_microcontroller_data_tool` to confirm archives and manifests are present
4. Proceed to `/extraction-configuration` for config setup or `/log-processing` if config exists

---

## Bridge to code integration

When transitioning from MCP-based discovery to writing MicroControllerInterface code, use this mapping.

### Parameter mapping

| MCP Discovery                            | Code Parameter                | How                                                  |
|------------------------------------------|-------------------------------|------------------------------------------------------|
| Device path from `list_microcontrollers` | `port`                        | Pass the device path directly (e.g., `/dev/ttyACM0`) |
| Microcontroller ID                       | `controller_id`               | Use the discovered ID as `np.uint8(id)`              |
| Baudrate used in discovery               | `baudrate`                    | Same value (default: 115200)                         |
| MQTT broker host/port                    | `MQTTCommunication(ip, port)` | Pass to MQTTCommunication constructor                |

### System ID semantics

| Range   | Assignment                         | Notes                                             |
|---------|------------------------------------|---------------------------------------------------|
| 101-150 | MicroControllerInterface instances | Advised production range; not enforced            |
| 1-255   | Valid range                        | Any np.uint8 value; must be unique per DataLogger |

Allocate controller IDs sequentially starting at 101 (e.g., 101, 102, 103 for a 3-controller setup).
Source IDs must be unique across **all** sources sharing a DataLogger, including sources from other
libraries (e.g., ataraxis-video-system). The 101-150 range avoids collisions with other libraries'
advised ranges.

---

## Troubleshooting

| Symptom                                           | Likely Cause                          | Resolution                                                |
|---------------------------------------------------|---------------------------------------|-----------------------------------------------------------|
| `list_microcontrollers` → "No valid serial ports" | No USB devices connected              | Check physical connections and USB cables                 |
| Port shows "No microcontroller"                   | Firmware not loaded or wrong baudrate | Verify firmware and try alternate baudrate                |
| Port shows "Connection Failed"                    | Permission denied or port in use      | Check serial port permissions; close conflicting programs |
| MQTT broker unreachable                           | Broker not running                    | Start the broker service                                  |
| Assembly fails                                    | Directory has no .npy files           | Verify DataLogger was stopped and flushed                 |
| Discovery finds no sources                        | Missing manifest files                | Use `write_microcontroller_manifest_tool` to tag sessions |
| MCP tools unavailable                             | Server not running                    | Use `/mcp-environment-setup` to diagnose                  |

---

## Related skills

| Skill                        | Relationship                                                       |
|------------------------------|--------------------------------------------------------------------|
| `/microcontroller-interface` | Covers writing MicroControllerInterface code after testing via MCP |
| `/extraction-configuration`  | Downstream: configure extraction parameters before processing      |
| `/log-input-format`          | Reference: documents the archive format produced by this workflow  |
| `/log-processing`            | Downstream: processes archives assembled by this skill             |
| `/log-processing-results`    | Downstream: analyzes output from processed archives                |
| `/pipeline`                  | Context: end-to-end orchestration and multi-controller planning    |
| `/mcp-environment-setup`     | Prerequisite: MCP server connectivity for all tool interactions    |

---

## Verification checklist

```text
Microcontroller Setup:
- [ ] Discovered microcontrollers via list_microcontrollers (recorded device paths and IDs)
- [ ] Verified MQTT broker connectivity if needed (check_mqtt_broker)
- [ ] Confirmed microcontroller manifests exist in DataLogger output directories
- [ ] Assembled log archives if needed (assemble_log_archives_tool)
- [ ] Verified recordings discoverable via discover_microcontroller_data_tool
- [ ] Noted controller IDs for extraction configuration
```
