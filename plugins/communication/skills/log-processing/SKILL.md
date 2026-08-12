---
name: log-processing
description: >-
  Orchestrates batch log processing via the ataraxis-communication-interface MCP server: archive discovery,
  batch preparation, job execution, progress monitoring, cancellation, and error recovery. Use when processing
  microcontroller log archives, extracting hardware module and kernel data, or managing batch processing jobs.
user-invocable: false
---

# Log processing

Runs the prepare-then-execute batch workflow that turns microcontroller log archives into extracted feather output.

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

---

## Agent requirements

You MUST use the ataraxis-communication-interface MCP tools for all processing operations. Do not import log processing
Python functions directly or run processing via CLI commands. If MCP tools are not available, invoke
`/communication-mcp-environment-setup` to diagnose and resolve connectivity issues. The `axci` CLI is the human path,
and `/cli-reference` owns it, including how `axci process`, which takes ONE log directory per invocation, diverges from
this batch workflow.

You MUST run `discover_microcontroller_data_tool` before calling `prepare_log_processing_batch_tool` to obtain confirmed
log directory paths and source IDs. Do not assume or guess directory paths or source IDs.

You MUST ask the user for one output directory path per log directory before preparing the batch, since there is no
default. You MUST also have a validated extraction config path, created and validated via `/extraction-configuration`.

---

## Available tools

### Discovery tools

`discover_microcontroller_data_tool` takes one required parameter, `root_directory`, the absolute path to search. It
recursively locates every `microcontroller_manifest.yaml` under that root, each of which tags one DataLogger output
directory holding controller log archives, and returns only the sources whose log archives exist on disk.

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

**Important:** Tag a legacy log directory that carries no `microcontroller_manifest.yaml` with
`write_microcontroller_manifest_tool` (see `/microcontroller-setup`) before running discovery over it.

### Preparation and execution tools

| Tool                                | Purpose                                                            |
|-------------------------------------|--------------------------------------------------------------------|
| `prepare_log_processing_batch_tool` | Creates execution manifest without starting execution (idempotent) |
| `execute_log_processing_jobs_tool`  | Dispatches prepared jobs for background execution                  |

**`prepare_log_processing_batch_tool` parameters:**

| Parameter            | Type        | Default    | Description                                                                          |
|----------------------|-------------|------------|--------------------------------------------------------------------------------------|
| `log_directories`    | `list[str]` | (required) | Absolute paths to DataLogger output directories. **Ask user.**                       |
| `source_ids`         | `list[str]` | (required) | Confirmed source IDs from `discover_microcontroller_data_tool`. Strings, never ints. |
| `output_directories` | `list[str]` | (required) | Absolute paths for per-directory output. Must match log_directories length.          |
| `config_path`        | `str`       | (required) | Absolute path to the validated ExtractionConfig YAML file.                           |

**Note:** Every ordering the library returns sorts source identifiers **as strings**, so `"10"` precedes `"2"`. A report
that needs numeric order must sort the identifiers itself rather than presenting the returned order.

**Note:** `source_ids` entries are strings and are compared by exact string equality. Pass the `source_id` values
`discover_microcontroller_data_tool` returned verbatim. The manifest tools report the same controller as an integer
`id`, so an ID copied from there must be converted with `str()` first (see `/log-input-format`).

**Note:** An empty `source_ids` list is a shorthand for "every controller the extraction config declares", not for "no
controllers". It is also a filter footgun: one `source_ids` list is applied uniformly to every entry of
`log_directories`, so a list built from one recording becomes the request for all of them.

**Note:** A requested controller that yields no job is dropped into that directory's `skipped_sources` rather than
raising, so a directory with no matching archives returns `jobs: []` and `source_ids: []` while still reporting
`success: True`. See [error-routing.md](references/error-routing.md) for each skip reason and its remedy.

**Return structure:**
```text
success:                Boolean flag. False only when `log_directories` is empty AND at least one directory
                        raised; a run whose every path was merely invalid still reports true with no jobs
log_directories{}:      Dict keyed by the input path string, one entry per directory that resolved without
                        raising, including a directory that produced no job at all:
  tracker_path:         Absolute path to that directory's ProcessingTracker YAML file
  output_directory:     Absolute path to the `microcontroller_data/` subdirectory the jobs write into
  source_ids[]:         Controller IDs that produced a job, in the same order as `jobs`
  jobs[]:               Job descriptors, sorted heaviest estimated memory first. Each entry carries the twelve
                        execute keys plus `memory_mb`, `status`, and `error_message` when the tracker holds one
  summary{}:            Tracker counts: `total`, `succeeded`, `failed`, `running`, `scheduled`
  skipped_sources[]:    Requested IDs that produced no job, each `{source_id, reason}` (see Error routing)
total_log_directories:  Number of entries in `log_directories`, which is not the number that produced jobs
total_jobs:             Number of job descriptors across every entry
invalid_paths[]:        Input paths that are not directories (present only if any)
failed_directories[]:   Directories whose preparation raised, each `{log_directory, error}` (present only if any)
```

**Note:** `summary` counts every job the tracker on disk holds, while `jobs` lists only what this call prepared. A
directory prepared for a subset of its controllers therefore reports a `summary` total larger than its `jobs` length.
Decide what still needs running from each job's own `status`, not from the summary.

**Note:** Re-preparing deletes every tracker entry whose controller the current `microcontroller_manifest.yaml` no
longer registers, along with that entry's recorded outcome, so regenerating or editing a manifest erases the
completed-job history of every controller it dropped. A preparation that resolves no job writes nothing at all, creating
neither the output subdirectory nor the tracker.

**`execute_log_processing_jobs_tool` parameters:**

| Parameter          | Type         | Default    | Description                                                                                     |
|--------------------|--------------|------------|-------------------------------------------------------------------------------------------------|
| `jobs`             | `list[dict]` | (required) | Job descriptors from the execution manifest. Pass each dict through unchanged (see note below). |
| `core_budget`      | `int`        | `-1`       | Cores the session may commit. -1 auto-resolves from the host, keeping a reserve.                |
| `memory_budget_mb` | `int`        | `-1`       | Memory in MB the session may commit. -1 auto-resolves from host physical memory.                |

**Note:** Every job descriptor must carry all twelve keys the preparation emitted, `log_directory`, `archive_path`,
`output_directory`, `config_path`, `tracker_path`, `job_name`, `job_id`, `source_id`, `core_weight`, `message_count`,
`archive_bytes`, and `modeled`. A hand-assembled descriptor that drops the sizing keys is rejected into `invalid_jobs`
with "Missing or unreadable sizing keys from the prepared manifest."

**Return structure:**
```text
started:            Boolean flag, true when the session was claimed and the manager thread started
total_jobs:         Valid job descriptors the session accepted; admission against the budgets happens later
core_budget:        Cores the session may commit across all concurrently running jobs
memory_budget_mb:   Memory in MB the session may commit across all concurrently running jobs
pool_size:          Job slots the shared pool opened
job_allocations[]:  Per-job sizing the session resolved:
  job_id:           Canonical hexadecimal job identifier
  source_id:        Controller ID the job reads
  cores:            Cores this job received
  memory_mb:        Estimated memory this job holds
  message_count:    Messages the archive holds
  modeled:          False when the figures fell back to the baseline rather than being read
invalid_jobs[]:     Descriptors rejected before dispatch, each carrying an `error` key (present only if any)
```

When no descriptor survives validation the tool returns an error dictionary carrying `invalid_jobs` instead, and
`started` is absent.

### Monitoring and management tools

| Tool                               | Purpose                                                          |
|------------------------------------|------------------------------------------------------------------|
| `get_log_processing_status_tool`   | Per-job status of active execution session                       |
| `get_log_processing_timing_tool`   | Per-job timing and session-level throughput                      |
| `cancel_log_processing_tool`       | Cancels active session, clears pending queue                     |
| `reset_log_processing_jobs_tool`   | Resets specific or all jobs to SCHEDULED for retry               |
| `get_batch_status_overview_tool`   | Aggregate status across all log directories under root           |
| `clean_log_processing_output_tool` | Deletes `microcontroller_data/` subdirectories for re-processing |

**Note:** `get_log_processing_status_tool`, `get_log_processing_timing_tool`, and `cancel_log_processing_tool` report
only on the single active in-memory execution session (the most recent `execute_log_processing_jobs_tool` call) and
return a no-active-session response otherwise. That covers a call made before execute, and a call made after a server
restart, even where trackers with running jobs exist on disk. Status and timing return `'No execution session exists.'`,
and cancel returns `{canceled: False}` with `'No execution session is active.'`. For status of directories not in the
active session, use `get_batch_status_overview_tool`, which reads trackers from disk.

**Completion test:** a finished session reads `active: False` **with** a populated `jobs` list, while a server that
never held one reads `active: False` with a `message` and no `jobs` key at all. Branch on the presence of `jobs`, not on
`active` alone.

**`get_log_processing_status_tool` return structure:**
```text
active:           True while the session's manager thread is alive
canceled:         True once `cancel_log_processing_tool` cleared this session's pending queue
jobs[]:           One entry per job the session tracks, read from the trackers on disk:
  job_id:         Canonical hexadecimal job identifier
  source_id:      Controller ID the job reads
  status:         "SCHEDULED", "RUNNING", "SUCCEEDED", "FAILED", or "UNKNOWN"
  error_message:  Present only when the tracker recorded one for that job
  executor_id:    Present only when the tracker recorded one
summary:          Session counts: `total`, `succeeded`, `failed`, `running`, `scheduled`
```

**Note:** `total` counts every job the session accepted, while the four status counts cover only the jobs whose tracker
entry resolved. A session holding `UNKNOWN` entries therefore sums below its own `total`, which is the shape an
unreadable or missing tracker takes.

**`get_log_processing_timing_tool` return structure:**
```text
active:                     True while the session's manager thread is alive
jobs[]:                     One entry per job the session's trackers record:
  job_id, source_id:        Job identity
  executor_id:              Executor that ran the job, when the tracker recorded one
  started_at:               Microsecond-precision UTC epoch the job started
  elapsed_seconds:          Present only while the job is RUNNING
  completed_at:             Microsecond-precision UTC epoch the job finished
  duration_seconds:         Present once the job carries both a start and a completion
session:                    Session-level rollup:
  total_elapsed_seconds:    Seconds since the earliest job start in the session
  completed_count:          Jobs that succeeded
  failed_count:             Jobs that failed
  running_count:            Jobs currently reporting an elapsed time
  pending_count:            Session jobs not yet counted as completed, failed, or running
  throughput_jobs_per_hour: Present once at least one job has completed
```

This tool, not the status tool, is the only source of the per-job Duration column in the Status formatting section. It
also supplies the **stall test**: a job whose `elapsed_seconds` keeps rising while `running_count` and `pending_count`
never change is stuck, whereas a batch that is merely slow moves `pending_count` down over time.

**`cancel_log_processing_tool` return structure:**
```text
canceled:                  True when a live session was canceled, False when none was active
message:                   Names the pending jobs cleared and the jobs still completing
final_state:               Present only when a session was canceled:
  succeeded_jobs:          This session's jobs that had already succeeded
  failed_jobs:             This session's jobs that had already failed
  active_jobs_at_cancel:   Jobs still running when the pending queue was cleared
```

**Note:** Cancellation clears the pending queue and stops new admissions. Jobs already running are left to finish. The
session therefore stays alive while they drain, and an `execute_log_processing_jobs_tool` call made during the drain
still returns "An execution session is already active". Poll `get_log_processing_status_tool` until `active` reads
`False` before executing or resetting anything. Cleared queued jobs stay `SCHEDULED` and are re-runnable directly, they
need no reset.

**`reset_log_processing_jobs_tool` parameters:**

| Parameter      | Type               | Default    | Description                                         |
|----------------|--------------------|------------|-----------------------------------------------------|
| `tracker_path` | `str`              | (required) | Absolute path to ProcessingTracker YAML file        |
| `source_ids`   | `list[str] / None` | `None`     | Source IDs to reset, if omitted, all jobs are reset |

**Note:** `source_ids` are matched against each tracker entry's specifier by exact string equality, so `"10"` does not
select the job of controller `101`, and omitting them is the only way to reset every job in the tracker.

`get_batch_status_overview_tool` takes one required parameter, `root_directory`, the absolute path under which trackers
are searched for. `clean_log_processing_output_tool` takes one required parameter, `output_directories`, the absolute
paths whose `microcontroller_data/` subdirectory and every file in it, feather files and tracker alike, are deleted.

**Return structure:**
```text
results[]:            One entry per requested directory:
  output_directory:   The requested path, echoed back
  cleaned:            Boolean flag, true both for a real deletion and for a directory that had nothing to delete
  data_path:          The deleted `microcontroller_data/` path; present on a real deletion and on a failed one
  message:            "Nothing to clean." when the directory held no `microcontroller_data/` subdirectory
  error:              "Directory does not exist.", "Path is not a directory.", or "Unable to delete: ..."
total_cleaned:        Entries whose `cleaned` flag reads true
total_directories:    Entries returned
```

**Note:** `cleaned: True` does not prove that anything was deleted, since a directory that held no
`microcontroller_data/` reports it alongside "Nothing to clean". `total_cleaned` therefore cannot confirm a real reset.
Require `data_path` on the entry to confirm one, and treat a `message` entry as a sign the output directory was not the
one processing wrote into.

---

## Pipeline architecture

```text
.npz log archives + extraction_config.yaml → execute_job → Polars DataFrames → .feather IPC files
```

- **ProcessingTracker** manages job lifecycle: `SCHEDULED` → `RUNNING` → `SUCCEEDED` / `FAILED` via YAML state files
- **Job identity:** every job is registered under the tracker job name `CONTROLLER_EXTRACTION_JOB_NAME`, and its
  `job_id` is that name hashed together with the `source_id`. One controller therefore keeps the same `job_id` in every
  recording, and a job is only unique batch-wide as the `(tracker_path, job_id)` pair
- **Single execution session** constraint: only one batch execution can run at a time
- **Process model:** every job body runs in a worker of one shared, spawn-context process pool, and a job admitted at
  more than one core opens its own extraction pool of that width inside its worker. A spawned child inherits nothing and
  re-imports the whole package, which is the cost the sizing model charges as `SPAWNED_CHILD_MEMORY_MB` for each
  process. Every worker is pinned to a single numeric-backend thread, so no worker oversubscribes the cores its job was
  admitted at
- **Parallel processing** activates automatically once an archive exceeds the library's parallel-processing threshold.
  The width a job actually receives is reported as its `core_weight` (prepare) and `cores` (execute)
- **Empty archives:** an archive with zero data messages completes as `SUCCEEDED` and produces no feather files. This is
  expected, not a failure to retry or clean
- **ExtractionConfig** controls which modules, kernel messages, and event codes are extracted per controller

`/log-processing-results` owns the output layout under `microcontroller_data/`, the feather file naming, and the tracker
filename.

---

## Processing workflow

### Execution model

The processing workflow uses a **prepare-then-execute** model:

1. **Prepare** creates an execution manifest (tracker files, job lists) without starting any computation. Calling it
   again on the same directories returns the existing execution manifest with current job statuses. See the idempotence
   note under `prepare_log_processing_batch_tool` for the one case in which re-preparing discards recorded outcomes.

2. **Execute** dispatches jobs from the execution manifest with resource allocation and background thread management.
   Only one execution session can be active at a time.

### Workflow steps

0. **Orient before starting**: Whenever the root may already have been processed (the user says "resume", "finish", or
   "re-run", or names an existing project root), call `get_batch_status_overview_tool` on that root first. It answers
   which directories are `not_started`, in flight, or `failed` before anything is prepared, so a resumed batch
   re-prepares only what it has to. A root reporting no tracker at all has not been processed, so step 1 starts it.

1. **Discover archives**: Call `discover_microcontroller_data_tool` with the user-provided root directory.

2. **Present discovery results**: Render one table row per entry of `sources[]`, carrying its `source_id`, `name`,
   `recording_root`, and module count, and close with the returned `total_sources` and `total_log_directories`.

3. **Confirm directories to process**: Ask the user which log directories to process. Accept all discovered directories
   or a user-selected subset. MUST confirm before proceeding.

4. **Confirm output directories**: Ask the user for the output directory paths (one per log directory). There is no
   default, output directories must be explicitly provided. MUST confirm before proceeding.

5. **Validate extraction config**: Ensure the user has a validated extraction config. If not, invoke
   `/extraction-configuration` to create one. The config path is required for batch preparation.

6. **Prepare batch**: Call `prepare_log_processing_batch_tool` with the confirmed log directories, source IDs, output
   directories, and config path. Then verify the returned `source_ids` against the set you requested, read
   `skipped_sources` for every difference (see Error routing), and check `modeled` on each job entry. A job carrying
   `modeled: false` was sized from a floor because its archive could not be read, so treat it as unsized and expect it
   to fail at extraction for the same reason.

7. **Confirm resource allocation**: Read the per-job `core_weight` and `memory_mb` the preparation stamped onto each job
   entry, and present those figures alongside the default budgets (`core_budget=-1` and `memory_budget_mb=-1`, both
   auto-resolving from the host). Ask whether the user wants to override, and quote what an explicit budget costs from
   Resource management. Never estimate the cost yourself, the preparation already sized every job from its archive.

8. **Execute jobs**: Call `execute_log_processing_jobs_tool` with the job descriptors from the execution manifest and
   confirmed resource settings. Read the returned `core_budget`, `memory_budget_mb`, `pool_size`, and
   `job_allocations[]` back to the user: those are the figures this session actually committed, and they can differ from
   the `core_weight` and `memory_mb` the preparation reported.

9. **Monitor progress**: Use `get_log_processing_status_tool` to check per-job progress. Optionally use
   `get_log_processing_timing_tool` for elapsed time and throughput metrics. Present status as a formatted table (see
   Status formatting section). The pool warms every slot before it admits the first job, so the first status read after
   execute normally shows every job `SCHEDULED`. That is not a stall. The manager then re-examines the running set on a
   fixed poll interval, so re-read after a few seconds rather than immediately.

10. **Handle completion**: The session is over once a status read passes the Completion test. Check its `jobs` list for
    failures. On success, invoke `/log-processing-results` to discover and analyze the output. On failure, see the Error
    routing section.

---

## Resource management

Every job is sized by the library, never by the agent. Read `core_weight` and `memory_mb` from the job entries the
preparation returns, and `core_budget`, `memory_budget_mb`, `pool_size`, and `job_allocations[]` from the execute
return, rather than computing a figure of your own. See [resource-management.md](references/resource-management.md) for
the assets that report each figure, and for how the model widens, admits, and refuses jobs.

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

A job reporting `UNKNOWN` is resolved by re-preparing, not by reset, which is for `FAILED` jobs. Status and timing
entries also carry an optional `executor_id`, which the tracker records as `pid:<process id>` for a job run by the local
pool. Use it to separate two failure shapes: several failed jobs sharing one `executor_id` mean a single worker keeps
dying, while failures spread across many distinct values mean the pool itself is being killed. The second shape routes
to the batch abandonment messages in Error routing.

When using `get_batch_status_overview_tool` for multi-directory status:

```text
**Batch Overview**

| Log Directory         | Status    | Succeeded | Failed | Total |
|-----------------------|-----------|-----------|--------|-------|
| /data/session1/logs/  | completed | 3         | 0      | 3     |
| /data/session2/logs/  | failed    | 1         | 1      | 2     |
```

Each entry's `status` is one of six labels. The first five are resolved from that tracker's own counts by a fixed
priority, so they are not mutually exclusive descriptions of the directory:

| Label         | Meaning                                                                                             |
|---------------|-----------------------------------------------------------------------------------------------------|
| `failed`      | At least one job failed. Outranks every other label, so the directory may still hold succeeded jobs |
| `completed`   | Every job in the tracker succeeded                                                                  |
| `processing`  | At least one job is currently running                                                               |
| `not_started` | Every job is still scheduled                                                                        |
| `in_progress` | Mixed outcomes with nothing running and nothing failed. Also a tracker holding no jobs at all       |
| `error`       | The tracker itself could not be read. The entry carries an `error` key instead of job details       |

Never report `failed` as "the directory failed". Read its `summary` and name the succeeded count too.

---

## Re-running failed jobs

A reset is refused while the execution session holding those jobs is still alive: the tool returns `reset: False` with a
message naming the contested jobs. Cancel the session with `cancel_log_processing_tool`, or wait for it to finish, and
wait out the drain the cancellation note describes before resetting. A `source_ids` list matching no tracker entry
likewise returns `reset: False` with "No matching jobs found to reset."

A reset that lands mid-flight is not lost. An entry reading `SCHEDULED` is never overwritten when the engine reaps a
finished job, so only an entry still reading `RUNNING` is reconciled to a failure, and the re-run the reset asked for
survives.

Every tracker write the engine makes is nevertheless **best-effort**. The failure write, the reset write, and the status
read the reap depends on each absorb any exception the tracker raises. A tracker that cannot be read or written
therefore leaves its jobs in whatever state they last held instead of failing the batch. A job still reading `RUNNING`
after the session ended points at the tracker file, not at the job.

1. Identify failed jobs from `get_log_processing_status_tool` output (check `error_message` field)
2. Call `reset_log_processing_jobs_tool` with the tracker path and failed source IDs
3. Confirm the call returned `reset: True` before continuing
4. Re-prepare or re-execute the reset jobs using the same workflow

To re-process an entire directory from scratch, call `clean_log_processing_output_tool` to delete the
`microcontroller_data/` subdirectory, then re-prepare and re-execute.

---

## Error routing

Only `invalid_paths` and `skipped_sources` are soft. Every other preparation failure places its directory in
`failed_directories` and leaves the rest of the batch prepared, so read all three lists before executing. See
[error-routing.md](references/error-routing.md) for every skip reason and its remedy, every preparation and execution
error, the extraction-config faults that arrive as `FAILED` jobs, and the batch abandonment messages.

---

## Related skills

| Skill                                  | Relationship                                                     |
|----------------------------------------|------------------------------------------------------------------|
| `/communication-mcp-environment-setup` | Prerequisite: MCP server connectivity                            |
| `/microcontroller-setup`               | Upstream: hardware discovery and manifest management             |
| `/microcontroller-interface`           | Upstream: code that produces the log data being processed        |
| `/extraction-configuration`            | Upstream: extraction config that controls what data is extracted |
| `/log-input-format`                    | Reference: input archive format and source ID semantics          |
| `/log-processing-results`              | Downstream: output data discovery and event analysis             |
| `/cli-reference`                       | Reference: the full `axci` command surface, human-facing         |
| `/pipeline`                            | Context: log processing is phase 5 of the end-to-end pipeline    |

---

## Verification checklist

```text
Log Processing Workflow, tool-settled (run `get_batch_status_overview_tool` on the root directory):
- [ ] Existing state checked before anything was prepared, when the root may already have been processed
- [ ] Archives discovered via `discover_microcontroller_data_tool`
- [ ] Batch prepared via `prepare_log_processing_batch_tool`
- [ ] Jobs executed via `execute_log_processing_jobs_tool`
- [ ] Every tracker entry reads SUCCEEDED, or reads FAILED with an `error_message` that was investigated

Log Processing Workflow, agent-judged:
- [ ] MCP server connected (if not, invoke `/communication-mcp-environment-setup`)
- [ ] Log directories confirmed with user
- [ ] Output directories confirmed with user
- [ ] Extraction config validated via `/extraction-configuration`
- [ ] Returned `source_ids`, `skipped_sources`, `invalid_paths`, and `failed_directories` all reconciled
- [ ] Resource allocation confirmed with user
- [ ] Failed jobs retried if needed (use `clean_log_processing_output_tool` for full reset)
- [ ] Successful output verified via `/log-processing-results`
```
