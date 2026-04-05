---
name: extraction-configuration
description: >-
  Complete reference for ExtractionConfig parameters, generation from manifest, validation, and lifecycle.
  Covers the full extraction configuration data model, MCP tools for reading, writing, and validating
  configs, event code semantics, and config lifecycle workflow. Use when creating, reading, writing, or
  validating extraction configurations for the log processing pipeline.
user-invocable: true
---

# Extraction configuration reference

Complete parameter reference for the extraction configuration system that controls which microcontroller
data is extracted from log archives during the log processing pipeline.

---

## Scope

**Covers:**
- ExtractionConfig dataclass hierarchy and all configuration parameters
- Config generation from microcontroller manifests
- Config reading, writing, and validation via MCP tools
- Event code semantics and assignment guidance
- Config lifecycle workflow from manifest to processing
- Example YAML structure
- Validation rules (structural and manifest cross-reference)

**Does not cover:**
- Log processing workflow, batch operations, or status monitoring (see `/log-processing`)
- Output data formats or event analysis (see `/log-processing-results`)
- Manifest management or hardware discovery (see `/microcontroller-setup`)
- MCP server connectivity issues (see `/mcp-environment-setup`)

**Handoff rules:** If the user asks about processing workflow or batch execution, invoke `/log-processing`.
If the user asks about output data, invoke `/log-processing-results`. If the user needs to create or
inspect a manifest first, invoke `/microcontroller-setup`.

---

## Agent requirements

You MUST use the ataraxis-communication-interface MCP tools for all configuration operations. Do not
import configuration Python classes directly or edit YAML files manually. If MCP tools are not available,
invoke `/mcp-environment-setup` to diagnose and resolve connectivity issues.

You MUST ask the user for event codes for each module and kernel entry. Event codes are firmware-specific
and cannot be determined by inspecting the codebase. Never guess or assume event codes.

You MUST validate the configuration before handing off to `/log-processing`. An invalid config will cause
processing failures.

---

## MCP configuration tools

| Tool                              | Purpose                                                          |
|-----------------------------------|------------------------------------------------------------------|
| `read_extraction_config_tool`     | Reads config YAML and returns structured controller data         |
| `write_extraction_config_tool`    | Writes config from structured controller data to YAML            |
| `validate_extraction_config_tool` | Validates config structure, optionally cross-references manifest |

### `read_extraction_config_tool`

| Parameter     | Type  | Default    | Description                                          |
|---------------|-------|------------|------------------------------------------------------|
| `config_path` | `str` | (required) | Absolute path to the extraction configuration YAML   |

**Return structure:**
```text
config_path:          Path to the config file
controllers[]:        List of controller extraction entries:
  controller_id:      Controller ID (integer)
  modules[]:          Module extraction entries:
    module_type:      Module type code (integer)
    module_id:        Module instance ID (integer)
    event_codes:      List of event codes to extract (integers)
  kernel:             Kernel extraction entry (or null):
    event_codes:      List of kernel event codes to extract (integers)
total_controllers:    Number of controller entries
```

### `write_extraction_config_tool`

| Parameter     | Type         | Default    | Description                                             |
|---------------|--------------|------------|---------------------------------------------------------|
| `config_path` | `str`        | (required) | Absolute path for the output YAML file                  |
| `controllers` | `list[dict]` | (required) | Controller extraction descriptors (see structure below) |

Each controller dictionary must have:
- `controller_id` (int): The controller ID to extract from
- `modules` (list[dict]): Module entries, each with `module_type` (int), `module_id` (int), `event_codes` (list[int])
- `kernel` (dict or null): Optional kernel entry with `event_codes` (list[int])

### `validate_extraction_config_tool`

| Parameter       | Type         | Default    | Description                                             |
|-----------------|--------------|------------|---------------------------------------------------------|
| `config_path`   | `str`        | (required) | Absolute path to the extraction config YAML to validate |
| `manifest_path` | `str / None` | `None`     | Optional manifest path for cross-reference validation   |

**Return structure:**
```text
valid:             Boolean indicating overall validity
config_path:       Path to the validated config file
errors[]:          List of validation error strings (empty if valid)
summary:
  total_controllers:  Number of controller entries
  total_modules:      Total module entries across all controllers
  controllers_with_kernel:  Number of controllers with kernel extraction
```

---

## Configuration data model

The extraction configuration uses a nested dataclass hierarchy:

```text
ExtractionConfig (YamlConfig)
└── controllers: list[ControllerExtractionConfig]
    ├── controller_id: int
    ├── modules: tuple[ModuleExtractionConfig, ...]
    │   ├── module_type: int       # Hardware module family code
    │   ├── module_id: int         # Hardware module instance ID
    │   └── event_codes: tuple[int, ...]  # Non-empty, unique event codes
    └── kernel: KernelExtractionConfig | None
        └── event_codes: tuple[int, ...]  # Non-empty, unique kernel event codes
```

### Parameter reference

**ControllerExtractionConfig:**

| Field           | Type                                 | Description                                     |
|-----------------|--------------------------------------|-------------------------------------------------|
| `controller_id` | `int`                                | The controller ID to extract data from          |
| `modules`       | `tuple[ModuleExtractionConfig, ...]` | Module extraction entries (empty with kernel)   |
| `kernel`        | `KernelExtractionConfig / None`      | Optional kernel message extraction              |

**ModuleExtractionConfig:**

| Field         | Type              | Description                                      |
|---------------|-------------------|--------------------------------------------------|
| `module_type` | `int`             | Hardware module family code (matches firmware)   |
| `module_id`   | `int`             | Hardware module instance ID (matches firmware)   |
| `event_codes` | `tuple[int, ...]` | Non-empty tuple of unique event codes to extract |

**KernelExtractionConfig:**

| Field         | Type              | Description                                  |
|---------------|-------------------|----------------------------------------------|
| `event_codes` | `tuple[int, ...]` | Non-empty tuple of unique kernel event codes |

---

## Event code semantics

Event codes identify specific message types within a module's or kernel's communication protocol. They
are defined in the firmware running on the microcontroller and are globally unique within each module
instance.

- **Module event codes** correspond to specific data or state messages sent by the hardware module
  (e.g., encoder position updates, sensor readings, command completion signals)
- **Kernel event codes** correspond to system-level messages (e.g., controller status, error reports,
  keepalive signals)
- Event codes above 50 are user-defined module events; codes 1-50 are reserved for system service messages

The agent CANNOT determine which event codes to use by inspecting the codebase. Event codes are
firmware-specific knowledge that must come from the user.

---

## Config lifecycle workflow

### Step 1: Discover recordings

Use `discover_microcontroller_data_tool` (see `/microcontroller-setup`) to locate manifest files and
identify which controllers and modules were active during recording.

### Step 2: Read manifest

Use `read_microcontroller_manifest_tool` (see `/microcontroller-setup`) to inspect the controller and
module entries. Record the controller IDs, module types, and module IDs.

### Step 3: Generate precursor config

Build the config structure with all controllers and modules from the manifest but with empty event codes.
Present the structure to the user so they can see which controllers and modules are available.

### Step 4: Ask user for event codes

For each module and (optionally) each kernel entry, ask the user which event codes to extract. Present
the modules clearly:

```text
Controller 101 ("teensy_main"):
  Module type=1, id=1 ("encoder"): Which event codes?
  Module type=2, id=1 ("lick_sensor"): Which event codes?
  Kernel: Extract kernel messages? If yes, which event codes?
```

### Step 5: Write completed config

Call `write_extraction_config_tool` with the user-provided event codes populated. Ask the user for the
output file path.

### Step 6: Validate config

Call `validate_extraction_config_tool` with the config path and optionally the manifest path for
cross-reference validation. Report any errors to the user.

### Step 7: Handoff to processing

Once validation passes, the config is ready for `/log-processing`. The config path is passed to
`prepare_log_processing_batch_tool` as the `config_path` parameter.

---

## Example YAML structure

A typical extraction configuration YAML file:

```yaml
controllers:
- controller_id: 101
  modules:
  - module_type: 1
    module_id: 1
    event_codes:
    - 51
    - 52
    - 53
  - module_type: 2
    module_id: 1
    event_codes:
    - 51
  kernel:
    event_codes:
    - 2
    - 10
- controller_id: 102
  modules:
  - module_type: 3
    module_id: 1
    event_codes:
    - 51
    - 52
  kernel: null
```

---

## Validation rules

### Structural validation

| Rule                                   | Error message pattern                    |
|----------------------------------------|------------------------------------------|
| Non-empty controllers list             | "No controller entries"                  |
| Each controller has extraction targets | "Controller has no modules or kernel"    |
| Non-empty event codes per entry        | "Empty event_codes for module/kernel"    |
| No duplicate event codes per entry     | "Duplicate event codes in module/kernel" |
| Unique (type, id) pairs per controller | "Duplicate module (type, id) pair"       |

### Manifest cross-reference validation (when manifest_path provided)

| Rule                                         | Error message pattern                           |
|----------------------------------------------|-------------------------------------------------|
| Controller ID exists in manifest             | "Controller ID not found in manifest"           |
| Module (type, id) pair exists in manifest    | "Module (type, id) not registered in manifest"  |

---

## Troubleshooting

| Issue                              | Resolution                                                       |
|------------------------------------|------------------------------------------------------------------|
| "Config file not found"            | Verify the config path exists                                    |
| "Unable to read extraction config" | Check YAML syntax; regenerate if corrupted                       |
| "Invalid module descriptor"        | Ensure each module has `module_type`, `module_id`, `event_codes` |
| Validation errors after write      | Fix the reported issues and re-validate                          |
| Controller ID not in manifest      | Verify the controller ID matches the manifest exactly            |
| Module not in manifest             | Check module_type and module_id match the manifest entries       |

---

## Related skills

| Skill                        | Relationship                                                     |
|------------------------------|------------------------------------------------------------------|
| `/microcontroller-setup`     | Upstream: manifest creation and recording discovery              |
| `/microcontroller-interface` | Upstream: code that produces the manifests used here             |
| `/log-input-format`          | Reference: archive format that extraction config targets         |
| `/log-processing`            | Downstream: consumes the validated extraction config             |
| `/log-processing-results`    | Downstream: output format depends on config targets              |
| `/pipeline`                  | Context: extraction config is phase 3 of the end-to-end pipeline |
| `/mcp-environment-setup`     | Prerequisite: MCP server connectivity for config tools           |

---

## Verification checklist

```text
Extraction Configuration:
- [ ] MCP server connected (if not, invoke `/mcp-environment-setup`)
- [ ] Manifest inspected to identify controllers and modules
- [ ] Event codes obtained from user for each module and kernel entry
- [ ] Config written via `write_extraction_config_tool`
- [ ] Config validated via `validate_extraction_config_tool` (with manifest cross-reference)
- [ ] All validation errors resolved
- [ ] Config path ready for `/log-processing` handoff
```
