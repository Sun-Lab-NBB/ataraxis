# Staged read tools

Records the three-stage response contract `discover_microcontroller_data_tool` and `get_batch_status_overview_tool`
share, and the parameters and return fields each of them carries.

---

## The three stages

Both tools widen in the same three steps, and a caller pays for only the stage it asks for.

| Stage       | Requested by                        | Carries                                                       |
|-------------|-------------------------------------|---------------------------------------------------------------|
| Bare        | The root directory alone            | Counts, a `breakdown` per filterable axis, and nothing listed |
| Semi-detail | Any filter, or `include_items=True` | A page of items carrying their identity, plus paging fields   |
| Detail      | `detailed=True`                     | The per-item field that grows a whole-project read fastest    |

**A bare call lists nothing.** It returns no item list and no paging fields at all, so an agent that reads `sources` or
`log_directories` straight out of a bare response reads a key that is absent. Ask for the listing explicitly whenever
you intend to render one.

---

## Shared fields and rules

- `breakdown` maps each filterable axis to its value counts, and it answers "which IDs, names, or statuses are present
  and how many of each" without listing anything. Read filter values from here rather than from a listed page.
- `rows`, `matched_rows`, `start_row`, and `next_start_row` accompany every listed page. `matched_rows` counts the
  filter matches before the page cap, so a `rows` below it means the page was capped.
- Walk a long result by following `next_start_row` until it reads null. A page that fills its own limit exactly may
  still end the matches, so the null is the terminator rather than a short page.
- `limit` defaults to 200, or to 50 under `detailed`. A value at or below zero lifts the cap and returns every match
  from the requested start.
- The counts and the `breakdown` span every discovered item whatever the filters name, so narrowing what is listed
  never distorts what is reported.
- A filter naming a value the scan did not find returns an error dictionary naming what is available, rather than an
  empty page. An empty page therefore means the filters matched nothing that also survived paging, never a typo.
- A listed item omits any field holding nothing, so a source with no registered modules carries no `modules` key even
  under `detailed`.

---

## discover_microcontroller_data_tool

| Parameter        | Type               | Default    | Description                                                 |
|------------------|--------------------|------------|-------------------------------------------------------------|
| `root_directory` | `str`              | (required) | Absolute path to the root directory to search recursively   |
| `source_ids`     | `list[str] / None` | `None`     | Restricts the listing to these controller IDs               |
| `name`           | `str / None`       | `None`     | Restricts the listing to one controller name                |
| `limit`          | `int / None`       | `None`     | Sources to list. Defaults to 200, or to 50 under `detailed` |
| `start_row`      | `int`              | `0`        | Match index to begin the listing at                         |
| `include_items`  | `bool`             | `False`    | Lists sources when no filter is named                       |
| `detailed`       | `bool`             | `False`    | Adds `log_archive` and `modules` to each listed source      |

**Return structure:**
```text
log_directories:        Flat list of log directory paths (pass directly to the prepare tool)
total_sources:          Number of confirmed source entries
total_log_directories:  Number of log directories with archives
breakdown{}:            Counts spanning every confirmed source:
  source_id{}:          Controller ID string -> sources carrying it
  name{}:               Controller name -> sources carrying it
sources[]:              Present under a filter, under `include_items`, and on a scan confirming no source:
  recording_root:       Path to the recording root directory
  source_id:            Source ID string (controller ID)
  name:                 Controller name from manifest
  log_directory:        Absolute path to the DataLogger output directory
  log_archive:          Absolute path to the .npz archive. Present only under `detailed`
  modules[]:            Module entries from manifest. Present only under `detailed`
rows:                   Sources this response carries. Accompanies `sources`
matched_rows:           Sources matching the filters, before the page cap
start_row:              Match index this page begins at
next_start_row:         Start row of the following page, or null at the end of the matches
```

`log_directories`, the counts, and the `breakdown` are unconditional, so a bare call is enough to prepare a batch and
to name every controller the scan found. Reach for the listing when a table needs each source's own row, and for
`detailed` when it needs the module listings.

---

## get_batch_status_overview_tool

| Parameter        | Type               | Default    | Description                                                       |
|------------------|--------------------|------------|-------------------------------------------------------------------|
| `root_directory` | `str`              | (required) | Absolute path to the root directory to search for trackers        |
| `statuses`       | `list[str] / None` | `None`     | Restricts the listing to directories carrying these status labels |
| `limit`          | `int / None`       | `None`     | Directories to list. Defaults to 200, or to 50 under `detailed`   |
| `start_row`      | `int`              | `0`        | Match index to begin the listing at                               |
| `include_items`  | `bool`             | `False`    | Lists directories when no status is named                         |
| `detailed`       | `bool`             | `False`    | Adds `tracker_path`, `jobs`, and `error` to each listed directory |

**Return structure:**
```text
total_log_directories:  Directories holding a tracker
summary{}:              Aggregate job counts across every directory: `succeeded`, `failed`, `running`, `scheduled`
breakdown{}:            Counts spanning every discovered directory:
  status{}:             Status label -> directories carrying it
log_directories[]:      Present under a status filter and under `include_items`:
  log_directory:        Absolute path to the DataLogger output directory
  status:               One of the six labels the Status formatting section lists
  summary{}:            That directory's own job counts
  tracker_path:         Absolute path to the tracker YAML. Present only under `detailed`
  jobs[]:               That directory's per-job entries. Present only under `detailed`
  error:                Present only under `detailed`, and only where the tracker could not be read
rows:                   Directories this response carries. Accompanies `log_directories`
matched_rows:           Directories matching the filters, before the page cap
start_row:              Match index this page begins at
next_start_row:         Start row of the following page, or null at the end of the matches
```

The `breakdown` answers the routing question a resumed batch asks, which is how many directories are `failed`,
`not_started`, or still `processing`, so read it before deciding whether to list anything. Name the statuses that
matter, or pass `include_items=True`, when the answer needs the directories themselves.
