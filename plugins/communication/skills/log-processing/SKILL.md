---
name: log-processing
description: >-
  Orchestrates batch log processing via the ataraxis-communication-interface MCP server: archive discovery,
  batch preparation, job execution, progress monitoring, cancellation, and error recovery. Use when processing
  microcontroller log archives, extracting hardware module and kernel data, or managing batch processing jobs.
user-invocable: false
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

This rule is absolute and has no carve-out. Orchestration happens through the MCP tools, or through the
`axci` CLI a user runs by hand. There is no third path an agent may take.

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

| Parameter            | Type        | Default    | Description                                                                          |
|----------------------|-------------|------------|--------------------------------------------------------------------------------------|
| `log_directories`    | `list[str]` | (required) | Absolute paths to DataLogger output directories. **Ask user.**                       |
| `source_ids`         | `list[str]` | (required) | Confirmed source IDs from `discover_microcontroller_data_tool`. Strings, never ints. |
| `output_directories` | `list[str]` | (required) | Absolute paths for per-directory output. Must match log_directories length.          |
| `config_path`        | `str`       | (required) | Absolute path to the validated ExtractionConfig YAML file.                           |

**Note:** every ordering the library returns sorts source identifiers **as strings**, so `"10"` precedes `"2"`.
A report that needs numeric order must sort the identifiers itself rather than presenting the returned order.

**Note:** `source_ids` entries are strings and are compared by exact string equality. Pass the `source_id` values
`discover_microcontroller_data_tool` returned verbatim. The manifest tools report the same controller as an integer
`id`, so an ID copied from there must be converted with `str()` first (see `/log-input-format`).

**Note:** An empty `source_ids` list is a shorthand for "every controller the extraction config declares", not for
"no controllers". It is also a filter footgun: one `source_ids` list is applied uniformly to every entry of
`log_directories`, so a list built from one recording becomes the request for all of them.

**Note:** Prepare keeps only the source IDs that the microcontroller manifest registers, that the extraction
config declares, and that resolve to exactly one on-disk `{source_id}_log.npz` archive. Every dropped ID is
reported, with its reason, under that directory's `skipped_sources` list rather than raising. A directory with no
matching archives returns `jobs: []` and `source_ids: []` while still reporting `success: True`. After preparing,
verify the returned source_ids/jobs match the requested set, and read `skipped_sources` for the reason behind any
mismatch rather than assuming a missing archive. Each reason and its remedy are tabulated under Lenient sourcing.

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
  skipped_sources[]:    Requested IDs that produced no job, each `{source_id, reason}` (see Lenient sourcing)
total_log_directories:  Number of entries in `log_directories`, which is not the number that produced jobs
total_jobs:             Number of job descriptors across every entry
invalid_paths[]:        Input paths that are not directories (present only if any)
failed_directories[]:   Directories whose preparation raised, each `{log_directory, error}` (present only if any)
```

**Note:** `summary` counts every job the tracker on disk holds, while `jobs` lists only what this call prepared. A
directory prepared for a subset of its controllers therefore reports a `summary` total larger than its `jobs`
length. Decide what still needs running from each job's own `status`, not from the summary.

**Note:** Preparation is idempotent against the manifest rather than against the request. The full set of controllers
the microcontroller manifest registers decides only which tracker entries are foreign, so preparing one controller
leaves every sibling's recorded outcome intact. Registration itself covers the prepared controllers alone: a tracker
prepared for one controller holds one entry, and a sibling appears only once it is itself prepared. A `summary` total
below the manifest's controller count is therefore normal, not a lost registration. The alignment is destructive in
one direction: a tracker entry whose controller
the current manifest no longer registers is deleted along with its recorded outcome. Regenerating or editing a
manifest and re-preparing therefore erases the completed-job history of every controller the manifest dropped. A
preparation that resolves no job writes nothing at all, creating neither the output subdirectory nor the tracker.

**`execute_log_processing_jobs_tool` parameters:**

| Parameter          | Type         | Default    | Description                                                                                     |
|--------------------|--------------|------------|-------------------------------------------------------------------------------------------------|
| `jobs`             | `list[dict]` | (required) | Job descriptors from the execution manifest. Pass each dict through unchanged (see note below). |
| `core_budget`      | `int`        | `-1`       | Cores the session may commit; -1 auto-resolves from the host, keeping a reserve.                |
| `memory_budget_mb` | `int`        | `-1`       | Memory in MB the session may commit; -1 auto-resolves from host physical memory.                |

**Note:** every job descriptor must carry all twelve keys the preparation emitted — `log_directory`,
`archive_path`, `output_directory`, `config_path`, `tracker_path`, `job_name`, `job_id`, `source_id`,
`core_weight`, `message_count`, `archive_bytes`, and `modeled`. A hand-assembled descriptor that drops the sizing
keys is rejected into `invalid_jobs` with "Missing or unreadable sizing keys from the prepared manifest."

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

These four session-level keys and the `job_allocations` list are the authoritative record of what the sizing
model decided for this batch. Report them to the user instead of estimating from archive sizes. When no descriptor
survives validation the tool returns an error dictionary carrying `invalid_jobs` instead, and `started` is absent.

### Monitoring and management tools

| Tool                                | Purpose                                                               |
|-------------------------------------|-----------------------------------------------------------------------|
| `get_log_processing_status_tool`    | Per-job status of active execution session                            |
| `get_log_processing_timing_tool`    | Per-job timing and session-level throughput                           |
| `cancel_log_processing_tool`        | Cancels active session, clears pending queue                          |
| `reset_log_processing_jobs_tool`    | Resets specific or all jobs to SCHEDULED for retry                    |
| `get_batch_status_overview_tool`    | Aggregate status across all log directories under root                |
| `clean_log_processing_output_tool`  | Deletes `microcontroller_data/` subdirectories for re-processing      |

**Note:** `get_log_processing_status_tool`, `get_log_processing_timing_tool`, and `cancel_log_processing_tool`
report only on the single active in-memory execution session (the most recent `execute_log_processing_jobs_tool`
call) and return a no-active-session response otherwise — e.g. when called before execute or after a server
restart, even if trackers with running jobs exist on disk (status/timing return `'No execution session exists.'`;
cancel returns `{canceled: False}` with `'No execution session is active.'`). For status of directories not in the
active session, use `get_batch_status_overview_tool`, which reads trackers from disk.

**Completion test:** a finished session reads `active: False` **with** a populated `jobs` list, while a server that
never held one reads `active: False` with a `message` and no `jobs` key at all. Branch on the presence of `jobs`,
not on `active` alone.

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

This tool, not the status tool, is the only source of the per-job Duration column in the Status formatting section.
It also supplies the **stall test**: a job whose `elapsed_seconds` keeps rising while `running_count` and
`pending_count` never change is stuck, whereas a batch that is merely slow moves `pending_count` down over time.

**`cancel_log_processing_tool` return structure:**
```text
canceled:                  True when a live session was canceled, False when none was active
message:                   Names the pending jobs cleared and the jobs still completing
final_state:               Present only when a session was canceled:
  succeeded_jobs:          This session's jobs that had already succeeded
  failed_jobs:             This session's jobs that had already failed
  active_jobs_at_cancel:   Jobs still running when the pending queue was cleared
```

**Note:** Cancellation clears the pending queue and stops new admissions; jobs already running are left to finish.
The session therefore stays alive while they drain, and an `execute_log_processing_jobs_tool` call made during the
drain still returns "An execution session is already active". Poll `get_log_processing_status_tool` until `active`
reads `False` before executing or resetting anything. Cleared queued jobs stay `SCHEDULED` and are re-runnable
directly — they need no reset.

**`reset_log_processing_jobs_tool` parameters:**

| Parameter      | Type               | Default    | Description                                         |
|----------------|--------------------|------------|-----------------------------------------------------|
| `tracker_path` | `str`              | (required) | Absolute path to ProcessingTracker YAML file        |
| `source_ids`   | `list[str] / None` | `None`     | Source IDs to reset; if omitted, all jobs are reset |

**Note:** `source_ids` are matched against each tracker entry's specifier by exact string equality, never by
substring, so `"10"` does not select the job of controller `101`. Omitting `source_ids` is the only way to reset
every job in the tracker.

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
`microcontroller_data/` reports it alongside "Nothing to clean.". `total_cleaned` therefore cannot confirm a real
reset. Require `data_path` on the entry to confirm one, and treat a `message` entry as a sign the output directory
was not the one processing wrote into.

---

## Lenient sourcing

Preparation over MCP never fails a whole directory because one controller could not be sourced. Every requested
controller that yields no job is recorded under that directory's `skipped_sources` as a `{source_id, reason}` pair,
and the remaining controllers still prepare. The three reason strings are fixed, so route on them directly:

| `reason` (verbatim)                                                         | What it means                                                                                                                                         | Remedy                                                                                                                                                                      |
|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `The controller is absent from the microcontroller manifest.`               | The `microcontroller_manifest.yaml` under this log directory registers no controller with that ID, so nothing confirms this library wrote the archive | Register the controller with `write_microcontroller_manifest_tool` (see `/microcontroller-setup`), re-run discovery, then prepare again                                     |
| `The controller is absent from the extraction configuration.`               | The extraction config declares no `controller_id` entry for it, so nothing declares what to extract                                                   | Add the controller entry via `/extraction-configuration`, re-validate, then prepare again                                                                                   |
| `The controller's log archive is absent or resolves to more than one file.` | No `{source_id}_log.npz` resolved under the tree, or several did, which means the tree spans more than one logger                                     | Assemble the archive with `assemble_log_archives_tool` (see `/microcontroller-setup`) when none exists; pass one DataLogger output directory per entry when several matched |

**Note:** Leniency is a property of the MCP path alone. `axci process` prepares with strict sourcing, where the same
three conditions raise instead of being recorded: the first two raise `ValueError` and the third raises
`FileNotFoundError`. A user reporting a hard CLI failure and an agent reporting a silent skip are looking at the
same misconfiguration.

**Note:** Two sourcing problems are never skips. A tree holding several `microcontroller_manifest.yaml` files, and
prepared archives that resolve into several different directories, both raise on the MCP path too and land that
directory in `failed_directories` (see Error routing).

---

## Pipeline architecture

Multi-target data extraction pipeline:

```text
.npz log archives + extraction_config.yaml → execute_job → Polars DataFrames → .feather IPC files
```

Key architectural facts:
- **ProcessingTracker** manages job lifecycle: `SCHEDULED` → `RUNNING` → `SUCCEEDED` / `FAILED` via YAML state files
- **Job identity:** every job is registered under the tracker job name `CONTROLLER_EXTRACTION_JOB_NAME`, and its
  `job_id` is that name hashed together with the `source_id`. One controller therefore keeps the same `job_id` in
  every recording, and a job is only unique batch-wide as the `(tracker_path, job_id)` pair
- **Single execution session** constraint: only one batch execution can run at a time
- **Process model:** every job body runs in a worker of one shared, spawn-context process pool, and a job admitted
  at more than one core opens its own extraction pool of that width inside its worker. A spawned child inherits
  nothing and re-imports the whole package, which is the cost the sizing model charges as `SPAWNED_CHILD_MEMORY_MB`
  for each process. Every worker is pinned to a single numeric-backend thread, so no worker oversubscribes the
  cores its job was admitted at
- **Parallel processing** activates automatically once an archive exceeds the library's parallel-processing
  threshold; the width a job actually receives is reported as its `core_weight` (prepare) and `cores` (execute)
- **Empty archives:** an archive with zero data messages completes as `SUCCEEDED` and produces no feather files —
  this is expected, not a failure to retry or clean
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
   current job statuses. See the idempotence note under `prepare_log_processing_batch_tool` for what that
   protects and for the one case in which re-preparing discards recorded outcomes.

2. **Execute** dispatches jobs from the manifest with resource allocation and background thread management.
   Only one execution session can be active at a time.

### Pre-processing checklist

```text
- [ ] Archives discovered or log directory paths provided
- [ ] Log directories confirmed with user
- [ ] Output directories provided by user (required, no default)
- [ ] Extraction config validated (invoke `/extraction-configuration` if needed)
- [ ] Resource allocation confirmed with user (`core_budget`, `memory_budget_mb`)
```

**STOP**: If any checkbox is incomplete, do not proceed. Complete the missing steps first.

### Workflow steps

0. **Orient before starting** — Whenever the root may already have been processed (the user says "resume", "finish",
   or "re-run", or names an existing project root), call `get_batch_status_overview_tool` on that root first. It
   reads trackers from disk, needs no execution session, and costs one call. It answers which directories are
   `not_started`, in flight, or `failed` before anything is prepared, so a resumed batch re-prepares only what it
   has to. When it reports no tracker at all, nothing has been processed and step 1 is the real starting point.

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
   source IDs, output directories, and config path. Then verify the returned `source_ids` against the set you
   requested, read `skipped_sources` for every difference (see Lenient sourcing), and check `modeled` on each job
   entry. A job carrying `modeled: false` was sized from a floor because its archive could not be read, so treat it
   as unsized and expect it to fail at extraction for the same reason.

7. **Confirm resource allocation** — Read the per-job `core_weight` and `memory_mb` the preparation
   stamped onto each job entry, and present those figures alongside the default budgets (`core_budget=-1`
   and `memory_budget_mb=-1`, both auto-resolving from the host). Ask whether the user wants to override.
   Never estimate the cost yourself — the preparation already sized every job from the archive it reads.
   Two overrides carry risk. A positive `core_budget` reserves nothing for the host, unlike `-1`. A positive
   `memory_budget_mb` is taken verbatim and is never checked against host RAM, so an over-large value invites the
   host to kill workers, and a small one collapses the pool to a single slot.

8. **Execute jobs** — Call `execute_log_processing_jobs_tool` with the job descriptors from the execution
   manifest and confirmed resource settings. Read the returned `core_budget`, `memory_budget_mb`, `pool_size`, and
   `job_allocations[]` back to the user: those are the figures this session actually committed, and they can differ
   from the `core_weight` and `memory_mb` the preparation reported.

9. **Monitor progress** — Use `get_log_processing_status_tool` to check per-job progress. Optionally use
   `get_log_processing_timing_tool` for elapsed time and throughput metrics. Present status as a
   formatted table (see Status Formatting section). The pool warms every slot before it admits the first job, so
   the first status read after execute normally shows every job `SCHEDULED`; that is not a stall. The manager then
   re-examines the running set on a fixed poll interval, so re-read after a few seconds rather than immediately.

10. **Handle completion** — The session is over when the status read returns `active: False` alongside a populated
    `jobs` list (see Completion test). Check that list for failures. On success, invoke `/log-processing-results`
    to discover and analyze the output. On failure, see Error Routing section.

---

## Resource management

Every job is sized by the library, never by the agent. Preparation reads each archive's zip directory once and
stamps the resulting figures onto the job entry, and execution resolves the session budgets and reports what it
allocated. Read those figures from the assets below rather than recomputing them. The sizing model is tuned per
release, so any formula reproduced in this skill would drift out of agreement with the library running the batch.

### Where the sizing figures come from

| Asset                                                 | Reports                                                                          |
|-------------------------------------------------------|----------------------------------------------------------------------------------|
| `prepare_log_processing_batch_tool` → `jobs[]`        | Per job: `core_weight`, `memory_mb`, `message_count`, `archive_bytes`, `modeled` |
| `execute_log_processing_jobs_tool` → return value     | Resolved `core_budget`, `memory_budget_mb`, `pool_size`, and `job_allocations[]` |
| `estimate_archive_job_memory_mb(archive_path, cores)` | `(memory_mb, modeled)` for one archive, without preparing a batch                |

Use the first to plan a batch, the second to report what the batch actually committed, and the third when
planning outside the batch tools. The third is exported from the package top level for exactly that purpose and
costs one stat call, reading the archive's file metadata rather than its contents.

**`estimate_archive_job_memory_mb` does not count messages.** It sizes memory from the archive's size on disk
alone, and it takes the core count as an argument rather than deriving one. It therefore answers "how much
memory would this job hold at N cores", not "how wide should this job be". A caller that treats its result as a
full sizing will under-width every job. To get the width as well, let the preparation tool size the batch and
read `core_weight` back.

**`modeled: false`** means the archive could not be read and the figure is a floor to plan around rather than a
measurement. Treat such a job as unsized, and expect its real footprint to exceed the reported one. It also
collapses the job to a single core whatever ceiling it was sized under, because an unmodeled footprint reports
no messages, so an unsized job comes back narrow as well as under-estimated.

### How the model behaves

These are the invariants to reason with. Read the absolute figures from the assets above rather than quoting
any number from this skill:

- A job's core count rises with the number of messages its archive holds, up to `CONTROLLER_EXTRACTION_JOB_CORES`,
  the widest allocation the library gives one extraction job. An archive below the parallel-processing threshold is
  read sequentially on a single core.
- Three caps bound that width and the narrowest of them wins: `CONTROLLER_EXTRACTION_JOB_CORES`, the ceiling the
  budget or the caller sets, and the workers the archive's own message count repays. A mid-size archive is therefore
  held well below the widest allocation even when the budget allows more. Read the width a job actually received from
  its `core_weight` (prepare) and `cores` (execute), never from the budget or the ceiling that was requested.
- A job's memory estimate rises with the archive's size on disk and with the cores it holds, because every
  worker opens the archive itself.
- The figures preparation stamps on a job are advisory. Execution re-derives `core_weight` and re-estimates
  `memory_mb` for every descriptor against this session's own budgets, so the same manifest run at a narrower
  `core_budget` reports narrower `cores` in `job_allocations` than its `core_weight` claimed. Quote the execute
  figures whenever the two disagree.
- Both budgets auto-resolve from the host when left at `-1`, holding back a reserve for host operations. An
  explicit positive value reserves nothing: `core_budget` is honored up to the host's logical core count, and
  `memory_budget_mb` is taken verbatim and is never capped against host RAM.
- One shared process pool serves the whole batch, and the slots it opens are reported as `pool_size` in the execute
  return. The slot count follows the smallest of the job count, the core budget, and the job bodies the memory
  budget can hold, so an unusually small memory budget collapses the pool to a single slot and the batch runs one
  job at a time whatever the core budget allows.
- Every **idle** pool slot is charged a spawned child's baseline against the memory budget, and an admitted job
  credits its own slot back. A large `pool_size` beside a small memory budget therefore admits fewer concurrent
  jobs than the per-job `memory_mb` figures alone suggest. Size a batch from `pool_size` and the budget together,
  never from the per-job estimates on their own.
- Queued jobs are considered heaviest first and admitted one at a time whenever their cores and memory both fit
  the remaining budget, so lighter jobs backfill the capacity a heavier one leaves spare. A job larger than the
  whole budget is admitted alone so the batch still progresses. Only **one** such job is forced, and only while
  nothing else is running, so a second oversized job waits for the batch to drain rather than starting beside it.
- A job whose estimate passes the host's **total physical memory** is failed before it is ever admitted, with an
  `error_message` naming both figures. Resetting it re-runs the same check and fails it again, so the remedy is a
  host with more memory or a smaller archive, never a different budget.

The forced-alone admission is the earliest out-of-memory indicator a batch produces: it emits a WARNING naming the
job, its estimate, and the batch budget. That warning is invisible over the stdio MCP transport, which disables the
console, so on the MCP path infer the same condition from a `job_allocations` entry whose `memory_mb` passes the
returned `memory_budget_mb`.

Reduce `core_budget` to narrow both the concurrent set and the width any single job receives. Reduce
`memory_budget_mb` to narrow the concurrent set alone.

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

A job may report status `UNKNOWN` when its tracker entry is unreadable or missing — resolve by re-preparing, not
by reset (which is for `FAILED` jobs). Status and timing entries also carry an optional `executor_id`, which the
tracker records as `pid:<process id>` for a job run by the local pool. Use it to separate two failure shapes:
several failed jobs sharing one `executor_id` mean a single worker keeps dying, while failures spread across many
distinct `executor_id` values mean the pool itself is being killed, which routes to Batch abandonment.

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

| Label         | Meaning                                                                                          |
|---------------|----------------------------------------------------------------------------------------------------|
| `failed`      | At least one job failed. Outranks every other label, so the directory may still hold succeeded jobs |
| `completed`   | Every job in the tracker succeeded                                                                 |
| `processing`  | At least one job is currently running                                                              |
| `not_started` | Every job is still scheduled                                                                       |
| `in_progress` | Mixed outcomes with nothing running and nothing failed; also a tracker holding no jobs at all      |
| `error`       | The tracker itself could not be read. The entry carries an `error` key instead of job details      |

Never report `failed` as "the directory failed" — read its `summary` and name the succeeded count too.

---

## Re-running failed jobs

A reset is refused while the execution session holding those jobs is still alive: the tool returns
`reset: False` with a message naming the contested jobs. Cancel the session with `cancel_log_processing_tool`,
or wait for it to finish, before resetting. Cancelling does not end the session on the spot — running jobs drain
first — so poll `get_log_processing_status_tool` until `active` reads `False`, then reset. A `source_ids` list
matching no tracker entry likewise returns `reset: False` with "No matching jobs found to reset."

A reset that lands mid-flight is not lost. An entry reading `SCHEDULED` is never overwritten when the engine
reaps a finished job, so only an entry still reading `RUNNING` is reconciled to a failure, and the re-run the
reset asked for survives.

Every tracker write the engine makes is nevertheless **best-effort**. The failure write, the reset write, and
the status read the reap depends on each absorb any exception the tracker raises, so a tracker that cannot be
read or written leaves its jobs in whatever state they last held instead of failing the batch. A job still
reading `RUNNING` after the session ended points at the tracker file, not at the job.

1. Identify failed jobs from `get_log_processing_status_tool` output (check `error_message` field)
2. Call `reset_log_processing_jobs_tool` with the tracker path and failed source IDs
3. Confirm the call returned `reset: True` before continuing
4. Re-prepare or re-execute the reset jobs using the same workflow

To re-process an entire directory from scratch, call `clean_log_processing_output_tool` to delete the
`microcontroller_data/` subdirectory, then re-prepare and re-execute.

---

## Error routing

### Preparation errors

| Error                                                            | Resolution                                                                                                                                                                                                                 |
|------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Non-existent / non-directory log path                            | Not a hard error; surfaces in the returned `invalid_paths` list. Verify the path exists and is a directory                                                                                                                 |
| "Length mismatch"                                                | Ensure output_directories matches log_directories length                                                                                                                                                                   |
| "Permission denied"                                              | Check filesystem permissions                                                                                                                                                                                               |
| "Extraction config not found: ..."                               | Verify extraction config path; use `/extraction-configuration`                                                                                                                                                             |
| "Invalid extraction config: ..."                                 | Validate config via `/extraction-configuration`                                                                                                                                                                            |
| "The directory tree holds N microcontroller_manifest.yaml files" | A hard per-directory failure, reported in `failed_directories`, not a skip. The path spans several recordings or several DataLogger instances. Pass each DataLogger output directory as its own entry of `log_directories` |
| "The resolved log archives sit in N different directories"       | The same fault seen through the archives instead of the manifests. Split the request the same way                                                                                                                          |
| "contains no controller entries"                                 | The manifest under that directory registers no controller. Re-tag the directory via `/microcontroller-setup`, then re-run discovery                                                                                        |
| "It declares no controllers."                                    | The extraction config itself is empty of controllers. Rebuild it via `/extraction-configuration`                                                                                                                           |
| A timeout acquiring the tracker lock                             | Another process holds that output directory's tracker — most often a user's own `axci process` run. Wait for it to finish, or prepare a different directory                                                                |

**Note:** Only `invalid_paths` and `skipped_sources` are soft. Every row above that names a directory-level failure
places that directory in `failed_directories` and leaves the rest of the batch prepared, so always read both lists
before executing.

### Execution errors

| Error                                    | Resolution                                          |
|------------------------------------------|-----------------------------------------------------|
| "An execution session is already active" | Wait for current session or cancel first            |
| "No valid jobs to execute"               | Verify job descriptors have all required keys       |
| "Tracker file not found"                 | Re-prepare the batch to regenerate tracker files    |

### Processing failure routing

| Error Pattern                        | Action                                                                                                                                                                      |
|--------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Archive not found / file read errors | Verify .npz archives exist in log directory. Cross-check `modeled` in the prepare output: a job prepared with `modeled: false` already failed to have its archive read once |
| Invalid extraction config            | Validate config via `/extraction-configuration`                                                                                                                             |
| MCP tools unavailable                | Invoke `/communication-mcp-environment-setup`                                                                                                                               |
| Worker killed by the host            | Lower `core_budget` first, then `memory_budget_mb`. A repeat is a pool break — see Batch abandonment                                                                        |
| Corrupt tracker or partial output    | Call `clean_log_processing_output_tool`, then re-prepare                                                                                                                    |

### Configuration faults that surface as failed jobs

Three extraction-config faults are not checked when the batch is prepared. They are raised inside the job body, so
they arrive as `FAILED` jobs carrying the message in `error_message` after execution has already started. All three
are exactly what `validate_extraction_config_tool` reports, so running that tool before preparing removes them.

| `error_message` contains                                                    | Cause and remedy                                                                                                                                                                          |
|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| "Module with type code T and ID code I has empty event_codes."              | That module entry declares no event codes. Validation reports the same fault as "event_codes is empty". Fill the entry via `/extraction-configuration`, then reset and re-execute the job |
| "Kernel extraction has empty event_codes."                                  | The controller's kernel entry declares no event codes. Fill it or remove the kernel entry entirely                                                                                        |
| "The controller config has no modules and no kernel extraction configured." | The controller entry declares no extraction target at all. Give it at least one module or a kernel entry                                                                                  |

### Batch abandonment

The engine absorbs a bounded number of worker kills before it gives up on the batch. When it gives up, every
unfinished job — running and queued alike — is marked `FAILED` with the same reason, so a whole batch failing at
once with one shared `error_message` is an abandonment rather than many independent data faults.

| `error_message` begins with                                                                                               | What happened                                                                                                                                                                        | Action                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| "The batch's shared worker pool broke ... times, which passes the ... rebuilds one session allows."                       | The host killed pool workers more times than one session tolerates; the message names both the count and the limit. Usually the memory budget promises more than the host can supply | Lower `core_budget` first, then `memory_budget_mb`, reset the failed jobs, and re-execute |
| "The batch's shared worker pool broke and could not be rebuilt:"                                                          | The replacement pool could not be created, commonly because the host cannot spawn more processes                                                                                     | Free host resources, then reset and re-execute                                            |
| "The batch's execution manager stopped:"                                                                                  | The manager thread itself raised. A pool whose workers do not all start within the warm-up timeout arrives here too                                                                  | Lower `core_budget` so fewer slots are opened, then reset and re-execute                  |
| "The worker running this job was killed ... times while the job ran alone, which passes the ... requeues one job allows." | One individual job is fatal. Only a job running alone is charged a requeue, so this names a single oversized or corrupt archive                                                      | Exclude that `source_id` from the batch, or move it to a host with more memory            |
| "Job terminated without updating tracker status."                                                                         | The job body died without recording an outcome, and the engine reconciled the entry so it does not read `RUNNING` forever                                                            | Reset that job and re-execute. A recurrence is the individually-fatal case above          |

**Note:** A job the engine requeues is returned to `SCHEDULED` for you, so a batch that recovers on its own needs no
manual reset. Only the jobs the engine explicitly failed need one.

---

## CLI reference (human-facing — do not invoke)

> **CLI reference — for answering user questions only.** The `axci` command-line interface is a **human-facing**
> tool. **Agents must never invoke `axci` commands** — every agent-driven operation has an equivalent MCP tool
> (noted in the table). This section exists solely so the agent can answer user questions about the CLI.
>
> `/cli-reference` is the canonical reference for the whole `axci` command surface, covering every command and
> every option with its type, default and effect. The table below is the processing subset. Invoke
> `/cli-reference` for anything it does not answer.

This is the human path that the "do not run processing via CLI" instruction in Agent requirements refers to.
`axci process` handles ONE directory per invocation, whereas the MCP workflow batches many. The two paths also
behave differently, which matters whenever a user reports something the MCP workflow cannot reproduce. The other
`axci` commands are documented in `/microcontroller-setup`, `/extraction-configuration`, and
`/communication-mcp-environment-setup`.

| Command        | Key options                                                                                                                           | Purpose                                                       | MCP equivalent                                                                   |
|----------------|---------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|----------------------------------------------------------------------------------|
| `axci process` | `-ld/--log-directory`, `-od/--output-directory`, `-c/--config`, `-id/--job-id`, `-s/--specifier`, `-w/--workers`, `-np/--no-progress` | Extracts module and kernel data from one directory's archives | `prepare_log_processing_batch_tool` / `execute_log_processing_jobs_tool` (batch) |

**Option semantics:**

| Option              | What it actually does                                                                                                                                                                                                                |
|---------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `-ld`, `-od`, `-c`  | The one log directory, the output root, and the extraction config. The short `-c` is `--config` here, while `axci config show` spells it `--config-path`                                                                             |
| `-id/--job-id`      | External single-job dispatch: runs only the job whose canonical hexadecimal identifier matches, which is how an external scheduler hands out one unit of work. When it is set, `-s` is ignored                                       |
| `-s/--specifier`    | A repeatable controller-ID selector, repeated once per controller. Every ID must also be declared in the extraction config and resolve to one archive, or the run raises. Omitting it processes every controller the config declares |
| `-w/--workers`      | A ceiling on the workers ONE job receives, not a batch width. `-1` resolves it from the host's cores minus the host reserve, and the result is capped at `CONTROLLER_EXTRACTION_JOB_CORES`. `1` makes every job sequential           |
| `-np/--no-progress` | Suppresses the extraction progress bars. It only changes anything for a job wide enough to run in parallel, since a sequential job renders nothing either way, so it matters mainly for non-interactive runs                         |

**How the CLI path diverges from the MCP path:**

| Divergence            | Consequence for the answer you give the user                                                                                                                                                                                                                              |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Strict sourcing       | A controller the manifest or the config does not register, or one whose archive does not resolve to exactly one file, raises instead of landing in `skipped_sources` (see Lenient sourcing)                                                                               |
| No sizing             | The CLI reads no archive before dispatch, weighs nothing against a budget, and runs its jobs one after another, so `core_weight`, `memory_mb`, `pool_size`, and `job_allocations` have no CLI counterpart                                                                 |
| No error recovery     | The local-mode loop has no exception handling. The first job that raises aborts the whole invocation, and every job behind it stays `SCHEDULED` with no requeue and no failure record of its own — unlike the batch engine, which requeues and then fails jobs explicitly |
| Empty result is fatal | A recording that resolves no job raises `FileNotFoundError`, where `prepare_log_processing_batch_tool` reports the same situation as `success: True` with an empty `jobs` list                                                                                            |

**Note:** The CLI and the MCP server contend for the same ProcessingTracker lock. A user running `axci process`
against a directory the agent is preparing or executing makes whichever side asks second raise a `TimeoutError`.
Ask the user to stop their CLI run before preparing or executing that directory.

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
Log Processing Workflow:
- [ ] MCP server connected (if not, invoke `/communication-mcp-environment-setup`)
- [ ] Existing state checked via `get_batch_status_overview_tool` when the root may already have been processed
- [ ] Archives discovered via `discover_microcontroller_data_tool`
- [ ] Log directories confirmed with user
- [ ] Output directories confirmed with user
- [ ] Extraction config validated via `/extraction-configuration`
- [ ] Batch prepared via `prepare_log_processing_batch_tool`
- [ ] Returned `source_ids`, `skipped_sources`, `invalid_paths`, and `failed_directories` all reconciled
- [ ] Resource allocation confirmed with user
- [ ] Jobs executed via `execute_log_processing_jobs_tool`
- [ ] Status monitored until all jobs complete or fail
- [ ] Failed jobs investigated and retried if needed (use `clean_log_processing_output_tool` for full reset)
- [ ] Successful output verified via `/log-processing-results`
```
