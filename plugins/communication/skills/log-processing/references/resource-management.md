# Resource management

Records how the library sizes each log processing job, and which returned figure carries each sizing decision.

---

## Where the sizing figures come from

Every job is sized by the library, never by the agent. Preparation reads each archive's zip directory once and stamps
the resulting figures onto the job entry, and execution resolves the session budgets and reports what it allocated. Read
those figures from the assets below rather than recomputing them, because the sizing model is tuned per release and any
formula reproduced in this file would drift out of agreement with the library running the batch.

| Asset                                                 | Reports                                                                          |
|-------------------------------------------------------|----------------------------------------------------------------------------------|
| `prepare_log_processing_batch_tool` → `jobs[]`        | Per job: `core_weight`, `memory_mb`, `message_count`, `archive_bytes`, `modeled` |
| `execute_log_processing_jobs_tool` → return value     | Resolved `core_budget`, `memory_budget_mb`, `pool_size`, and `job_allocations[]` |
| `estimate_archive_job_memory_mb(archive_path, cores)` | `(memory_mb, modeled)` for one archive, without preparing a batch                |

Use the first to plan a batch and the second to report what the batch actually committed. The third sizes memory from
the archive's size on disk at a core count the caller supplies. It therefore answers "how much memory would this job
hold at N cores" rather than "how wide should this job be", and it serves planning outside the batch tools alone.

---

## How the model behaves

Read the absolute figures from the assets above rather than quoting any number from this file:

- Three caps bound a job's width and the narrowest of them wins: `CONTROLLER_EXTRACTION_JOB_CORES`, the ceiling the
  budget or the caller sets, and the workers the archive's own message count repays. Read the width a job actually
  received from its `core_weight` (prepare) and `cores` (execute), never from the budget or the ceiling requested.
- Execution re-derives `core_weight` and re-estimates `memory_mb` for every descriptor against this session's own
  budgets, so quote the execute figures whenever they disagree with the ones preparation stamped.
- Both budgets auto-resolve from the host when left at `-1`, holding back a reserve for host operations. An explicit
  positive `core_budget` is honored up to the host's logical core count, and `memory_budget_mb` is taken verbatim and is
  never capped against host RAM, so an over-large value invites the host to kill workers.
- `pool_size` follows the smallest of the job count, the core budget, and the job bodies the memory budget can hold, and
  every idle slot is charged a spawned child's baseline against that budget. Size a batch from `pool_size` and the
  budget together, never from the per-job `memory_mb` estimates on their own.
- Queued jobs are admitted heaviest first whenever their cores and memory both fit the remaining budget, so lighter jobs
  backfill the capacity a heavier one leaves spare. One job larger than the whole budget is forced through alone, and
  only while nothing else is running, so a second oversized job waits for the batch to drain.
- A job whose estimate passes the host's **total physical memory** is failed before it is ever admitted, with an
  `error_message` naming both figures. Resetting it re-runs the same check, so the remedy is a host with more memory or
  a smaller archive, never a different budget.

The forced-alone admission is the earliest out-of-memory indicator a batch produces, and its WARNING is invisible over
the stdio MCP transport. On the MCP path, infer the condition from a `job_allocations` entry whose `memory_mb` passes
the returned `memory_budget_mb`. Reduce `core_budget` to narrow both the concurrent set and the width any single job
receives, and `memory_budget_mb` to narrow the concurrent set alone.
