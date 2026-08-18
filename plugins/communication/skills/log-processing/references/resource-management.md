# Resource management

Records how the library sizes each log processing job, and which returned figure carries each sizing decision.

---

## Where the sizing figures come from

Every job is sized by the library, never by the agent. Preparation reads each archive's zip directory once and stamps
the resulting figures onto the job entry, and execution resolves the session budgets and reports what it allocated. Read
those figures from the assets below rather than recomputing them, because the sizing model is tuned per release and any
formula reproduced in this file would drift out of agreement with the library running the batch.

| Asset                                             | Reports                                                                          |
|---------------------------------------------------|----------------------------------------------------------------------------------|
| `prepare_log_processing_batch_tool` → `jobs[]`    | Per job: `core_weight`, `memory_mb`, `message_count`, and `archive_bytes`        |
| `execute_log_processing_jobs_tool` → return value | Resolved `core_budget`, `memory_budget_mb`, `pool_size`, and `job_allocations[]` |
| `size_archive_job(archive_path)`                  | `JobSizing(cores, memory_mb)` for one archive, without preparing a batch         |

Use the first to plan a batch and the second to report what the batch actually committed. The third reads one archive
and answers both halves of the sizing model from that single read, so a caller planning outside the batch tools
reproduces neither the width rule nor the memory model.

---

## How the model behaves

- A job's width takes one of two values with nothing between them. An archive holding fewer data messages than
  `_PARALLEL_EXTRACTION_THRESHOLD` opens no pool and takes a single core, and every archive at or above it takes the
  declared `CONTROLLER_EXTRACTION_JOB_CORES` allocation. Execution then collapses that width onto the session's core
  budget wherever the budget is narrower, which is the only narrowing the model applies. Read the width a job actually
  received from its `core_weight` (prepare) and `cores` (execute), never from the budget requested.
- The `PARALLEL_PROCESSING_THRESHOLD` named elsewhere decides how the archive reader batches messages inside a pool, and
  it carries a different value, so a claim about one threshold settles nothing about the other.
- Execution re-derives `core_weight` and re-estimates `memory_mb` for every descriptor against this session's own
  budgets, so quote the execute figures whenever they disagree with the ones preparation stamped.
- Both budgets auto-resolve from the host when left at `-1`, holding back a reserve for host operations. An explicit
  positive `core_budget` is honored up to the host's logical core count, and `memory_budget_mb` is taken verbatim and is
  never capped against host RAM, so an over-large value invites the host to kill workers.
- `pool_size` follows the smallest of the job count, the core budget, and the job bodies half the memory budget can
  hold, and never falls below one. Every idle slot is charged a spawned child's baseline against that half. Size a
  batch from `pool_size` and the budget together, never from the per-job `memory_mb` estimates on their own.
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
