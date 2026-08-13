# Staged read tools

Records the three-stage response contract `discover_camera_data_tool` and `get_batch_status_overview_tool` share, and
the parameters and return fields each of them carries.

---

## The three stages

Both tools widen in the same three steps, and a caller pays for only the stage it asks for.

| Stage       | Requested by                        | Carries                                                       |
|-------------|-------------------------------------|---------------------------------------------------------------|
| Bare        | The root directory alone            | Counts, a `breakdown` per filterable axis, and nothing listed |
| Semi-detail | Any filter, or `include_items=True` | A page of items carrying their identity, plus paging fields   |
| Detail      | `detailed=True` alongside either    | The per-item fields that grow a whole-project read fastest    |

**A bare call lists nothing.** It carries no paging fields, and it carries no item list wherever the scan found
anything, so an agent that reads `sources` or `output_directories` straight out of a bare response reads a key that is
absent. Ask for the listing explicitly whenever you intend to render one.

---

## Shared fields and rules

- `breakdown` maps each filterable axis to its value counts, and it answers "which source IDs, camera names, or statuses
  are present and how many of each" without listing anything. Read filter values from here rather than from a page.
- `rows`, `matched_rows`, `start_row`, and `next_start_row` accompany every listed page. `matched_rows` counts the
  filter matches before the page cap, so a `rows` below it means the page was capped.
- Walk a long result by following `next_start_row` until it reads null. A page that fills its own limit exactly may
  still end the matches, so the null is the terminator rather than a short page.
- `limit` defaults to 200, or to 50 under `detailed`. A value at or below zero lifts the cap and returns every match
  from the requested start.
- The counts and the `breakdown` span every discovered item whatever the filters name, so narrowing what is listed
  never distorts what is reported.
- A filter naming a value the scan did not find returns an error dictionary naming what is available, rather than an
  empty page. The one exception is `discover_camera_data_tool` over a root that confirms no source at all, which
  returns an empty `sources` list and an empty `breakdown` whatever the filters name. Outside that case an empty page
  means the filters matched nothing that also survived paging, never a typo.
- A listed item omits any field holding nothing, so an unmatched video file and an unprocessed source carry no
  `video_file` and no `timestamps_file` key even under `detailed`. Read an absent key as the field holding nothing.
- `detailed` widens the fields a listed item carries and never asks for the listing itself, so it is passed alongside a
  filter or `include_items` rather than on its own.

---

## discover_camera_data_tool

| Parameter        | Type               | Default    | Description                                                 |
|------------------|--------------------|------------|-------------------------------------------------------------|
| `root_directory` | `str`              | (required) | Absolute path to the root directory to search recursively   |
| `source_ids`     | `list[str] / None` | `None`     | Restricts the listing to these camera source IDs            |
| `name`           | `str / None`       | `None`     | Restricts the listing to one camera name                    |
| `limit`          | `int / None`       | `None`     | Sources to list. Defaults to 200, or to 50 under `detailed` |
| `start_row`      | `int`              | `0`        | Match index to begin the listing at                         |
| `include_items`  | `bool`             | `False`    | Keyword-only. Lists sources when no filter is named         |
| `detailed`       | `bool`             | `False`    | Keyword-only. Widens each listed source with its paths      |

**Return structure:**
```text
log_directories:        Flat list of log directory paths (pass directly to the prepare tool)
total_sources:          Number of confirmed source entries
total_log_directories:  Number of log directories with archives
breakdown{}:            Counts spanning every confirmed source:
  source_id{}:          Source ID string -> sources carrying it
  name{}:               Camera name -> sources carrying it
sources[]:              Present under a filter, under `include_items`, and on a scan confirming no source:
  recording_root:       Path to the recording root directory
  source_id:            Source ID string (the camera's system_id)
  name:                 Camera name from manifest
  log_directory:        Absolute path to the DataLogger output directory
  log_archive:          Absolute path to the .npz archive. Present only under `detailed`
  video_file:           Absolute path to the matched video file. Present only under `detailed`, and only when matched
  timestamps_file:      Absolute path to the processed feather file. Present only under `detailed`, and only when
                        the source has been processed
rows:                   Sources this response carries. Accompanies a requested listing
matched_rows:           Sources matching the filters, before the page cap
start_row:              Match index this page begins at
next_start_row:         Start row of the following page, or null at the end of the matches
```

`log_directories`, the counts, and the `breakdown` are unconditional, so a bare call is enough to prepare a batch and to
name every camera the scan found. Reach for the listing when a table needs each source's own row, and for `detailed`
when it needs the archive, video, or feather paths that verification and analysis consume.

---

## get_batch_status_overview_tool

| Parameter        | Type               | Default    | Description                                                       |
|------------------|--------------------|------------|-------------------------------------------------------------------|
| `root_directory` | `str`              | (required) | Absolute path to the root directory to search for trackers        |
| `statuses`       | `list[str] / None` | `None`     | Restricts the listing to directories carrying these status labels |
| `limit`          | `int / None`       | `None`     | Directories to list. Defaults to 200, or to 50 under `detailed`   |
| `start_row`      | `int`              | `0`        | Match index to begin the listing at                               |
| `include_items`  | `bool`             | `False`    | Keyword-only. Lists directories when no status is named           |
| `detailed`       | `bool`             | `False`    | Keyword-only. Adds `tracker_path`, `jobs`, and `error`            |

**Return structure:**
```text
total_output_directories:  Directories holding a tracker
summary{}:                 Aggregate job counts across every directory: `succeeded`, `failed`, `running`, `scheduled`
breakdown{}:               Counts spanning every discovered directory:
  status{}:                Status label -> directories carrying it
output_directories[]:      Present under a status filter and under `include_items`:
  output_directory:        Absolute path to the `camera_timestamps/` subdirectory holding the tracker
  status:                  One of the six labels the SKILL.md "Status formatting" section lists
  summary{}:               That directory's own job counts
  tracker_path:            Absolute path to the tracker YAML. Present only under `detailed`
  jobs[]:                  That directory's per-job entries. Present only under `detailed`
  error:                   Present only under `detailed`, and only where the tracker could not be read
rows:                      Directories this response carries. Accompanies a requested listing
matched_rows:              Directories matching the filters, before the page cap
start_row:                 Match index this page begins at
next_start_row:            Start row of the following page, or null at the end of the matches
```

The `breakdown` answers the routing question a resumed batch asks, which is how many directories are `failed`,
`not_started`, or still `processing`, so read it before deciding whether to list anything. Name the statuses that
matter, or pass `include_items=True`, when the answer needs the directories themselves.
