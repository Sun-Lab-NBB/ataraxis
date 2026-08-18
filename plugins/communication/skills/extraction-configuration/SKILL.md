---
name: extraction-configuration
description: >-
  Documents every ExtractionConfig parameter, its generation from a manifest, its validation, and its
  lifecycle. Covers the configuration data model, the MCP tools for reading, writing, and validating
  configs, event code semantics, and the config lifecycle workflow. Use when creating, reading, writing,
  or validating extraction configurations for the log processing pipeline.
user-invocable: false
---

# Extraction configuration reference

Documents the extraction configuration system that controls which microcontroller data is extracted from log archives
during the log processing pipeline.

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
- MCP server connectivity issues (see `/communication-mcp-environment-setup`)

---

## Agent requirements

You MUST use the ataraxis-communication-interface MCP tools for all configuration operations. Do not import
configuration Python classes directly or edit YAML files manually. If MCP tools are not available, invoke
`/communication-mcp-environment-setup` to diagnose and resolve connectivity issues.

You MUST ask the user for the custom event codes (51-250) of every module entry. Those codes are firmware-specific and
cannot be determined by inspecting the codebase. Never guess or assume them. Kernel event codes and module service codes
are a fixed library enumeration, so select those yourself instead of asking (see "Event code semantics").

You MUST validate the configuration before handing off to `/log-processing`. An invalid config will cause processing
failures.

---

## MCP configuration tools

| Tool                              | Purpose                                                          |
|-----------------------------------|------------------------------------------------------------------|
| `read_extraction_config_tool`     | Reads config YAML and returns structured controller data         |
| `write_extraction_config_tool`    | Writes config from structured controller data to YAML            |
| `validate_extraction_config_tool` | Validates config structure, optionally cross-references manifest |

### `read_extraction_config_tool`

| Parameter     | Type  | Default    | Description                                        |
|---------------|-------|------------|----------------------------------------------------|
| `config_path` | `str` | (required) | Absolute path to the extraction configuration YAML |

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

**Return structure:**
```text
success:            Boolean flag, true when the file was written
config_path:        Path to the written config file
controller_count:   Number of controller entries written
```

### `validate_extraction_config_tool`

| Parameter       | Type         | Default    | Description                                             |
|-----------------|--------------|------------|---------------------------------------------------------|
| `config_path`   | `str`        | (required) | Absolute path to the extraction config YAML to validate |
| `manifest_path` | `str / None` | `None`     | Manifest path for cross-reference validation            |

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

| Field           | Type                                 | Description                                   |
|-----------------|--------------------------------------|-----------------------------------------------|
| `controller_id` | `int`                                | The controller ID to extract data from        |
| `modules`       | `tuple[ModuleExtractionConfig, ...]` | Module extraction entries (empty with kernel) |
| `kernel`        | `KernelExtractionConfig / None`      | Optional kernel message extraction            |

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

Event codes identify specific message types within a module's or kernel's communication protocol. They are defined in
the firmware running on the microcontroller and are globally unique within each module instance.

- **Module event codes** correspond to specific data or state messages sent by the hardware module (e.g., encoder
  position updates, sensor readings, command completion signals)
- **Kernel event codes** correspond to system-level messages (e.g., controller status, error reports, keepalive signals)
- Module codes 51-250 are the user-defined events a specific firmware build declares. Codes 0-50 are reserved for the
  base Module class service messages, and codes above 250 are not valid custom codes

Extraction filters each module against only its own `event_codes` set, with no cross-module leakage. Listing the same
code value under two modules is therefore safe and isolated. Any message for an unlisted module or an unlisted event
code is silently excluded from output with no error.

The kernel menu spans 0-10 and the module service menu spans 0-3 inside the reserved 0-50 band.
`/microcontroller:firmware-module` owns both tables. ataraxis-communication-interface mirrors them as the
`KernelStatusCodes` and `ModuleStatusCodes` IntEnums.

---

## Config lifecycle workflow

**Reusing an existing config.** Before running the steps below, ask whether a config already exists for this recording.
If one does, call `read_extraction_config_tool` on it and show the user the controllers, modules, and event codes it
already carries. Resume at step 4 for the entries that need changing instead of regenerating the file, since a rewrite
discards every event code the user previously supplied.

### Step 1: Discover recordings

Use `discover_microcontroller_data_tool` (see `/microcontroller-setup`) to locate manifest files and identify which
controllers and modules were active during recording.

### Step 2: Read manifest

Use `read_microcontroller_manifest_tool` (see `/microcontroller-setup`) to inspect the controller and module entries.
Record the controller IDs, module types, and module IDs.

### Step 3: Generate precursor config

Build the config structure with all controllers and modules from the manifest. Present the structure to the user so they
can see which controllers and modules are available.

No MCP tool exposes precursor generation. The server registers exactly three configuration tools, read, write, and
validate, so pick one of two routes:

| Route            | Use when                                     | Procedure                                                          |
|------------------|----------------------------------------------|--------------------------------------------------------------------|
| Manual (default) | The manifest is small or already summarized  | Build the structure from the step 2 entries and carry it to step 4 |
| CLI-assisted     | The manifest is large or error-prone to copy | Have the user run `axci config create`, then read the result back  |

**CLI-assisted route.** Ask the user to run `axci config create -m <manifest_path> -o <output_path>`, then call
`read_extraction_config_tool` on `<output_path>`. Do not invoke `axci` yourself. This replaces hand-transcription of the
manifest with a tool read, so the controller IDs, module types, and module IDs cannot drift from the manifest.

**What the precursor contains.** One controller entry per registered controller, one module entry per registered module,
every `event_codes` list empty, and `kernel: null` on every entry. Kernel extraction is never pre-populated, so a user
who wants kernel messages needs a kernel entry added in step 5.

**Note:** a precursor round-trips through `read_extraction_config_tool` cleanly, but it fails
`validate_extraction_config_tool` on the empty-`event_codes` rule. Do not validate before step 5.

### Step 4: Ask user for event codes

Present the modules clearly:

```text
Controller 101 ("teensy_main"):
  Module type=1, id=1 ("encoder"): which custom event codes (51-250)?
  Module type=2, id=1 ("lick_sensor"): which custom event codes (51-250)?
  Kernel: extract kernel messages? (codes come from the fixed 0-10 menu, no user knowledge needed)
```

### Step 5: Write completed config

Call `write_extraction_config_tool` with the user-provided event codes populated. Ask the user for the output file path,
offering `extraction_configuration.yaml` as the conventional name. That is the value of the library's
`EXTRACTION_CONFIGURATION_FILENAME` constant, and nothing enforces it, so any path the user names is accepted.

Then call `read_extraction_config_tool` on the same path and confirm the round trip. The write tool reports only
`controller_count`, so it cannot show that a module entry or a kernel entry landed with the codes intended. Compare the
returned `controllers` against what the user supplied before moving on.

### Step 6: Validate config

Call `validate_extraction_config_tool` with the config path **and** the manifest path. Always pass `manifest_path`, even
though the tool accepts `None`. Without it, only the structural rules run, and a config naming a controller or a module
the manifest never registered still reads `valid: true`. Report any errors to the user.

### Step 7: Handoff to processing

Once validation passes, the config is ready for `/log-processing`. The config path is passed to
`prepare_log_processing_batch_tool` as the `config_path` parameter.

---

## Example YAML structure

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

| Rule                                   | Error message template                                                                             |
|----------------------------------------|----------------------------------------------------------------------------------------------------|
| Non-empty controllers list             | "Config contains no controller entries."                                                           |
| Each controller has extraction targets | "Controller {id}: No modules and no kernel configured. At least one is required."                  |
| Non-empty event codes per entry        | "Controller {id}, module ({type}, {id}): event_codes is empty."                                    |
| No duplicate event codes per entry     | "Controller {id}, module ({type}, {id}): event_codes contains duplicates."                         |
| Unique (type, id) pairs per controller | "Controller {id}, module ({type}, {id}): Duplicate module (type, id) pair within this controller." |

Every entry-level message opens with the label of the offending entry, "Controller {id}, module ({type}, {id})" for a
module and "Controller {id}, kernel" for a kernel, so the two `event_codes` rules above read identically for both.

### Manifest cross-reference validation (when manifest_path provided)

| Rule                                      | Error message template                                                                    |
|-------------------------------------------|-------------------------------------------------------------------------------------------|
| Manifest path exists                      | "Manifest file not found: {manifest_path}"                                                |
| Manifest path points to a file            | "Manifest path is not a file: {manifest_path}"                                            |
| Manifest parses                           | "Unable to read manifest for cross-referencing: {error}"                                  |
| Controller ID exists in manifest          | "Controller {id}: Not registered in manifest. Registered IDs: {registered_ids}."          |
| Module (type, id) pair exists in manifest | "Controller {id}, module ({type}, {id}): Not registered in manifest for this controller." |

A `manifest_path` that does not exist, is not a file, or cannot be parsed is reported through `errors` (so `valid` reads
false), not as a tool-level error dictionary.

### Enforcement beyond validation

The config may list only a subset of the manifest's controllers and modules, because extraction is selective. A
requested `controller_id` that the manifest does not register, that this config does not declare, or whose `.npz`
archive is missing or ambiguous is **dropped and reported, not raised**, on the MCP preparation path. The dropped ID
lands in that directory's `skipped_sources` list while the batch still reports success.

`/log-processing` is authoritative for this model. Its `references/error-routing.md` carries the "Lenient sourcing"
section naming the three skip reasons and the remedy each one routes to. Leniency belongs to the MCP path alone, so do
not expect an exception there when a controller is dropped.

The consequence here: `validate_extraction_config_tool` never inspects the log directory, so a structurally valid config
can silently extract nothing. The manifest cross-reference in step 6 catches the registration half of that before
processing. The archive half is only visible in `skipped_sources` after preparing.

The empty-`event_codes` rule is checked by `validate_extraction_config_tool` but not by preparation. A module or kernel
entry with an empty `event_codes` list, and a controller entry declaring neither modules nor kernel, each fail that
controller's job with a `ValueError` at runtime. A config that skipped step 6 therefore reaches execution before the
problem surfaces.

---

## Troubleshooting

| Issue                               | Resolution                                                       |
|-------------------------------------|------------------------------------------------------------------|
| "Config file not found"             | Verify the config path exists                                    |
| "Path is not a file"                | Point `config_path` at the .yaml file, not its directory         |
| "Unable to read extraction config"  | Check YAML syntax. Regenerate if corrupted                       |
| "Unable to parse extraction config" | Validation could not load the YAML. Check syntax                 |
| "Unable to write extraction config" | Check that the parent directory is writable                      |
| "Invalid controller data"           | Ensure each module has `module_type`, `module_id`, `event_codes` |
| Validation errors after write       | Fix the reported issues and re-validate                          |
| Controller ID not in manifest       | Verify the controller ID matches the manifest exactly            |
| Module not in manifest              | Check module_type and module_id match the manifest entries       |
| Valid config extracts nothing       | Re-validate with `manifest_path`, then read `skipped_sources`    |

---

## CLI reference (human-facing, do not invoke)

> **The `axci` CLI is a HUMAN-FACING tool. You MUST never invoke it.** Every agent-driven configuration operation has
> an equivalent MCP tool. `/cli-reference` is canonical for the whole `axci` command surface, including the options of
> the `axci config create` command step 3 hands to a user.

---

## Related skills

| Skill                                  | Relationship                                                               |
|----------------------------------------|----------------------------------------------------------------------------|
| `/microcontroller-setup`               | Upstream: manifest creation and recording discovery                        |
| `/microcontroller-interface`           | Upstream: code that produces the manifests used here                       |
| `/microcontroller:firmware-module`     | Upstream: firmware Module that defines the event codes this config targets |
| `/log-input-format`                    | Reference: archive format that extraction config targets                   |
| `/log-processing`                      | Downstream: consumes the validated extraction config                       |
| `/log-processing-results`              | Downstream: output format depends on config targets                        |
| `/cli-reference`                       | Reference: the full `axci` command surface, human-facing                   |
| `/pipeline`                            | Context: extraction config is phase 3 of the end-to-end pipeline           |
| `/communication-mcp-environment-setup` | Prerequisite: MCP server connectivity for config tools                     |

---

## Verification checklist

```text
Extraction Configuration:
- [ ] MCP server connected (if not, invoke `/communication-mcp-environment-setup`)
- [ ] Existing config, if any, read via `read_extraction_config_tool` before regenerating
- [ ] Manifest inspected to identify controllers and modules
- [ ] Custom module event codes (51-250) obtained from the user
- [ ] Kernel event codes selected from the fixed menu
- [ ] Config written via `write_extraction_config_tool`
- [ ] Written config read back via `read_extraction_config_tool` to confirm the round trip
- [ ] Config validated via `validate_extraction_config_tool` with `manifest_path` supplied
- [ ] All validation errors resolved
- [ ] Config path ready for `/log-processing` handoff
```
