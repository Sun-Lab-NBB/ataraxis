---
name: log-processing
description: >-
  Orchestrates batch log processing via the ataraxis-communication-interface MCP server: archive discovery,
  batch preparation, job execution, progress monitoring, cancellation, and error recovery. Use when processing
  microcontroller log archives, extracting hardware module and kernel data, or managing batch processing jobs.
user-invocable: true
---

# Log processing

Orchestrates the batch log processing workflow: discover log archives, prepare execution manifests, dispatch
data extraction jobs, monitor progress, and hand off to downstream skills for output verification and analysis.

---

## Scope

**Covers:**
- Archive discovery and recording hierarchy resolution
- Batch preparation and execution manifest creation
- Job execution with resource allocation
- Progress monitoring and timing
- Cancellation and cleanup
- Failed job reset and retry
- Batch status overview across directories
- Status formatting and presentation

**Does not cover:**
- Input data format, archive structure, or source ID semantics (see `/log-input-format`)
- Output data formats, feather file schema, or event analysis (see `/log-processing-results`)
- Extraction configuration management (see `/extraction-configuration`)
- Microcontroller hardware setup or discovery (see `/microcontroller-setup`)
- MCP server connectivity issues (see `/communication-mcp-environment-setup`)

**Handoff rules:** If the user asks about archive format, source IDs, or DataLogger output, invoke
`/log-input-format`. If the user asks about feather file contents, event distribution, or timing analysis,
invoke `/log-processing-results`. If the user needs to create or edit an extraction config, invoke
`/extraction-configuration`. If MCP tools are unavailable, invoke `/communication-mcp-environment-setup`.

---

## Agent requirements

You MUST use the ataraxis-communication-interface MCP tools for all processing operations. Do not import
log processing Python functions directly or run processing via CLI commands. If MCP tools are not available,
invoke `/communication-mcp-environment-setup` to diagnose and resolve connectivity issues.

You MUST run `discover_microcontroller_data_tool` before calling `prepare_log_processing_batch_tool` to
obtain confirmed log directory paths and source IDs. Do not assume or guess directory paths or source IDs.

You MUST ask the user for output directory paths before preparing the batch — there is no default, and
output directories are required for every log directory being processed.

You MUST have a validated extraction config path before processing. Use `/extraction-configuration` to
create and validate the config if one does not exist.

---

## Available tools

### Discovery tools

| Tool                                 | Purpose                                                                        |
|--------------------------------------|--------------------------------------------------------------------------------|
| `discover_microcontroller_data_tool` | Searches for manifests, locates archives, and returns confirmed source entries |

Uses **manifest-based routing**: recursively searches for `microcontroller_manifest.yaml` files under the
root directory. Each manifest identifies a DataLogger output directory containing controller log archives.
Only sources whose log archives exist on disk are included.

**Parameters:**

| Parameter        | Type  | Default    | Description                                           |
|------------------|-------|------------|-------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to the root directory to search         |

**Return structure:**
```text
sources[]:              Flat list of confirmed source entries:
  recording_root:       Path to the recording root directory
  source_id:            Source ID string (controller ID)
  name:                 Controller name from manifest
  log_archive:          Absolute path to the .npz archive
  log_directory:        Absolute path to the DataLogger output directory
  modules[]:            Module entries from manifest
log_directories:        Flat list of log directory paths (pass directly to prepare tool)
total_sources:          Number of confirmed source entries
total_log_directories:  Number of log directories with archives
```

**Important:** This tool requires `microcontroller_manifest.yaml` files to exist in DataLogger output
directories. For legacy sessions without manifests, use `write_microcontroller_manifest_tool` (see
`/microcontroller-setup`) to retroactively tag log directories before running discovery.

### Preparation and execution tools

| Tool                                    | Purpose                                                             |
|-----------------------------------------|---------------------------------------------------------------------|
| `prepare_log_processing_batch_tool`     | Creates execution manifest without starting execution (idempotent)  |
| `execute_log_processing_jobs_tool`      | Dispatches prepared jobs for background execution                   |

**`prepare_log_processing_batch_tool` parameters:**

| Parameter            | Type        | Default    | Description                                                                   |
|----------------------|-------------|------------|-------------------------------------------------------------------------------|
| `log_directories`    | `list[str]` | (required) | Absolute paths to DataLogger output directories. **Ask user.**                |
| `source_ids`         | `list[str]` | (required) | Confirmed source IDs from `discover_microcontroller_data_tool`.               |
| `output_directories` | `list[str]` | (required) | Absolute paths for per-directory output. Must match log_directories length.   |
| `config_path`        | `str`       | (required) | Absolute path to the validated ExtractionConfig YAML file.                    |

**`execute_log_processing_jobs_tool` parameters:**

| Parameter       | Type         | Default    | Description                                                                                                           |
|-----------------|--------------|------------|-----------------------------------------------------------------------------------------------------------------------|
| `jobs`          | `list[dict]` | (required) | Job descriptors from prepare manifest (log_directory, output_directory, tracker_path, job_id, source_id, config_path) |
| `worker_budget` | `int`        | `-1`       | Total CPU cores for the session; -1 for automatic resolution. Controls memory footprint.                              |

### Monitoring and management tools

| Tool                                | Purpose                                                               |
|-------------------------------------|-----------------------------------------------------------------------|
| `get_log_processing_status_tool`    | Per-job status of active execution session                            |
| `get_log_processing_timing_tool`    | Per-job timing and session-level throughput                           |
| `cancel_log_processing_tool`        | Cancels active session, clears pending queue                          |
| `reset_log_processing_jobs_tool`    | Resets specific or all jobs to SCHEDULED for retry                    |
| `get_batch_status_overview_tool`    | Aggregate status across all log directories under root                |
| `clean_log_processing_output_tool`  | Deletes `microcontroller_data/` subdirectories for re-processing      |

**`reset_log_processing_jobs_tool` parameters:**

| Parameter      | Type               | Default    | Description                                         |
|----------------|--------------------|------------|-----------------------------------------------------|
| `tracker_path` | `str`              | (required) | Absolute path to ProcessingTracker YAML file        |
| `source_ids`   | `list[str] / None` | `None`     | Source IDs to reset; if omitted, all jobs are reset |

**`get_batch_status_overview_tool` parameters:**

| Parameter        | Type  | Default    | Description                                            |
|------------------|-------|------------|--------------------------------------------------------|
| `root_directory` | `str` | (required) | Absolute path to root directory to search for trackers |

**`clean_log_processing_output_tool` parameters:**

| Parameter            | Type        | Default    | Description                                                                          |
|----------------------|-------------|------------|--------------------------------------------------------------------------------------|
| `output_directories` | `list[str]` | (required) | Absolute paths to output directories containing `microcontroller_data/` to delete    |

Deletes the `microcontroller_data/` subdirectory and all contents (feather files, tracker) under each
output directory. After cleanup, the output directories can be passed to
`prepare_log_processing_batch_tool` to reinitialize from scratch.

---

## Pipeline architecture

Multi-target data extraction pipeline:

```text
.npz log archives + extraction_config.yaml → execute_job → Polars DataFrames → .feather IPC files
```

Key architectural facts:
- **ProcessingTracker** manages job lifecycle: `SCHEDULED` → `RUNNING` → `SUCCEEDED` / `FAILED` via YAML state files
- **Single execution session** constraint: only one batch execution can run at a time
- **Parallel processing** activates automatically for archives with >=2000 messages
- **ExtractionConfig** controls which modules, kernel messages, and event codes are extracted per controller
- **Output layout:** All processing output is written under a `microcontroller_data/` subdirectory within the
  output directory provided by the user
- **Output naming:** `controller_{source_id}_module_{type}_{id}.feather` for module data,
  `controller_{source_id}_kernel.feather` for kernel data
- **Tracker filename:** `microcontroller_processing_tracker.yaml`
- **Cleanup:** Use `clean_log_processing_output_tool` to delete the `microcontroller_data/` subdirectory and
  reinitialize processing from scratch

---

## Processing workflow

### Execution model

The processing workflow uses a **prepare-then-execute** model:

1. **Prepare** creates an execution manifest (tracker files, job lists) without starting any computation.
   This step is idempotent — calling it again on the same directories returns the existing manifest with
   current job statuses.

2. **Execute** dispatches jobs from the manifest with resource allocation and background thread management.
   Only one execution session can be active at a time.

### Pre-processing checklist

```text
- [ ] Archives discovered or log directory paths provided
- [ ] Log directories confirmed with user
- [ ] Output directories provided by user (required, no default)
- [ ] Extraction config validated (invoke `/extraction-configuration` if needed)
- [ ] Resource allocation confirmed with user (workers, parallelism)
```

**STOP**: If any checkbox is incomplete, do not proceed. Complete the missing steps first.

### Workflow steps

1. **Discover archives** — Call `discover_microcontroller_data_tool` with the user-provided root directory.

2. **Present discovery results** — Show the discovered sources, source IDs, controller names, and module
   listings. Format the discovery data as a readable summary so the user can see what was found.

3. **Confirm directories to process** — Ask the user which log directories to process. Accept all
   discovered directories or a user-selected subset. MUST confirm before proceeding.

4. **Confirm output directories** — Ask the user for the output directory paths (one per log directory).
   There is no default — output directories must be explicitly provided. MUST confirm before proceeding.

5. **Validate extraction config** — Ensure the user has a validated extraction config. If not, invoke
   `/extraction-configuration` to create one. The config path is required for batch preparation.

6. **Prepare batch** — Call `prepare_log_processing_batch_tool` with the confirmed log directories,
   source IDs, output directories, and config path.

7. **Confirm resource allocation** — Present the default worker budget (worker_budget=-1 for
   automatic resolution to available CPU cores) and ask if the user wants to override. Explain that
   the budget controls memory footprint and the system allocates workers per job automatically
   based on archive size.

8. **Execute jobs** — Call `execute_log_processing_jobs_tool` with the job descriptors from the prepare
   manifest and confirmed resource settings.

9. **Monitor progress** — Use `get_log_processing_status_tool` to check per-job progress. Optionally use
   `get_log_processing_timing_tool` for elapsed time and throughput metrics. Present status as a
   formatted table (see Status Formatting section).

10. **Handle completion** — When all jobs finish, check for failures. On success, invoke
    `/log-processing-results` to discover and analyze the output. On failure, see Error Routing section.

---

## Resource management

The execution tool uses **budget-based worker allocation** with a single `worker_budget` parameter that
directly controls memory footprint (each worker spawns a separate process). Before dispatching, the tool
probes each archive's message count and allocates workers using tier-based scaling:

The system uses two layers of allocation:

1. **Per-job tier assignment** — each job is assigned a worker count based on its archive's message count
   using a sqrt-derived formula (`ceil(sqrt(messages / 1,000))`), rounded to the nearest multiple of 5.
   This determines the per-job worker tier:

| Archive Size  | Worker Tier      | Typical Scenario               |
|---------------|------------------|--------------------------------|
| < 2,000 msgs  | 1 (sequential)   | Short recording                |
| 10,000 msgs   | 5                | Brief session                  |
| 50,000 msgs   | 10               | Medium session                 |
| 250,000 msgs  | 15               | Extended session               |
| 648,000 msgs  | 25               | Long continuous recording      |

2. **Budget-limited concurrency** — the available budget limits how many jobs can run concurrently, not
   the per-job worker count. Jobs are grouped by their tier and dispatched in groups that fit within the
   remaining budget. Two cores are reserved for system operations.

When `worker_budget=-1`, the system resolves the total using the host machine's available CPU cores
via `resolve_worker_count`. Reduce `worker_budget` to limit memory footprint on constrained systems.

---

## Status formatting

When presenting batch status to the user, format as a table:

```text
**Log Processing Status**

Summary: 5/8 jobs complete | 1 running | 2 queued | 0 failed

| Source ID | Status    | Duration |
|-----------|-----------|----------|
| 101       | SUCCEEDED | 12.5s    |
| 102       | SUCCEEDED | 8.3s     |
| 103       | RUNNING   | 6.1s     |
| 104       | SCHEDULED | --       |
```

When using `get_batch_status_overview_tool` for multi-directory status:

```text
**Batch Overview**

| Log Directory         | Status    | Succeeded | Failed | Total |
|-----------------------|-----------|-----------|--------|-------|
| /data/session1/logs/  | completed | 3         | 0      | 3     |
| /data/session2/logs/  | failed    | 1         | 1      | 2     |
```

---

## Re-running failed jobs

1. Identify failed jobs from `get_log_processing_status_tool` output (check `error_message` field)
2. Call `reset_log_processing_jobs_tool` with the tracker path and failed source IDs
3. Re-prepare or re-execute the reset jobs using the same workflow

To re-process an entire directory from scratch, call `clean_log_processing_output_tool` to delete the
`microcontroller_data/` subdirectory, then re-prepare and re-execute.

---

## Error routing

### Preparation errors

| Error                      | Resolution                                                        |
|----------------------------|-------------------------------------------------------------------|
| "Directory does not exist" | Verify path exists                                                |
| "Path is not a directory"  | Verify path is a directory, not a file                            |
| "Length mismatch"          | Ensure output_directories matches log_directories length          |
| "Permission denied"        | Check filesystem permissions                                      |
| "Config file not found"    | Verify extraction config path; use `/extraction-configuration`    |

### Execution errors

| Error                                    | Resolution                                          |
|------------------------------------------|-----------------------------------------------------|
| "An execution session is already active" | Wait for current session or cancel first            |
| "No valid jobs to execute"               | Verify job descriptors have all required keys       |
| "Tracker file not found"                 | Re-prepare the batch to regenerate tracker files    |

### Processing failure routing

| Error Pattern                        | Action                                                        |
|--------------------------------------|---------------------------------------------------------------|
| Archive not found / file read errors | Verify .npz archives exist in log directory                   |
| Invalid extraction config            | Validate config via `/extraction-configuration`               |
| MCP tools unavailable                | Invoke `/communication-mcp-environment-setup`                               |
| Out of memory                        | Reduce `worker_budget`                                        |
| Corrupt tracker or partial output    | Call `clean_log_processing_output_tool`, then re-prepare      |

---

## Related skills

| Skill                        | Role                                                             |
|------------------------------|------------------------------------------------------------------|
| `/communication-mcp-environment-setup`     | Prerequisite: MCP server connectivity                            |
| `/microcontroller-setup`     | Upstream: hardware discovery and manifest management             |
| `/microcontroller-interface` | Upstream: code that produces the log data being processed        |
| `/extraction-configuration`  | Upstream: extraction config that controls what data is extracted |
| `/log-input-format`          | Reference: input archive format and source ID semantics          |
| `/log-processing-results`    | Downstream: output data discovery and event analysis             |
| `/pipeline`                  | Context: log processing is phase 5 of the end-to-end pipeline    |

---

## Verification checklist

```text
Log Processing Workflow:
- [ ] MCP server connected (if not, invoke `/communication-mcp-environment-setup`)
- [ ] Archives discovered via `discover_microcontroller_data_tool`
- [ ] Log directories confirmed with user
- [ ] Output directories confirmed with user
- [ ] Extraction config validated via `/extraction-configuration`
- [ ] Batch prepared via `prepare_log_processing_batch_tool`
- [ ] Resource allocation confirmed with user
- [ ] Jobs executed via `execute_log_processing_jobs_tool`
- [ ] Status monitored until all jobs complete or fail
- [ ] Failed jobs investigated and retried if needed (use `clean_log_processing_output_tool` for full reset)
- [ ] Successful output verified via `/log-processing-results`
```
