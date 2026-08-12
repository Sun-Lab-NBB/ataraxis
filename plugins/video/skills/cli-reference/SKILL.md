---
name: cli-reference
description: >-
  Canonical reference for the human-facing axvs command-line interface: every command and option with its short
  form, long form, type, default, and effect, the MCP tool each command maps to, its failure modes, and how the
  CLI path diverges from the MCP path. Use when a user asks what an axvs command or option does, or when the MCP
  server is unavailable and the user must be told what to run by hand.
user-invocable: false
---

# CLI reference

> **The `axvs` CLI is a HUMAN-FACING tool. You MUST never invoke it.** Print the command, ask the user to run it,
> ask them to paste the output back.

**The one exemption** is `--help`. `axvs --help` and `axvs COMMAND --help` may be run, and no other `axvs` invocation is
exempt. `/video-mcp-environment-setup` owns that exemption.

---

## Scope

**Covers:**
- The complete `axvs` command surface: every node, its purpose, and its MCP equivalent
- Every declared option: short form, long form, type, default, required / flag / repeatable status, and effect
- Per-command failure modes and the exception each raises
- How `axvs run` diverges from an MCP video session, and how `axvs process` diverges from the MCP batch
- What to tell a user to run when the MCP server cannot be restored in this session

**Does not cover:**
- The MCP session tools, GenICam tools, and manifest tools (see `/camera-setup`)
- The MCP batch workflow, lenient sourcing, and resource sizing (see `/log-processing`)
- Writing VideoSystem code, which no `axvs` command replaces (see `/camera-interface`)
- Diagnosing why the MCP server is down (see `/video-mcp-environment-setup`)
- Driving log processing from Python. Orchestration runs through MCP or this CLI only

**Handoff rules:** If the user wants an operation performed rather than explained, use the MCP tools and invoke the
owning skill. If the MCP tools are unavailable, invoke `/video-mcp-environment-setup` first and fall back to the
CLI-command handoff table below only after the server cannot be restored.

---

## Agent requirements

You MUST answer CLI questions from this skill or from `axvs --help`, never from memory. When a user's report disagrees
with this reference, ask them to run `axvs COMMAND --help` and read the installed build's answer.

---

## Command surface

The CLI declares fifteen Click nodes: the root group, three subgroups, and eleven leaf commands. The entry point is
`axvs = "ataraxis_video_system.interfaces.cli:axvs_cli"`.

| Click node                 | Kind    | Purpose                                                           | MCP equivalent                    |
|----------------------------|---------|-------------------------------------------------------------------|-----------------------------------|
| `axvs`                     | group   | Entry point. Dispatches to the subcommands and prints the listing | None, dispatch only               |
| `axvs cti`                 | group   | Dispatches the GenTL Producer (.cti) file subcommands             | None, dispatch only               |
| `axvs cti set`             | command | Configures the GenTL Producer file used by every future runtime   | `set_cti_file_tool`               |
| `axvs cti check`           | command | Reports whether a valid Producer file is configured               | `get_cti_status_tool`             |
| `axvs check`               | group   | Dispatches the discovery and host-compatibility subcommands       | None, dispatch only               |
| `axvs check devices`       | command | Discovers cameras on both interfaces and prints their indices     | `list_cameras_tool`               |
| `axvs check compatibility` | command | Reports FFMPEG and NVIDIA GPU video-encoding support              | `check_runtime_requirements_tool` |
| `axvs run`                 | command | Runs an interactive single-camera imaging session                 | The session tools, see below      |
| `axvs process`             | command | Extracts frame timestamps from ONE log directory's archives       | The batch tools, see below        |
| `axvs mcp`                 | command | Starts the MCP server                                             | None, this command IS the server  |
| `axvs configure`           | group   | Dispatches the GenICam configuration subcommands                  | None, dispatch only               |
| `axvs configure read`      | command | Reads one GenICam node or lists every writable GenICam node       | `read_genicam_node_tool`          |
| `axvs configure write`     | command | Writes a value to one GenICam node                                | `write_genicam_node_tool`         |
| `axvs configure dump`      | command | Exports the camera's full configuration to a YAML file            | `dump_genicam_config_tool`        |
| `axvs configure load`      | command | Applies a YAML configuration file to the camera                   | `load_genicam_config_tool`        |

`axvs run` maps to `start_video_session_tool`, `start_frame_saving_tool`, `stop_frame_saving_tool`, and
`stop_video_session_tool` taken together. `axvs process` maps to `prepare_log_processing_batch_tool` and
`execute_log_processing_jobs_tool` taken together. Both mappings carry the divergences documented below.

**Note:** Twenty-five options are declared across the surface. Click adds `--help` to every node on top of those. The
CLI leaves `help_option_names` at its `["--help"]` default, so `-h` is never a help alias. On `axvs run`, `-h` is bound
to `--height` and consumes the next token as a pixel count, so `axvs run -h` fails on the missing value rather than
printing help.

---

## Option reference

Every option below is declared in `interfaces/cli.py`. "Required" means Click rejects the invocation without it (exit
code 2). Path options carry Click `click.Path` constraints, listed under Effect.

### `axvs cti set`

| Short | Long          | Type   | Default    | Form     | Effect                                                            |
|-------|---------------|--------|------------|----------|-------------------------------------------------------------------|
| `-f`  | `--file-path` | `Path` | (required) | required | The vendor `.cti` Producer to persist. Must exist and be readable |

`axvs cti check` and both `axvs check` subcommands declare no options of their own.

### `axvs run`

| Short | Long                 | Type     | Default    | Form     | Effect                                                                   |
|-------|----------------------|----------|------------|----------|--------------------------------------------------------------------------|
| `-i`  | `--interface`        | `Choice` | `"mock"`   | optional | Camera interface. Choice of `mock`, `harvesters`, and `opencv`           |
| `-c`  | `--camera-index`     | `int`    | `0`        | optional | Index of the camera within the chosen interface's discovery listing      |
| `-g`  | `--gpu-index`        | `int`    | `-1`       | optional | GPU device for encoding. Any value below zero selects CPU encoding       |
| `-o`  | `--output-directory` | `Path`   | (required) | required | Where the `.mp4` and the log directory are written. Must already exist   |
| `-m`  | `--monochrome`       | flag     | `False`    | flag     | Records grayscale. Applies to `opencv` and `mock` only, never Harvesters |
| `-w`  | `--width`            | `int`    | `600`      | optional | Frame width in pixels                                                    |
| `-h`  | `--height`           | `int`    | `400`      | optional | Frame height in pixels. **This is why `-h` is not a help alias**         |
| `-f`  | `--frame-rate`       | `int`    | `30`       | optional | Acquisition rate in frames per second                                    |

**Note on `-i`:** the default is `mock`, which produces synthetic frames and touches no hardware. A user testing a real
camera must name `opencv` or `harvesters` explicitly, so always include the option in a command handed to a user.

**Note on `-o`:** Click requires the directory to exist already, unlike `axvs process -od`. A user pointing at a path
they have not created yet is rejected at exit code 2 before the session starts.

### `axvs process`

| Short | Long                 | Type   | Default    | Form       | Effect                                                                            |
|-------|----------------------|--------|------------|------------|-----------------------------------------------------------------------------------|
| `-ld` | `--log-directory`    | `Path` | (required) | required   | The ONE DataLogger output directory, searched recursively for `.npz` archives     |
| `-od` | `--output-directory` | `Path` | (required) | required   | The output root. A `camera_timestamps/` subdirectory holding every output is made |
| `-id` | `--job-id`           | `str`  | `None`     | optional   | Runs ONLY the job whose hexadecimal identifier matches. Suppresses `-s`           |
| `-s`  | `--specifier`        | `str`  | `()`       | repeatable | A camera source ID to process. Repeat once per camera. Ignored when `-id` is set  |
| `-w`  | `--workers`          | `int`  | `-1`       | optional   | The ceiling on the workers ONE job receives, not a batch width. See the note      |
| `-np` | `--no-progress`      | flag   | `False`    | flag       | Suppresses the extraction progress bars. Bars are displayed by default            |

**Note on `-s`:** when omitted, every source the recording's `camera_manifest.yaml` registers is processed. Each named
ID must be registered in that manifest and must resolve to exactly one archive beneath `-ld`.

**Note on `-w`:** a value below 1 (the `-1` default) resolves the ceiling from the host's core count, and `-w 1` makes
every job sequential. Every job then runs at that one ceiling, because this path runs no per-archive sizing pass at all.
An archive below the library's parallel processing threshold still falls back to a sequential run whatever the ceiling
says. This is the sharpest divergence from the MCP batch, which sizes each job from its own archive.

**Note on `-id`:** a job identifier is a deterministic function of the job name and the camera's `source_id`, so one
camera keeps the same identifier across recordings. A user obtains it from the `job_id` key of a
`prepare_log_processing_batch_tool` job entry, or from the tracker YAML.

### `axvs mcp`

| Short | Long          | Type     | Default   | Form     | Effect                                                                    |
|-------|---------------|----------|-----------|----------|---------------------------------------------------------------------------|
| `-t`  | `--transport` | `Choice` | `"stdio"` | optional | Transport protocol. Choice of `stdio` and `streamable-http`, nothing else |

**Note:** `run_server()` also accepts a third value, `sse`, but the CLI `Choice` rejects it (exit code 2). Never pass
`-t sse`, and never tell a user to.

### `axvs configure` (group options)

| Short | Long                 | Type  | Default                        | Form       | Effect                                                                     |
|-------|----------------------|-------|--------------------------------|------------|----------------------------------------------------------------------------|
| `-c`  | `--camera-index`     | `int` | `None`                         | optional   | The Harvesters camera every subcommand operates on                         |
| `-b`  | `--blacklisted-node` | `str` | the library's GenICam node set | repeatable | A GenICam node excluded from read, dump, and load. Repeat per GenICam node |
|       | `--no-blacklist`     | flag  | `False`                        | flag       | Includes every writable GenICam node in read, dump, and load               |

**All three are parsed on the group**, so they must be supplied **before** the subcommand name, as in
`axvs configure -c 0 read -n Width`. Placing `-c` after the subcommand name aborts with "No such option". The option
carries no usable default, so omitting it raises a Click usage error naming the omission when the subcommand runs. It
cannot be marked required on the group without also blocking `axvs configure SUBCOMMAND --help`.

`-b` and `--no-blacklist` are mutually exclusive, and supplying both raises a Click usage error. Because `-b` carries a
non-empty default, the CLI settles the conflict by consulting Click's parameter source rather than the parsed tuple, so
the error fires only when the user actually named a GenICam node.

### `axvs configure` subcommands

| Command | Short | Long            | Type   | Default    | Form     | Effect                                                                  |
|---------|-------|-----------------|--------|------------|----------|-------------------------------------------------------------------------|
| `read`  | `-n`  | `--node-name`   | `str`  | `""`       | optional | The GenICam node to read. When empty, lists every writable GenICam node |
| `write` | `-n`  | `--node-name`   | `str`  | (required) | required | The GenICam node to write. Written even when the blacklist names it     |
| `write` | `-v`  | `--value`       | `str`  | (required) | required | The value, coerced to the GenICam node's own type                       |
| `dump`  | `-o`  | `--output-file` | `Path` | (required) | required | The YAML file to write. Need not exist                                  |
| `load`  | `-f`  | `--config-file` | `Path` | (required) | required | The YAML file to apply. Must exist and be readable                      |
| `load`  |       | `--strict`      | flag   | `False`    | flag     | Aborts on a camera model or serial mismatch instead of warning          |

**Note:** `-f` expands to `--file-path` on `axvs cti set`, to `--frame-rate` on `axvs run`, and to `--config-file` on
`axvs configure load`. The long forms are not interchangeable, so quote the long form for any of the three.

---

## Command behavior and failure modes

### `axvs cti set` and `axvs cti check`

`set` persists the Producer path for every later runtime and echoes `AXVS CTI file: Set to {path}.` on success.

`check` reports the platform limitation first, before it evaluates any stored path, because a host without the GenICam
runtime cannot validate a Producer and would otherwise blame the configuration for a platform gate.

| Observed                                         | Meaning                                                               |
|--------------------------------------------------|-----------------------------------------------------------------------|
| `AXVS CTI file: Unable to check. {reason}`       | The GenICam runtime is absent. See `/camera-setup` for the two causes |
| `AXVS CTI file: Configured and valid. Path: ...` | A Producer is set and still resolves                                  |
| `AXVS CTI file: Not configured or invalid.`      | No Producer is set, or the stored path no longer resolves             |

The `AXVS_CTI_PATH` environment variable supplies the path for a single runtime and takes precedence over the persisted
one, so a host reporting a Producer nobody configured has that variable set.

### `axvs check devices`

Probes OpenCV indices and enumerates GenTL devices, then prints each interface's cameras under its own heading. The line
format differs from `list_cameras_tool`, which returns the compact `OpenCV #0: 1920x1080@30fps` form. The CLI prints
`OpenCV camera 1: index=0, frame_height=..., frame_width=..., frame_rate=...` and numbers its lines from 1, so never
read the leading number as a camera index. Read `index=` instead.

| Condition              | Behavior                                                                                       |
|------------------------|------------------------------------------------------------------------------------------------|
| No OpenCV cameras      | Prints a warning line. Not an error                                                            |
| OpenCV cameras present | Warns first that OpenCV resolves no model or serial, and recommends `axvs run` to map indices  |
| GenICam runtime absent | Prints `Harvesters camera discovery skipped.` with the reason, and probes no GenTL device      |
| No Harvesters cameras  | Prints a warning line. Distinct from the skipped case above, so read which of the two appeared |

### `axvs check compatibility`

Reports one of three states and never raises. `Not met.` means FFMPEG is absent, and the GPU is not evaluated at all in
that branch. `Partially met.` means FFMPEG resolved but no NVIDIA GPU answered `nvidia-smi`. `Fully met.` means both.

### `axvs run`

Builds a DataLogger and one VideoSystem, then drives them from an interactive keypress loop. `q` terminates, `w` starts
frame saving, and `s` stops it. Every other key prints a warning and re-prompts.

The session's parameters are **fixed in the command**, and no option exposes them. This is the third parameter set in
the plugin, alongside the MCP session defaults in `/camera-setup` and the code defaults in `/camera-interface`.

| Fixed value                       | Consequence                                                                    |
|-----------------------------------|--------------------------------------------------------------------------------|
| `system_id=111`                   | The archive is `111_log.npz` and the video is `111.mp4`                        |
| `name="live_camera"`              | The manifest entry is registered under that name                               |
| `instance_name="axvs_live_run"`   | The log directory is `axvs_live_run_data_log/`, not the MCP session's          |
| `display_frame_rate=25`           | A preview window opens on every platform except macOS                          |
| `H264`, `FAST`, `YUV420`, `QP 15` | Compatibility-first encoding. Not the production set `/camera-interface` gives |

Archive assembly runs in a `finally` block, so an ordinary interrupt still assembles. Only a killed process leaves raw
`.npy` entries behind, and `/post-recording` owns the manual recovery.

**On termination the system may stay alive for up to 600 seconds** draining buffered frames. A user reporting a hung
`axvs run` shortly after pressing `q` is describing normal behavior.

### `axvs process`

Runs one recording's jobs **sequentially in a plain loop**.

| Condition                                  | Behavior                                                                                |
|--------------------------------------------|-----------------------------------------------------------------------------------------|
| No manifest, or no resolvable source       | `FileNotFoundError` naming the log directory. The recording resolved no job             |
| Several manifests under `-ld`              | `ValueError`. Pass each DataLogger output directory on its own invocation               |
| Manifest registers no source               | `ValueError`                                                                            |
| A requested source or job ID unregistered  | `ValueError`                                                                            |
| A source's archive absent or ambiguous     | `FileNotFoundError`                                                                     |
| Resolved archives span several directories | `ValueError` naming the one-directory-per-invocation rule                               |
| A job raises mid-run                       | The tracker marks that job FAILED, the exception propagates, and later jobs never start |

That last row is the divergence that matters most. The MCP batch isolates a failure to its own job and carries the rest
of the batch through, while the CLI abandons the remaining jobs at SCHEDULED. A user reporting "it stopped partway" has
hit a failure, not a cancellation.

**Note on tracker contention:** both paths lock the same `camera_processing_tracker.yaml` through its `.lock` file, on
the CLI side when the pipeline aligns the tracker and again on every job state transition. A user running `axvs process`
against a directory the agent is preparing or executing makes whichever side asks second raise a `TimeoutError`. Ask the
user to stop their CLI run before preparing or executing that directory, and never start an MCP batch over a directory
the user has a run in.

---

## Fallback: what to tell a user when MCP is unavailable

Every CLI-reachable capability appears below. Confirm the server is genuinely unrecoverable through
`/video-mcp-environment-setup` before handing any of these over.

| Blocked MCP tool                                                         | Tell the user to run                                   |
|--------------------------------------------------------------------------|--------------------------------------------------------|
| `set_cti_file_tool`                                                      | `axvs cti set -f <cti>`                                |
| `get_cti_status_tool`                                                    | `axvs cti check`                                       |
| `list_cameras_tool`                                                      | `axvs check devices`                                   |
| `check_runtime_requirements_tool`                                        | `axvs check compatibility`                             |
| `read_genicam_node_tool`                                                 | `axvs configure -c <index> read -n <node>`             |
| `write_genicam_node_tool`                                                | `axvs configure -c <index> write -n <node> -v <value>` |
| `dump_genicam_config_tool`                                               | `axvs configure -c <index> dump -o <yaml>`             |
| `load_genicam_config_tool`                                               | `axvs configure -c <index> load -f <yaml>`             |
| The four session tools                                                   | `axvs run -i <interface> -c <index> -o <dir>`          |
| `prepare_log_processing_batch_tool` + `execute_log_processing_jobs_tool` | `axvs process -ld <logs> -od <out>`                    |

Three caveats. `axvs run` is keypress-driven and records at fixed compatibility encoding, so it substitutes for a
session test and never for a production recording. `axvs process` handles ONE log directory per invocation, so a batch
spanning several becomes one invocation per directory, and it aborts at the first failing job. Neither `axvs` command
reports its results back in a machine-readable form, so ask the user to paste the terminal output.

Everything else genuinely blocks until the server is back: camera manifest read and write, archive assembly, video file
validation, camera data discovery, every batch status, timing, cancel, reset, and cleanup tool, and frame statistics
analysis. Say so plainly rather than improvising a substitute.

---

## Related skills

| Skill                          | Relationship                                                                         |
|--------------------------------|--------------------------------------------------------------------------------------|
| `/camera-setup`                | Owns the session, GenICam, and manifest tools behind `axvs run` and `axvs configure` |
| `/log-processing`              | Owns the MCP batch workflow and the authoritative job sizing model                   |
| `/video-mcp-environment-setup` | Owns the `--help` exemption, the `axvs mcp` transports, and MCP recovery             |
| `/camera-interface`            | Context: the production VideoSystem code no `axvs` command replaces                  |
| `/post-recording`              | Downstream: recovers an `axvs run` session whose process was killed before assembly  |
| `/log-input-format`            | Reference: the assembled archives `axvs process` reads under `-ld`                   |
| `/log-processing-results`      | Downstream: the feather output both paths write                                      |
| `/pipeline`                    | Context: where each CLI command sits in the end-to-end pipeline                      |

---

## Verification checklist

```text
Answering a CLI question:
- [ ] Answered from this skill or from `axvs COMMAND --help`, never from memory
- [ ] Quoted the long option form, and never presented `-h` as a help alias
- [ ] Named the MCP equivalent alongside any command the agent could have run itself
- [ ] Invoked no `axvs` command other than `--help`

Handing a user a CLI command:
- [ ] Confirmed MCP is genuinely unavailable via `/video-mcp-environment-setup` first
- [ ] Printed the command for the user instead of running it
- [ ] Named `-i opencv` or `-i harvesters` explicitly on any `axvs run` command, since the default is `mock`
- [ ] Warned about the abort-on-first-failure loop before recommending `axvs process`
- [ ] Confirmed no MCP batch is holding the tracker for that output directory
```
