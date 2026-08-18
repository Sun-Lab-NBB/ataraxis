---
name: log-processing
description: >-
  Orchestrates batch log processing via the ataraxis-video-system MCP server: archive discovery, batch preparation,
  job execution, progress monitoring, cancellation, and error recovery. Use when processing camera log archives,
  extracting frame timestamps, or managing batch processing jobs.
user-invocable: false
---

# Log processing

Orchestrates the batch log processing workflow: discover log archives, build the prepare manifest, dispatch timestamp
extraction jobs, monitor progress, and hand off to downstream skills for output verification and analysis.

---

## Scope

**Covers:**
- Archive discovery and recording hierarchy resolution
- Batch preparation and prepare manifest creation
- Job execution with resource allocation
- Progress monitoring and timing
- Cancellation and cleanup
- Failed job reset and retry
- Batch status overview across directories
- Status formatting and presentation

**Does not cover:**
- Input data format, archive structure, or source ID semantics (see `/log-input-format`)
- Output data formats, feather file schema, or frame statistics analysis (see `/log-processing-results`)
- Camera hardware setup or interactive testing (see `/camera-setup`)
- Writing VideoSystem integration code (see `/camera-interface`)
- Archive assembly, which must already have happened (see `/post-recording`)
- The `axvs process` command surface and its failure modes (see `/cli-reference`)
- MCP server connectivity issues (see `/video-mcp-environment-setup`)

---

## Agent requirements

You MUST use the ataraxis-video-system MCP tools for all processing operations. Do not import log processing Python
functions directly or run processing via CLI commands. If MCP tools are not available, invoke
`/video-mcp-environment-setup` to diagnose and resolve connectivity issues.

You MUST run `discover_camera_data_tool` before calling `prepare_log_processing_batch_tool` to obtain confirmed log
directory paths and source IDs. You MUST ask the user for output directory paths before preparing the batch. There is no
default, and output directories are required for every log directory being processed.

---

## Available tools

### Discovery tools

| Tool                        | Purpose                                                                           |
|-----------------------------|-----------------------------------------------------------------------------------|
| `discover_camera_data_tool` | Searches for camera manifests, locates archives, video files, and feather outputs |

Uses **manifest-based routing**: recursively searches for `camera_manifest.yaml` files under the root directory. Each
manifest identifies a DataLogger output directory containing axvs-produced log archives. Only sources whose log archives
exist on disk are included. For each confirmed source, resolves the paired video file and processed timestamp feather
file.

A bare call reports `log_directories`, `total_sources`, `total_log_directories`, and a `breakdown` naming every source
ID and camera name the scan found, which is everything the preparation step needs. It lists **no** `sources`. Pass
`include_items=True` to list them, and `detailed=True` to add each source's archive, video, and feather paths. See
[staged-reads.md](references/staged-reads.md) for the parameters, the return fields, and the paging rules.

**Important:** This tool requires `camera_manifest.yaml` files to exist in DataLogger output directories. Every
`VideoSystem.__init__()` call writes one, since `name` is a required constructor parameter. For legacy sessions without
manifests, use `write_camera_manifest_tool` (see `/camera-setup`) to retroactively tag log directories before running
discovery.

### Preparation and execution tools

| Tool                                | Purpose                                                 |
|-------------------------------------|---------------------------------------------------------|
| `prepare_log_processing_batch_tool` | Creates the prepare manifest without starting execution |
| `execute_log_processing_jobs_tool`  | Dispatches prepared jobs for background execution       |

**`prepare_log_processing_batch_tool` parameters:**

| Parameter            | Type        | Default    | Description                                                                   |
|----------------------|-------------|------------|-------------------------------------------------------------------------------|
| `log_directories`    | `list[str]` | (required) | Absolute paths to DataLogger output directories. **Ask the user.**            |
| `source_ids`         | `list[str]` | (required) | Confirmed source IDs, read from the discovery `breakdown`. Applied uniformly. |
| `output_directories` | `list[str]` | (required) | Absolute paths for per-directory output. Must match `log_directories` length. |

Passing an empty `source_ids` list prepares every source the log directory's `camera_manifest.yaml` registers. Sourcing
is lenient, so a requested source the manifest does not register, or one whose archive is absent or resolves to more
than one file, is recorded under `skipped_sources` with its reason instead of failing the whole batch.

**`prepare_log_processing_batch_tool` return structure:**

```text
success:                 False when no directory prepared and at least one failed, true otherwise
log_directories:         Mapping keyed by each log directory path, each value carrying:
  tracker_path:          Absolute path to the ProcessingTracker YAML file
  output_directory:      Absolute path to the created camera_timestamps/ subdirectory
  source_ids:            The source IDs that produced a dispatchable job
  jobs:                  Dispatchable job descriptors (see the execute tool below)
  summary:               Live tracker status counts (total, succeeded, failed, running, scheduled)
  skipped_sources:       Entries of {source_id, reason} for every source that produced no job
total_log_directories:   Number of log directories that prepared successfully
total_jobs:              Total dispatchable jobs across all directories
invalid_paths:           Present only when a named path is not a directory
failed_directories:      Present only when a directory's preparation raised. Entries of {log_directory, error}
```

The two failure keys separate two different faults. `invalid_paths` holds a bare path string for a name that is not a
directory on this host, which is a typo or a wrong mount. `failed_directories` holds a real directory whose preparation
raised, as an entry pairing the `log_directory` with the `error` the library reported, so read that message rather than
inferring the cause from the path.

A directory whose tree holds no `camera_manifest.yaml`, holds several of them, or whose resolved archives span several
parent directories reaches `failed_directories`. A tree holding no manifest registers no source at all, so either its
archives came from another library or the recording was logged without one. Pass each DataLogger output directory as
its own entry rather than a parent grouping several of them.

Read `failed_directories` on every call whatever `success` reports. A run that prepared some directories while others
failed still reads True, so the flag alone never establishes that every named directory was prepared.

**`execute_log_processing_jobs_tool` parameters:**

| Parameter          | Type         | Default    | Description                                                          |
|--------------------|--------------|------------|----------------------------------------------------------------------|
| `jobs`             | `list[dict]` | (required) | Job descriptors copied unmodified from the prepare manifest's `jobs` |
| `core_budget`      | `int`        | `-1`       | Keyword-only. Total CPU cores for the session, -1 auto-resolves      |
| `memory_budget_mb` | `int`        | `-1`       | Keyword-only. Total megabytes for the session, -1 auto-resolves      |

Pass each job descriptor through unchanged. The tool reads all ten keys the prepare manifest stamps onto it:
`log_directory`, `archive_path`, `output_directory`, `tracker_path`, `job_name`, `job_id`, `source_id`, `core_weight`,
`message_count`, and `archive_bytes`. A job missing any of them is rejected into `invalid_jobs`, and a call in which
every job is rejected returns `{"error": "No valid jobs to execute.", "invalid_jobs": [...]}`, whose entries repeat
each rejected job with the reason it was rejected.

**`execute_log_processing_jobs_tool` return structure:**

```text
started:            True once the background execution manager is running
total_jobs:         Number of valid jobs queued for this session
core_budget:        The resolved core budget for this session
memory_budget_mb:   The resolved memory budget for this session
pool_size:          The job slots the session's shared process pool opens
job_allocations:    Per job: job_id, source_id, cores, memory_mb, and message_count
invalid_jobs:       Present only when some submitted job could not be read
```

### Monitoring and management tools

| Tool                               | Purpose                                                             |
|------------------------------------|---------------------------------------------------------------------|
| `get_log_processing_status_tool`   | Per-job status of active execution session                          |
| `get_log_processing_timing_tool`   | Per-job timing and session-level throughput                         |
| `cancel_log_processing_tool`       | Cancels active session, clears pending queue                        |
| `reset_log_processing_jobs_tool`   | Resets specific or all jobs to SCHEDULED for retry                  |
| `get_batch_status_overview_tool`   | Aggregate status across all `camera_timestamps/` outputs under root |
| `clean_log_processing_output_tool` | Deletes `camera_timestamps/` subdirectories for re-processing       |

**`get_log_processing_timing_tool` return structure:**

Takes no parameters. Returns `active` (manager thread alive), a `jobs` list, and a `session` summary. Each job entry
carries `job_id`, `source_id`, and the `log_directory` the job reads, plus `executor_id`, `started_at` (microsecond
UTC), `elapsed_seconds` (running jobs only), `completed_at`, and `duration_seconds` (finished jobs only) where
available. A batch spanning two recordings of one camera carries that camera's `source_id` in both, so key a per-job
table on `log_directory` alongside `source_id`. The `session` block carries `total_elapsed_seconds`, `completed_count`,
`failed_count`, `running_count`, and `pending_count`, plus `throughput_jobs_per_hour` once at least one job has
completed. When no execution session exists it returns `{"active": false, "message": "No execution session exists."}`
with neither `jobs` nor `session`, so check for those keys before indexing them.

**`cancel_log_processing_tool` return structure:**

Takes no parameters. A live session returns `canceled: true`, a `session_ended` flag, a `message`, and a `final_state`.
That block's `succeeded_jobs` and `failed_jobs` count the jobs this session itself drove to a terminal outcome, which is
narrower than the tracker's own entries because a tracker records every job that ever wrote to its directory. Its
`active_jobs_at_cancel` is the number of jobs the session still had running when the queue was cleared.

`session_ended` reports whether the execution slot is free for the next batch. A call that leaves no job running waits
for the session to end and reports `Canceled. Cleared {n} pending job(s). No job was still running.`, and the following
`execute_log_processing_jobs_tool` call is accepted at once. A call made while jobs are still in flight reports
`Canceled. Cleared {n} pending job(s). {m} job(s) still completing. Poll get_log_processing_status_tool until 'active'
reads false before starting another execution.` Cancellation clears the pending queue alone, so read `session_ended`
first and poll `get_log_processing_status_tool` only while it reads false.

Every call made without a live session, including one made after the batch ran to completion, returns
`{"canceled": false, "session_ended": true, "message": "No execution session is active."}` with no `final_state`.

**`reset_log_processing_jobs_tool` parameters:**

| Parameter      | Type               | Default    | Description                                         |
|----------------|--------------------|------------|-----------------------------------------------------|
| `tracker_path` | `str`              | (required) | Absolute path to ProcessingTracker YAML file        |
| `source_ids`   | `list[str] / None` | `None`     | Source IDs to reset. If omitted, all jobs are reset |

`source_ids` are matched against each tracker job's `specifier`, which holds the source ID string. Returns
`reset: true`, a `jobs_reset` count, and the tracker's refreshed `jobs` list and `summary` counts. Returns
`{"reset": false, "message": "No matching jobs found to reset."}` when no job matches, and an `error` dictionary when
the tracker file is absent or cannot be read. Confirm `reset` is true and `jobs_reset` is non-zero before re-executing,
since all three shapes are truthy dictionaries.

**`get_batch_status_overview_tool`:**

Requires `root_directory`, the absolute path under which trackers are searched for. A bare call reports
`total_output_directories`, an aggregate `summary`, and a `breakdown` of directories per status, and it lists **no**
`output_directories`. Name `statuses`, or pass `include_items=True`, to list them, and `detailed=True` to add each
directory's tracker path and per-job entries. See [staged-reads.md](references/staged-reads.md) for both. The
`output_directory` value names the `camera_timestamps/` subdirectory holding the tracker, not the DataLogger output
directory the archives came from.

**`clean_log_processing_output_tool` parameters:**

| Parameter            | Type        | Default    | Description                                                                    |
|----------------------|-------------|------------|--------------------------------------------------------------------------------|
| `output_directories` | `list[str]` | (required) | Absolute paths to output directories containing `camera_timestamps/` to delete |

Deletes the `camera_timestamps/` subdirectory and all contents (feather files, tracker) under each output directory.
Returns a `results` list with per-directory outcomes plus `total_cleaned` and `total_directories` counts.

Each `results` entry carries `output_directory` and a `cleaned` flag. A successful delete adds `timestamps_path`. A
directory that had no `camera_timestamps/` to remove adds `message: "Nothing to clean."` and still reports
`cleaned: true`. An output directory that is absent or is not a directory reports `error` alone with `cleaned: false`,
while a failed delete reports both `timestamps_path` and `error`. Because `total_cleaned` counts the flag rather than
actual deletions, a run over wrong paths can still report every directory cleaned. Confirm each entry carries a
`timestamps_path` before reporting a full reset.

---

## Pipeline architecture

Single-phase timestamp extraction pipeline:

```text
.npz log archives → extract_logged_camera_timestamps → Polars DataFrame → .feather IPC files
```

- **ProcessingTracker** manages job lifecycle: `SCHEDULED` → `RUNNING` → `SUCCEEDED` / `FAILED` via YAML state files
- **Single execution session** constraint: only one batch execution can run at a time
- **Parallel processing** activates automatically once an archive reaches the library's parallel extraction threshold
  (see Resource management)
- **Output layout:** All processing output is written under a `camera_timestamps/` subdirectory within the
  output directory provided by the user
- **Output naming:** `camera_{source_id}_timestamps.feather` (Feather IPC format)
- **Tracker filename:** `camera_processing_tracker.yaml`

---

## Processing workflow

### Execution model

The processing workflow uses a **prepare-then-execute** model:

1. **Prepare** builds the prepare manifest (tracker files, job lists) without starting any computation.
   This step is idempotent. Calling it again on the same directories returns the existing manifest with
   current job statuses.

2. **Execute** dispatches jobs from the manifest with resource allocation and background thread management.

### Workflow steps

1. **Discover archives**: Call `discover_camera_data_tool` with the user-provided root directory.

2. **Present discovery results**: Call it again with `include_items=True` and `detailed=True`, render one table row per
   entry of `sources[]` carrying its `source_id`, `name`, `recording_root`, and `log_archive`, and close with the
   returned `total_sources` and `total_log_directories`.

3. **Confirm directories to process**: Ask the user which log directories to process. Accept all
   discovered directories or a user-selected subset. You MUST confirm before proceeding.

4. **Confirm output directories**: Ask the user for the output directory paths, one per log directory. You MUST
   confirm before proceeding.

5. **Prepare batch**: Call `prepare_log_processing_batch_tool` with the confirmed log directories,
   source IDs, and output directories. All three parameters are required. Reconcile the result against the
   directories you asked for before continuing. A directory missing from `log_directories` sits under
   `invalid_paths` or `failed_directories`, and a confirmed source that produced no job sits under its
   directory's `skipped_sources`. Report every one of them to the user rather than executing a shortened batch
   silently.

6. **Confirm resource allocation**: Present both defaults, `core_budget=-1` and `memory_budget_mb=-1`, and ask whether
   the user wants to override either. The Resource management section covers what each budget bounds.

7. **Execute jobs**: Call `execute_log_processing_jobs_tool` with the job descriptors from the prepare
   manifest and confirmed resource settings.

8. **Monitor progress**: Use `get_log_processing_status_tool` to check per-job progress. Optionally use
   `get_log_processing_timing_tool` for elapsed time and throughput metrics. Present status as a
   formatted table (see the Status formatting section).

9. **Handle completion**: When all jobs finish, check for failures. On success, invoke
   `/log-processing-results` to discover and analyze the output. On failure, see the Error routing section.

---

## Resource management

Every job is sized by the library, never by the agent. Preparation reads each archive's zip directory once and stamps
the resulting figures onto the job descriptor, and execution resolves the session budgets and reports what it allocated.
Read those figures from the assets below rather than recomputing them, because the sizing model is tuned per release and
any formula reproduced in this skill would drift out of agreement with the library running the batch.

| Asset                                             | Reports                                                                          |
|---------------------------------------------------|----------------------------------------------------------------------------------|
| `prepare_log_processing_batch_tool` → `jobs[]`    | Per job: `core_weight`, `memory_mb`, `message_count`, and `archive_bytes`        |
| `execute_log_processing_jobs_tool` → return value | Resolved `core_budget`, `memory_budget_mb`, `pool_size`, and `job_allocations[]` |
| `size_archive_job(archive_path)`                  | `JobSizing(cores, memory_mb)` for one archive, without preparing a batch         |

Use the first to plan a batch and the second to report what the batch actually committed. The third is the library's
single-archive sizing entry point, which an external scheduler calls to derive both figures from one archive read. This
workflow reads the two tool assets instead, since it drives no processing from Python.

How the model behaves:

- A job's width takes one of two values with nothing between them. An archive holding fewer data messages than
  `_PARALLEL_EXTRACTION_THRESHOLD` opens no pool and takes a single core, and every archive at or above it takes the
  declared `CAMERA_EXTRACTION_JOB_CORES` allocation. Execution then collapses that width onto the session's core
  budget wherever the budget is narrower, which is the only narrowing the model applies. Read the width a job actually
  received from its `core_weight` (prepare) and `cores` (execute), never from the budget requested.
- `_PARALLEL_EXTRACTION_THRESHOLD` decides whether a job opens a pool at all, and the `PARALLEL_PROCESSING_THRESHOLD`
  that the `ataraxis-data-structures` archive reader applies decides how it batches messages inside one. They carry
  different values, so a claim about one settles nothing about the other. The leading underscore marks the visibility
  split, since the first is private to `orchestration/allocation.py` and the second is a public library export.
- Execution re-derives `core_weight` and re-estimates `memory_mb` for every descriptor against this session's own
  budgets, so quote the execute figures whenever they disagree with the ones preparation stamped. The re-derivation
  reads the figures carried in the submitted job descriptors and touches no archive.
- Both budgets auto-resolve from the host when left at `-1`, holding back a reserve for host operations. An
  explicit positive `core_budget` is honored up to the host's logical core count, while `memory_budget_mb` is taken
  verbatim and is never capped against host RAM, so an over-large value invites the host to kill workers.
- `pool_size` follows the smallest of the job count, the core budget, and the job bodies that the share of the
  memory budget reserved for warmed bodies can hold. Every idle slot is charged a spawned child's baseline
  against that budget.
- Queued jobs are admitted heaviest first whenever their cores and memory both fit the remaining budget, so lighter
  jobs backfill the capacity a heavier one leaves spare. One job larger than the whole budget is forced through
  alone, and only while nothing else is running.
- A job whose estimate passes the host's **total physical memory** is failed before it is ever admitted, with an
  `error_message` naming both figures. The estimate scales with the width the job is granted, so re-executing the
  reset job under a smaller explicit `core_budget` lowers it and can bring it under the host's memory.
  `memory_budget_mb` does not enter this check at all. When even a single-core estimate passes host memory, the
  remedy is a host with more memory or a smaller archive.
- The session rebuilds its shared pool a bounded number of times. Passing that bound abandons the batch and fails
  every unfinished job with the reason recorded on its tracker. A job is separately requeued a bounded number of
  times after a break it ran alone through, and passing that bound fails **that job alone** while the rest of the
  batch continues.

The forced-alone admission is the earliest out-of-memory indicator a batch produces, and its WARNING is invisible over
the stdio MCP transport. On the MCP path, infer the condition from a `job_allocations` entry whose `memory_mb` passes
the returned `memory_budget_mb`. Reduce `core_budget` to narrow both the concurrent set and the width any single job
receives, and `memory_budget_mb` to narrow the concurrent set alone.

---

## Status formatting

The `get_log_processing_status_tool` response carries both an `active` flag (manager thread alive) and a `canceled`
flag, plus a `message` of `No execution session exists.` when no session ever ran. Treat `active` as the completion
signal and `canceled` as the cancellation path. Per-job entries carry the `log_directory` the job reads, and may also
include an `executor_id`.

A per-job entry reports a fifth status, `UNKNOWN`, when its tracker cannot be read or when the tracker holds no entry
for the job. An `UNKNOWN` job is counted in none of the `summary` status counts while still counting toward
`summary.total`, so the four counts can sum below the total. Treat such a job as unresolved rather than complete, and
re-prepare the batch to regenerate the tracker.

When presenting batch status to the user, format as a table:

```text
**Log Processing Status**

Summary: 5/8 jobs complete | 1 running | 2 queued | 0 failed

| Source ID | Status    | Duration |
|-----------|-----------|----------|
| 051       | SUCCEEDED | 12.5s    |
| 052       | SUCCEEDED | 8.3s     |
| 053       | RUNNING   | 6.1s     |
| 054       | SCHEDULED | n/a      |
```

When using `get_batch_status_overview_tool` for multi-directory status:

```text
**Batch Overview**

| Output directory                        | Status    | Succeeded | Failed | Total |
|-----------------------------------------|-----------|-----------|--------|-------|
| /data/session1/out/camera_timestamps/   | completed | 3         | 0      | 3     |
| /data/session2/out/camera_timestamps/   | failed    | 1         | 1      | 2     |
```

The per-directory `status` is one of five labels resolved by priority, `failed` (any failed job, which overrides all) >
`completed` (all succeeded) > `processing` (any running) > `not_started` (all scheduled) > `in_progress` (otherwise). A
sixth label, `error`, is reported when a tracker cannot be read. A directory with 3 succeeded and 1 failed reports
`failed`.

---

## Re-running failed jobs

1. Identify failed jobs from `get_log_processing_status_tool` output (check the `error_message` field)
2. Call `reset_log_processing_jobs_tool` with the tracker path and failed source IDs
3. Re-prepare or re-execute the reset jobs using the same workflow

To re-process an entire directory from scratch, call `clean_log_processing_output_tool` to delete the
`camera_timestamps/` subdirectory, then re-prepare and re-execute.

---

## Error routing

### Preparation errors

| Signal                                | Resolution                                                             |
|---------------------------------------|------------------------------------------------------------------------|
| `Length mismatch`                     | Ensure `output_directories` matches `log_directories` length           |
| Path listed under `invalid_paths`     | The name is not a directory on this host. Check the path and the mount |
| Entry under `failed_directories`      | Read its `error`: no manifest, several manifests, or several outputs   |
| Source listed under `skipped_sources` | Read its `reason`: unregistered source, or absent/ambiguous archive    |
| `success` false with nothing prepared | Every named directory failed. Resolve each `failed_directories` error  |

Preparation returns an error dictionary for the length mismatch alone. Every other per-directory failure is reported
under `invalid_paths` or `failed_directories`, and every per-source failure under that directory's `skipped_sources`.

### Execution errors

| Error                                    | Resolution                                                            |
|------------------------------------------|-----------------------------------------------------------------------|
| `An execution session is already active` | Cancel it, then poll only while its `session_ended` flag reads false  |
| `No valid jobs to execute`               | Verify job descriptors have all required keys                         |
| `Tracker file not found`                 | Re-prepare the batch to regenerate tracker files                      |

### Processing failure routing

| Error pattern                        | Action                                                   |
|--------------------------------------|----------------------------------------------------------|
| Archive not found / file read errors | Verify `.npz` archives exist in the log directory        |
| MCP tools unavailable                | Invoke `/video-mcp-environment-setup`                    |
| Out of memory                        | Reduce `memory_budget_mb`, and `core_budget` if needed   |
| Corrupt tracker or partial output    | Call `clean_log_processing_output_tool`, then re-prepare |

---

## CLI equivalent

`axvs process` is the user-facing route to this workflow, and `/cli-reference` is canonical for its options and failure
modes. **You MUST never invoke `axvs` commands.** Three divergences matter when discussing it with a user:

- It handles ONE log directory per invocation, whereas this batch carries many
- It runs its jobs sequentially and aborts at the first failing job, whereas this batch isolates a failure to its
  own job and carries the rest through
- A positive `-w` reaches every job verbatim, whereas preparation here sizes each job from its own archive. Left at its
  `-1` default, `-w` resolves each job's width from that job's own archive exactly as this batch does, so the
  divergence is the explicit value alone

Both paths lock the same `camera_processing_tracker.yaml`, so never prepare or execute over a directory the user has an
`axvs process` run in. Whichever side asks second raises a `TimeoutError`.

---

## Related skills

| Skill                          | Relationship                                                    |
|--------------------------------|-----------------------------------------------------------------|
| `/video-mcp-environment-setup` | Prerequisite: MCP server connectivity                           |
| `/camera-setup`                | Upstream: camera discovery and testing                          |
| `/camera-interface`            | Upstream: VideoSystem integration code that produces logs       |
| `/post-recording`              | Upstream: verifies archives before processing                   |
| `/cli-reference`               | Reference: the `axvs process` command equivalent to this batch  |
| `/log-input-format`            | Reference: input archive format and source ID semantics         |
| `/log-processing-results`      | Downstream: output data discovery and frame statistics analysis |
| `/pipeline`                    | Context: log processing is phase 5 of the end-to-end pipeline   |

---

## Verification checklist

```text
Log Processing Workflow, tool-settled (call `discover_camera_data_tool`, `prepare_log_processing_batch_tool`,
`execute_log_processing_jobs_tool`, and `get_log_processing_status_tool`):
- [ ] Archives discovered via `discover_camera_data_tool`
- [ ] Batch prepared via `prepare_log_processing_batch_tool`
- [ ] Jobs executed via `execute_log_processing_jobs_tool`
- [ ] Status monitored until all jobs complete or fail
- [ ] Output written to `camera_timestamps/` subdirectory under each output directory

Log Processing Workflow, reader-judged:
- [ ] MCP server connected (if not, invoke `/video-mcp-environment-setup`)
- [ ] Log directories confirmed with user
- [ ] Output directories provided by user (required, no default)
- [ ] Resource allocation confirmed with user
- [ ] Failed jobs investigated and retried if needed (use `clean_log_processing_output_tool` for full reset)
- [ ] Successful output verified via `/log-processing-results`
```
