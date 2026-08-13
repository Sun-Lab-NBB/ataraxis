---
name: cli-reference
description: >-
  Canonical reference for the human-facing axci command-line interface. Covers every command and option with its short
  form, long form, type, default, and effect, the MCP tool each command maps to, its failure modes, and how the CLI path
  diverges from the MCP path. Use when a user asks what an axci command or option does, or when the MCP server is
  unavailable and the user must be told what to run by hand.
user-invocable: false
---

# CLI reference

> **The `axci` CLI is a HUMAN-FACING tool. You MUST never invoke it.** Print the command, ask the user to run it, and
> ask them to paste the output back.

**The one exemption** is `--help`. `axci --help` and `axci COMMAND --help` may be run, and no other `axci` invocation is
exempt. `/communication-mcp-environment-setup` owns that exemption.

---

## Scope

**Covers:**
- The complete `axci` command surface: every node, its purpose, and its MCP equivalent
- Every declared option: short form, long form, type, default, required / flag / repeatable status, and effect
- Per-command failure modes and the exception each raises
- How `axci process` diverges from the MCP batch path, and why the two report the same fault differently
- What to tell a user to run when the MCP server cannot be restored in this session

**Does not cover:**
- The MCP batch workflow, lenient sourcing, and resource sizing (see `/log-processing`)
- Extraction config authoring, validation, and event-code semantics (see `/extraction-configuration`)
- Manifest management, archive assembly, and hardware discovery (see `/microcontroller-setup`)
- Diagnosing why the MCP server is down (see `/communication-mcp-environment-setup`)
- Driving log processing from Python. Orchestration runs through MCP or this CLI only

**Handoff rules:** If the user wants an operation performed rather than explained, use the MCP tools and invoke the
owning skill. If the MCP tools are unavailable, invoke `/communication-mcp-environment-setup` first and fall back to the
CLI-command handoff table below only after the server cannot be restored.

---

## Agent requirements

You MUST answer CLI questions from this skill or from `axci --help`, never from memory. When a user's report disagrees
with this reference, ask them to run `axci COMMAND --help` and read the installed build's answer.

---

## Command surface

The CLI declares eight Click nodes: the root group, one subgroup, and six leaf commands. The entry point is `axci =
"ataraxis_communication_interface.interfaces.cli:axci_cli"`.

| Node                 | Kind    | Purpose                                                                          | MCP equivalent                                                                       |
|----------------------|---------|----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `axci`               | group   | Entry point. Dispatches to the subcommands and prints the command listing        | None, dispatch only                                                                  |
| `axci id`            | command | Discovers connected Arduino/Teensy controllers running ataraxis-micro-controller | `list_microcontrollers_tool`                                                         |
| `axci mqtt`          | command | Reports whether an MQTT broker is reachable at a host and port                   | `check_mqtt_broker_tool`                                                             |
| `axci config`        | group   | Dispatches the extraction configuration subcommands                              | None, dispatch only                                                                  |
| `axci config create` | command | Writes a precursor extraction config from a microcontroller manifest             | None, user-run precursor route (`write_extraction_config_tool` writes a full config) |
| `axci config show`   | command | Prints a config's controllers, modules, event codes, and kernel setting          | `read_extraction_config_tool`                                                        |
| `axci process`       | command | Extracts module and kernel data from ONE log directory's archives                | `prepare_log_processing_batch_tool` + `execute_log_processing_jobs_tool` (batch)     |
| `axci mcp`           | command | Starts the MCP server                                                            | None, this command IS the server                                                     |

**Note:** Fourteen options are declared across the surface. Click adds `--help` to every node on top of those. The CLI
leaves `help_option_names` at its `["--help"]` default, so `-h` is never a help alias. On `axci mqtt`, `-h` is bound to
`--host` and consumes the next token as a hostname. Always quote the long form to a user.

---

## Option reference

Every option below is declared in `interfaces/cli.py`. "Required" means Click rejects the invocation without it (exit
code 2). Path options carry Click `click.Path` constraints, listed under Effect.

### `axci id`

| Short | Long         | Type  | Default  | Form     | Effect                                                                             |
|-------|--------------|-------|----------|----------|------------------------------------------------------------------------------------|
| `-b`  | `--baudrate` | `int` | `115200` | optional | Identification baudrate. Used only by UART controllers. Ignored by USB controllers |

**Note:** 115200 is the option default, not a universal board default. A UART board flashed at another speed reports
`[No microcontroller]` at the wrong baudrate rather than failing. `/microcontroller:firmware-module`, "Serial speed",
owns the contract these rates form with the PC, and `/platformio-config` owns the per-board `monitor_speed` values.

### `axci mqtt`

| Short | Long     | Type  | Default       | Form     | Effect                                    |
|-------|----------|-------|---------------|----------|-------------------------------------------|
| `-h`  | `--host` | `str` | `"127.0.0.1"` | optional | IP address or hostname of the MQTT broker |
| `-p`  | `--port` | `int` | `1883`        | optional | Socket port the MQTT broker listens on    |

### `axci config create`

| Short | Long              | Type   | Default    | Form     | Effect                                                                         |
|-------|-------------------|--------|------------|----------|--------------------------------------------------------------------------------|
| `-m`  | `--manifest-path` | `Path` | (required) | required | The `microcontroller_manifest.yaml` to read. Must exist and be a readable file |
| `-o`  | `--output-path`   | `Path` | (required) | required | The `.yaml` file to write. Need not exist. Parents are created automatically   |

### `axci config show`

| Short | Long            | Type   | Default    | Form     | Effect                                                             |
|-------|-----------------|--------|------------|----------|--------------------------------------------------------------------|
| `-c`  | `--config-path` | `Path` | (required) | required | The extraction config `.yaml` to print. Must exist and be readable |

### `axci process`

| Short | Long                 | Type   | Default    | Form       | Effect                                                                                                                                             |
|-------|----------------------|--------|------------|------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| `-ld` | `--log-directory`    | `Path` | (required) | required   | The ONE DataLogger output directory to search recursively for `.npz` archives. Must exist and be a readable directory                              |
| `-od` | `--output-directory` | `Path` | (required) | required   | The output root. A `microcontroller_data/` subdirectory is created under it and holds every feather file and the tracker                           |
| `-c`  | `--config`           | `Path` | (required) | required   | The extraction config `.yaml`. Must exist and be a readable file                                                                                   |
| `-id` | `--job-id`           | `str`  | `None`     | optional   | Runs ONLY the job whose canonical hexadecimal identifier matches. External single-job dispatch. Suppresses `-s`                                    |
| `-s`  | `--specifier`        | `str`  | `()`       | repeatable | A controller ID to process. Repeat once per controller. When omitted, every controller the config declares is processed. Ignored when `-id` is set |
| `-w`  | `--workers`          | `int`  | `-1`       | optional   | The workers ONE job receives, not a batch width. See the note below                                                                                |
| `-np` | `--no-progress`      | flag   | `False`    | flag       | Suppresses the extraction progress bar. Bars are displayed by default                                                                              |

**Note:** `-c` expands to `--config` here but to `--config-path` on `axci config show`. The short form is safe to quote
to a user for either command. The long forms are not interchangeable.

**Note on `-w`:** the value is a width the job runs at, and the library caps it against nothing. A non-positive value
(the `-1` default) resolves the width from each archive in turn, which yields a single worker below the library's
parallel extraction threshold and `CONTROLLER_EXTRACTION_JOB_CORES`, the declared per-job allocation, at or above it. A
positive value is passed through verbatim to every job, above the declared allocation and above the host's own core
count alike, so a user who names one owns the oversubscription it buys. `-w 1` makes every job sequential. The command's
own `--help` prints the declared allocation as a concrete figure. Read it from there rather than from this skill.

**Note on `-id`:** a job identifier is `CONTROLLER_EXTRACTION_JOB_NAME` hashed together with the controller's
`source_id`, so one controller keeps the same identifier in every recording. A user obtains it from the `job_id` key of
a `prepare_log_processing_batch_tool` job entry, or from the tracker YAML. The identifier is resolved against the
extraction config, not against the archives on disk, so a missing sibling archive cannot mask it.

### `axci mcp`

| Short | Long          | Type     | Default   | Form     | Effect                                                                    |
|-------|---------------|----------|-----------|----------|---------------------------------------------------------------------------|
| `-t`  | `--transport` | `Choice` | `"stdio"` | optional | Transport protocol. Choice of `stdio` and `streamable-http`, nothing else |

**Note:** `run_server()` also accepts a third value, `sse`, but the CLI `Choice` rejects it (exit code 2). Never pass
`-t sse`, and never tell a user to.

---

## Command behavior and failure modes

### `axci id`

Lists valid serial ports, evaluates each in its own worker process, and prints one numbered line per port. Ports whose
USB PID is `None` are filtered out before evaluation, which mainly affects Linux hosts. Each line renders one of three
`evaluate_port` outcomes: `[Microcontroller ID: N]`, `[No microcontroller]`, or `[Connection Failed: ...]`.

| Condition                     | Behavior                                                                                                         |
|-------------------------------|------------------------------------------------------------------------------------------------------------------|
| No valid ports                | Prints "No valid serial ports detected." and exits normally. Not an error                                        |
| One port unreachable          | `evaluate_port` catches every exception and returns an error string, so the sweep over the other ports continues |
| Permission denied on the port | Renders as `[Connection Failed]`. Check `/dev/ttyACM*` group membership                                          |
| Wrong UART baudrate           | Renders as `[No microcontroller]`. Retry at the board's own speed before concluding anything                     |

### `axci mqtt`

Constructs an `MQTTCommunication` client, connects, reports, and disconnects.

| Condition          | Behavior                                                                                                                            |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Broker reachable   | Prints a SUCCESS line naming host and port                                                                                          |
| Broker unreachable | Catches the `ConnectionError` and prints an ERROR line. **The command still exits 0**, so a script cannot branch on its exit status |

**Note:** the unreachable message covers every socket-level failure, including a refused connection, a timeout, and a
hostname that could not be resolved. Have the user verify the host string as well as the broker service.

### `axci config create`

Reads the manifest, emits one controller entry per registered controller with one module entry per registered module,
and writes the result. Every `event_codes` list is **empty** and kernel extraction is left unconfigured.

| Condition                           | Behavior                                                           |
|-------------------------------------|--------------------------------------------------------------------|
| `-m` missing or not a file          | Click rejects the invocation before the command body runs (exit 2) |
| Manifest registers no controllers   | `ValueError` naming the manifest path                              |
| `-o` does not end in `.yaml`/`.yml` | `ValueError` from the YAML writer                                  |
| `-o` parents do not exist           | Created automatically                                              |
| `-o` already exists                 | Overwritten without a prompt                                       |

**Note:** the generated file is a **precursor**, not a usable config. Processing it as written fails inside the job body
with an empty-event-codes error. The user must fill in the event codes, and add a kernel entry if kernel messages are
wanted, before `axci process` can use it. `/extraction-configuration` owns the event-code semantics.

### `axci config show`

Prints the config path, then each controller ID, each module as `(module_type, module_id): events=[...]`, and the kernel
line as either its event codes or `Kernel: not configured`.

| Condition                           | Behavior                                           |
|-------------------------------------|----------------------------------------------------|
| `-c` missing or not a readable file | Click rejects the invocation (exit 2)              |
| `-c` does not end in `.yaml`/`.yml` | `ValueError` from the YAML reader                  |
| `controllers` key present but empty | `ValueError` from `ExtractionConfig.__post_init__` |

**Note:** `show` is a printer, not a validator. It reports an empty `events=[]` without complaint, and it checks nothing
against a manifest. Only `validate_extraction_config_tool` performs the real checks.

### `axci process`

Drives `run_log_processing_pipeline` in `orchestration/pipeline.py`. It resolves the job set with **strict** sourcing,
echoes the resolved controller IDs, opens the tracker, and runs the jobs **one after another**.

| Exception                     | Trigger                                                                                                                                                                                                                                                                                                           |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `FileNotFoundError`           | A requested controller's archive is absent or resolves to more than one file. The log directory resolves no job at all. A nonexistent `-ld` or `-c` never reaches this path, because Click rejects it with exit code 2                                                                                            |
| `ValueError`                  | The tree holds more than one `microcontroller_manifest.yaml`. A manifest registers no controllers. The config declares no controllers. A requested controller is unregistered in the manifest or absent from the config. `-id` matches no configured controller. The resolved archives sit in several directories |
| `ValueError` (at job runtime) | A configured module or the kernel declares empty event codes. A controller declares no extraction target at all. The archive carries no onset timestamp message. A logged message's payload size disagrees with its prototype code                                                                                |
| `OSError`                     | A directory beneath the log directory cannot be read                                                                                                                                                                                                                                                              |
| `TimeoutError`                | The tracker's `.LOCK` file cannot be acquired within the timeout, most often a concurrent MCP batch over the same output directory                                                                                                                                                                                |

**Note:** the first four `ValueError` triggers and the `FileNotFoundError` archive trigger are the strict-sourcing
counterparts of the MCP path's `skipped_sources` reasons. `/log-processing` is authoritative for the lenient model.
Cross-reference the **Lenient sourcing** table in its `references/error-routing.md` when a user's hard CLI failure and
an agent's silent skip describe the same misconfiguration.

**Note:** output layout is identical on both paths, `microcontroller_data/` under `-od`, holding
`controller_{source_id}_module_{type}_{id}.feather`, `controller_{source_id}_kernel.feather`, and
`microcontroller_processing_tracker.yaml`.

### `axci mcp`

| Condition            | Behavior                                                                                      |
|----------------------|-----------------------------------------------------------------------------------------------|
| `-t stdio` (default) | Calls `console.disable()` and prints **nothing**, because the JSON-RPC stream shares stdout   |
| `-t streamable-http` | Echoes `Starting AXCI MCP server with streamable-http transport...`, then blocks serving HTTP |
| Broken dependency    | Traceback instead of the startup line                                                         |

Use `streamable-http` for any hand-launched smoke test, never the `stdio` default.
`/communication-mcp-environment-setup` owns that procedure.

---

## How `axci process` diverges from the MCP path

These four divergences are the whole reason a user's CLI report can describe behavior the MCP workflow cannot reproduce.
Read them before answering "why did it fail for me but not for you".

| Divergence            | CLI behavior                                                                                                                                                                                                                                               | MCP behavior                                                                                                                                     |
|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| Strict sourcing       | `prepare_jobs` runs with its `strict_sources` default, so an unregistered, unconfigured, or unresolvable controller **raises** and no job runs                                                                                                             | `prepare_log_processing_batch_tool` passes `strict_sources=False`, records the same three conditions in `skipped_sources`, and prepares the rest |
| No error recovery     | The job loop carries **no exception handling**. The first job that raises aborts the invocation. The tracker records that one job `FAILED`, and every job behind it stays `SCHEDULED` with no failure record of its own and no requeue                     | The batch engine requeues a broken job, then fails jobs explicitly with a named reason                                                           |
| Empty result is fatal | A log directory resolving no job raises `FileNotFoundError` naming it                                                                                                                                                                                      | The same situation returns `success: True` with `jobs: []` and `source_ids: []`                                                                  |
| No budget sizing      | Resolves each job's width from its own archive at the `-w -1` default and stops there, estimating no memory and weighing nothing against a budget. `memory_mb`, `archive_bytes`, `modeled`, `pool_size`, and `job_allocations` have **no CLI counterpart** | Sizes every job from its archive's zip directory and admits jobs against resolved core and memory budgets                                        |

**Note on `-np`:** the flag reaches only the parallel extraction path. A job that runs sequentially renders no progress
bar whatever the flag says. That covers `-w 1`, an archive the `-w -1` default sizes to a single worker because it sits
below the parallel extraction threshold, and an archive the reader itself batches sequentially because it sits below
the parallel processing threshold. The flag therefore matters mainly for non-interactive runs, where the bar would
otherwise pollute a captured log.

**Note on tracker contention:** both paths lock the same `microcontroller_processing_tracker.yaml` through its `.LOCK`
file, on the CLI side when the pipeline aligns the tracker and again on every job state transition. A user running `axci
process` against a directory the agent is preparing or executing makes whichever side asks second raise a
`TimeoutError`. Ask the user to stop their CLI run before preparing or executing that directory, and never start an MCP
batch over a directory the user has a run in.

---

## Fallback: what to tell a user when MCP is unavailable

Only these four MCP capabilities have a CLI path.

| Blocked MCP tool                                                         | Tell the user to run                            |
|--------------------------------------------------------------------------|-------------------------------------------------|
| `list_microcontrollers_tool`                                             | `axci id -b <baudrate>`                         |
| `check_mqtt_broker_tool`                                                 | `axci mqtt -h <host> -p <port>`                 |
| `read_extraction_config_tool`                                            | `axci config show -c <config>`                  |
| `prepare_log_processing_batch_tool` + `execute_log_processing_jobs_tool` | `axci process -ld <logs> -od <out> -c <config>` |

Two caveats on the last row. `axci process` handles ONE log directory per invocation, so a batch spanning several log
directories becomes one invocation per directory. It also demands a finished extraction configuration. With the server
down, the user generates the precursor with `axci config create -m <manifest> -o <config>` and fills in the event codes
by hand.

Everything else genuinely blocks until the server is back: manifest read and write, archive assembly, recording
discovery, extraction config write and validate, every batch status, timing, cancel, and reset tool, and every output
verification, query, and cleanup tool. Say so plainly rather than improvising a substitute.

---

## Related skills

| Skill                                  | Relationship                                                                         |
|----------------------------------------|--------------------------------------------------------------------------------------|
| `/log-processing`                      | Owns the MCP batch workflow and the authoritative lenient sourcing model             |
| `/extraction-configuration`            | Owns config authoring, validation, and event-code semantics behind `axci config`     |
| `/microcontroller-setup`               | Owns manifests, archive assembly, and the discovery behind `axci id` and `axci mqtt` |
| `/communication-mcp-environment-setup` | Owns the `--help` exemption, the hand-launch smoke test, and MCP recovery            |
| `/log-processing-results`              | Downstream: the feather output both paths write                                      |
| `/log-input-format`                    | Reference: the assembled archives `axci process` reads under `-ld`                   |
| `/microcontroller-interface`           | Context: the PC-side recording code, which no `axci` command replaces                |
| `/microcontroller:firmware-module`     | Reference: the serial speed contract behind the `axci id` baudrate                   |
| `/pipeline`                            | Context: where each CLI command sits in the end-to-end pipeline                      |

---

## Verification checklist

```text
Answering a CLI question:
- [ ] Answered from this skill or from `axci COMMAND --help`, never from memory
- [ ] Quoted the long option form and never presented `-h` as a help alias
- [ ] Named the MCP equivalent alongside any command the agent could have run itself
- [ ] Invoked no `axci` command other than `--help`

Handing a user a CLI command:
- [ ] Confirmed MCP is genuinely unavailable via `/communication-mcp-environment-setup` first
- [ ] Printed the command for the user instead of running it
- [ ] Warned about strict sourcing and the abort-on-first-failure loop before recommending `axci process`
- [ ] Confirmed no MCP batch is holding the tracker for that output directory
```
