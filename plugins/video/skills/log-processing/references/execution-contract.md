# Execution contract

Records the two ways `execute_log_processing_jobs_tool` names its work, what it requires of a job descriptor, and every
shape it returns.

---

## Naming the work

The tool accepts either the descriptors an earlier preparation emitted or the arguments the preparation itself takes.

| Form                 | Arguments named                            | What the tool does                             |
|----------------------|--------------------------------------------|------------------------------------------------|
| Prepare and dispatch | `log_directories` and `output_directories` | Rebuilds the manifest here, then dispatches it |
| Descriptor           | `jobs`                                     | Dispatches exactly the descriptors named       |

Prepare and dispatch is the default, because it starts a batch of any size without echoing a whole manifest back
through the call, and preparation is idempotent, so the rebuild returns the jobs an earlier preparation already
reported. Use the descriptor form when you filtered or reordered that manifest and intend to dispatch a subset.

A call naming neither form dispatches nothing and returns, verbatim:

```text
No work was named. Pass the job descriptors from prepare_log_processing_batch_tool as 'jobs', or pass
'log_directories' and 'output_directories' to prepare and dispatch in one call.
```

`source_ids` is read on the prepare-and-dispatch form alone, where an unset or empty list dispatches every source the
recording's `camera_manifest.yaml` registers. `output_directories` must match the length of `log_directories`.

---

## Descriptor requirements

A descriptor passed under `jobs` carries all ten keys the prepare manifest stamps onto it: `log_directory`,
`archive_path`, `output_directory`, `tracker_path`, `job_name`, `job_id`, `source_id`, `core_weight`, `message_count`,
and `archive_bytes`. The last two are required alongside the other eight, because the tool reads the archive figures out
of the mapping rather than resolving them from the archive on disk. Pass each descriptor through unchanged.

Three checks reject a descriptor into `invalid_jobs`, each entry repeating the submitted mapping with an added `error`:

| Fault                                             | `error` text                                        |
|---------------------------------------------------|-----------------------------------------------------|
| Any of the ten keys absent                        | Names the absent keys and all ten a mapping carries |
| `message_count` or `archive_bytes` not an integer | Names both submitted values                         |
| `tracker_path` naming no file on disk             | `Tracker file not found: {path}`                    |

---

## Return structure

```text
started:              True once the background execution manager is running
total_jobs:           Number of valid jobs queued for this session
core_budget:          The resolved core budget for this session
memory_budget_mb:     The resolved memory budget for this session
pool_size:            The job slots the session's shared process pool opens
job_allocations:      Per job: job_id, source_id, cores, memory_mb, and message_count
skipped_sources:      Sources that produced no job, each {log_directory, source_id, reason}
invalid_paths:        Named paths that are not directories on this host
failed_directories:   Directories whose preparation raised, each {log_directory, error}
invalid_jobs:         Submitted mappings rejected before dispatch, each carrying an error key
```

`started`, `total_jobs`, both budgets, `pool_size`, and `job_allocations` are present on every dispatch.
`skipped_sources`, `invalid_paths`, and `failed_directories` are spliced in by a call that rebuilt the manifest, each
omitted when the preparation reported nothing for it, and each `skipped_sources` entry is tagged with the
`log_directory` that reported it because one dispatch spans many directories. `invalid_jobs` is present only when a
submitted mapping was rejected.

**A rebuild call is the only place the preparation's accounting is reported**, since the caller never sees the
preparation's own response. Reconcile all three keys from the dispatch response exactly as you would from a separate
preparation. An agent that reads `job_allocations` alone drops every skipped source silently.

---

## Error shapes

| Condition                                       | Returned dictionary                                 |
|-------------------------------------------------|-----------------------------------------------------|
| Neither form named                              | The "No work was named." error quoted above         |
| A rebuild that fails outright                   | The preparation's own bare `{error}` dictionary     |
| No submitted mapping survived validation        | `{error, invalid_jobs}`, plus the preparation notes |
| A session already holds the single session slot | The session-active refusal quoted below             |

A rebuild that fails forwards the preparation's error dictionary unchanged and dispatches nothing, so a length
mismatch returns `{"error": "Length mismatch: 2 log directories but 1 output directories."}` and nothing else. Every
other preparation fault is soft and travels under `invalid_paths`, `failed_directories`, or `skipped_sources` instead.

The "No valid jobs to execute." response also carries the preparation notes on a rebuild call, so its `skipped_sources`,
`invalid_paths`, and `failed_directories` explain why nothing survived. `started` is absent from it, and from every
other shape in this table, so test for `started` rather than for the absence of `error`.

The session refusal reads in full: "An execution session is already active. Cancel it with cancel_log_processing_tool,
then read 'session_ended' from that call and poll get_log_processing_status_tool only while it reads false."
