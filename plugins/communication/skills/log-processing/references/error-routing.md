# Error routing

Catalogs every failure the log processing batch tools report, from a controller that produced no job through to a batch
the engine abandoned.

---

## Lenient sourcing

Preparation over MCP never fails a whole directory because one controller could not be sourced. Every requested
controller that yields no job is recorded under that directory's `skipped_sources` as a `{source_id, reason}` pair, and
the remaining controllers still prepare. The three reason strings are fixed, so route on them directly:

| `reason` (verbatim)                                                         | What it means                                                                                                                                         | Remedy                                                                                                                                                                      |
|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| "The controller is absent from the microcontroller manifest."               | The `microcontroller_manifest.yaml` under this log directory registers no controller with that ID, so nothing confirms this library wrote the archive | Register the controller with `write_microcontroller_manifest_tool` (see `/microcontroller-setup`), re-run discovery, then prepare again                                     |
| "The controller is absent from the extraction configuration."               | The extraction config declares no `controller_id` entry for it, so nothing declares what to extract                                                   | Add the controller entry via `/extraction-configuration`, re-validate, then prepare again                                                                                   |
| "The controller's log archive is absent or resolves to more than one file." | No `{source_id}_log.npz` resolved under the tree, or several did, which means the tree spans more than one logger                                     | Assemble the archive with `assemble_log_archives_tool` (see `/microcontroller-setup`) when none exists. Pass one DataLogger output directory per entry when several matched |

**Note:** Leniency is a property of the MCP path alone. `axci process` prepares with strict sourcing, where the same
three conditions raise instead of being recorded: the first two raise `ValueError` and the third raises
`FileNotFoundError`. A user reporting a hard CLI failure and an agent reporting a silent skip are looking at the same
misconfiguration.

**Note:** A sourcing problem belonging to the whole directory rather than to one controller is never a skip, and the
Preparation errors table below carries each one with the remedy it takes. A tree holding no
`microcontroller_manifest.yaml` is the one that reads most like a skip, since an absent manifest registers no controller
at all, and it raises under both sourcing modes rather than being recorded.

---

## Preparation errors

| Error                                                            | Resolution                                                                                                                                                                                                                 |
|------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Non-existent / non-directory log path                            | Not a hard error. Surfaces in the returned `invalid_paths` list. Verify the path exists and is a directory                                                                                                                 |
| "Length mismatch"                                                | Ensure output_directories matches log_directories length                                                                                                                                                                   |
| "Permission denied"                                              | Check filesystem permissions                                                                                                                                                                                               |
| "Extraction config not found: ..."                               | Verify extraction config path. Use `/extraction-configuration`                                                                                                                                                             |
| "Invalid extraction config: ..."                                 | Validate config via `/extraction-configuration`                                                                                                                                                                            |
| "... its tree holds no microcontroller_manifest.yaml"            | A per-directory failure reported in `failed_directories`, not a skip. Its archives are foreign to this library, or the recording was logged without a manifest. Tag it via `/microcontroller-setup`, then re-run discovery |
| "The directory tree holds N microcontroller_manifest.yaml files" | A hard per-directory failure, reported in `failed_directories`, not a skip. The path spans several recordings or several DataLogger instances. Pass each DataLogger output directory as its own entry of `log_directories` |
| "The resolved log archives sit in N different directories"       | The same fault seen through the archives instead of the manifests. Split the request the same way                                                                                                                          |
| "contains no controller entries"                                 | The manifest under that directory registers no controller. Re-tag the directory via `/microcontroller-setup`, then re-run discovery                                                                                        |
| "It declares no controllers."                                    | The extraction config itself is empty of controllers. Rebuild it via `/extraction-configuration`                                                                                                                           |
| A timeout acquiring the tracker lock                             | Another process holds that output directory's tracker, most often a user's own `axci process` run. Wait for it to finish, or prepare a different directory                                                                 |
| "Unable to size the ... job that reads the log archive"          | The archive resolves but cannot be read, so the job reading it cannot run either. The whole directory fails into `failed_directories`. Repair or reassemble that archive, then prepare again                               |

**Note:** Only `invalid_paths` and `skipped_sources` are soft. Every row above that names a directory-level failure
places that directory in `failed_directories` and leaves the rest of the batch prepared, so always read all three lists
before executing.

---

## Execution errors

| Error                                                            | Resolution                                                                                                          |
|------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| "An execution session is already active."                        | Cancel it, read `session_ended` from that call, and poll `get_log_processing_status_tool` only while it reads False |
| "No work was named."                                             | Pass `jobs`, or `log_directories`, `output_directories`, and `config_path`                                          |
| "No valid jobs to execute."                                      | Read `invalid_jobs` for the reason each descriptor was rejected                                                     |
| "Tracker file not found: ..."                                    | Re-prepare the batch to regenerate tracker files                                                                    |
| "Unable to read a ... job descriptor from the supplied mapping." | Re-prepare and pass the emitted dicts verbatim                                                                      |

---

## Processing failure routing

| Error Pattern                        | Action                                                                                                                                                |
|--------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Archive not found / file read errors | Verify .npz archives exist in log directory. An archive that resolves but cannot be read fails its whole directory at preparation, so no job reads it |
| Invalid extraction config            | Validate config via `/extraction-configuration`                                                                                                       |
| MCP tools unavailable                | Invoke `/communication-mcp-environment-setup`                                                                                                         |
| Worker killed by the host            | Lower `core_budget` first, then `memory_budget_mb`. A repeat is a pool break. See Batch abandonment                                                   |
| Corrupt tracker or partial output    | Call `clean_log_processing_output_tool`, then re-prepare                                                                                              |

---

## Configuration faults that surface as failed jobs

Three extraction-config faults are not checked when the batch is prepared. They are raised inside the job body, so they
arrive as `FAILED` jobs carrying the message in `error_message` after execution has already started. All three are
exactly what `validate_extraction_config_tool` reports, so running that tool before preparing removes them.

| `error_message` contains                                                    | Cause and remedy                                                                                                                                                                          |
|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| "Module with type code T and ID code I has empty event_codes."              | That module entry declares no event codes. Validation reports the same fault as "event_codes is empty". Fill the entry via `/extraction-configuration`, then reset and re-execute the job |
| "Kernel extraction has empty event_codes."                                  | The controller's kernel entry declares no event codes. Fill it or remove the kernel entry entirely                                                                                        |
| "The controller config has no modules and no kernel extraction configured." | The controller entry declares no extraction target at all. Give it at least one module or a kernel entry                                                                                  |

---

## Batch abandonment

The engine absorbs a bounded number of worker kills before it gives up on the batch. When it gives up, every unfinished
job, running and queued alike, is marked `FAILED` with the same reason, so a whole batch failing at once with one shared
`error_message` is an abandonment rather than many independent data faults.

| `error_message` begins with                                                                                               | What happened                                                                                                                                                                        | Action                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| "The batch's shared worker pool broke ... times, which passes the ... rebuilds one session allows."                       | The host killed pool workers more times than one session tolerates. The message names both the count and the limit. Usually the memory budget promises more than the host can supply | Lower `core_budget` first, then `memory_budget_mb`, reset the failed jobs, and re-execute |
| "The batch's shared worker pool broke and could not be rebuilt:"                                                          | The replacement pool could not be created, commonly because the host cannot spawn more processes                                                                                     | Free host resources, then reset and re-execute                                            |
| "The batch's execution manager stopped:"                                                                                  | The manager thread itself raised. A pool whose workers do not all start within the warm-up timeout arrives here too                                                                  | Lower `core_budget` so fewer slots are opened, then reset and re-execute                  |
| "The worker running this job was killed ... times while the job ran alone, which passes the ... requeues one job allows." | One individual job is fatal. Only a job running alone is charged a requeue, so this names a single oversized or corrupt archive                                                      | Exclude that `source_id` from the batch, or move it to a host with more memory            |
| "Job terminated without updating tracker status."                                                                         | The job body died without recording an outcome, and the engine reconciled the entry so it does not read `RUNNING` forever                                                            | Reset that job and re-execute. A recurrence is the individually-fatal case above          |

**Note:** A job the engine requeues is returned to `SCHEDULED` for you, so a batch that recovers on its own needs no
manual reset. Only the jobs the engine explicitly failed need one.
