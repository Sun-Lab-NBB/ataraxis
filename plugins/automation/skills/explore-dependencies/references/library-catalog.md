# Ataraxis library catalog

Static reference for the ataraxis and Sun Lab library ecosystem. Use this catalog to understand
which library covers which domain, resolve import names, and identify dependencies to explore.

---

## Ataraxis libraries

Nine core libraries maintained under the Sun-Lab-NBB GitHub organization.

### Python libraries

| Library                          | PyPI package                       | Import name                      | Abbr   |
|----------------------------------|------------------------------------|----------------------------------|--------|
| ataraxis-base-utilities          | `ataraxis-base-utilities`          | `ataraxis_base_utilities`        | axbu   |
| ataraxis-time                    | `ataraxis-time`                    | `ataraxis_time`                  | axt    |
| ataraxis-data-structures         | `ataraxis-data-structures`         | `ataraxis_data_structures`       | axds   |
| ataraxis-video-system            | `ataraxis-video-system`            | `ataraxis_video_system`          | axvs   |
| ataraxis-transport-layer-pc      | `ataraxis-transport-layer-pc`      | `ataraxis_transport_layer_pc`    | axtlpc |
| ataraxis-communication-interface | `ataraxis-communication-interface` | `ataraxis_communication_interface`| axci  |
| ataraxis-automation              | `ataraxis-automation`              | `ataraxis_automation`            | axa    |

### C++ libraries (PlatformIO)

These are PlatformIO dependencies for Arduino/Teensy firmware projects, not Python packages. They
cannot be explored with `python -c "import ..."`. Instead, locate them in the PlatformIO
`.pio/libdeps/` directory.

| Library                     | PlatformIO name                             | Abbr  |
|-----------------------------|---------------------------------------------|-------|
| ataraxis-transport-layer-mc | `Sun-Lab-NBB/ataraxis-transport-layer-mc`   | axtlmc|
| ataraxis-micro-controller   | `Sun-Lab-NBB/ataraxis-micro-controller`     | axmc  |

### Python + C++ extension

`ataraxis-time` includes C++ extension modules built with nanobind and scikit-build-core. The
Python import (`ataraxis_time`) exposes the C++ bindings through its public API, so the standard
Python exploration workflow applies.

---

## GitHub repositories

All repositories are under `https://github.com/Sun-Lab-NBB/`:

| Library                          | Repository URL                                                        |
|----------------------------------|-----------------------------------------------------------------------|
| ataraxis-base-utilities          | https://github.com/Sun-Lab-NBB/ataraxis-base-utilities                |
| ataraxis-time                    | https://github.com/Sun-Lab-NBB/ataraxis-time                          |
| ataraxis-data-structures         | https://github.com/Sun-Lab-NBB/ataraxis-data-structures               |
| ataraxis-video-system            | https://github.com/Sun-Lab-NBB/ataraxis-video-system                  |
| ataraxis-transport-layer-pc      | https://github.com/Sun-Lab-NBB/ataraxis-transport-layer-pc            |
| ataraxis-transport-layer-mc      | https://github.com/Sun-Lab-NBB/ataraxis-transport-layer-mc            |
| ataraxis-communication-interface | https://github.com/Sun-Lab-NBB/ataraxis-communication-interface       |
| ataraxis-micro-controller        | https://github.com/Sun-Lab-NBB/ataraxis-micro-controller              |
| ataraxis-automation              | https://github.com/Sun-Lab-NBB/ataraxis-automation                    |

---

## Sun Lab downstream libraries

Downstream libraries follow the `sl-*` naming pattern. These are application-level libraries
built on top of the ataraxis infrastructure for specific lab workflows:

- `sl-experiment` — Experiment orchestration and session management
- `sl-suite2p` — Neural imaging analysis pipeline (reimplemented suite2p)
- Other `sl-*` packages — Lab-specific analysis and acquisition tools

These follow the same exploration workflow as ataraxis libraries: match `sl-` prefixed package
names in `pyproject.toml`, resolve with `python -c "import ..."`, and read their `__all__` exports.

---

## Domain-to-library quick reference

Use this table to determine which library to explore when the task involves a specific domain.

| Domain                       | Library                          | Import name                        | Abbr   |
|------------------------------|----------------------------------|------------------------------------|--------|
| Console output, errors       | ataraxis-base-utilities          | `ataraxis_base_utilities`          | axbu   |
| Progress tracking            | ataraxis-base-utilities          | `ataraxis_base_utilities`          | axbu   |
| List/array/filesystem utils  | ataraxis-base-utilities          | `ataraxis_base_utilities`          | axbu   |
| Byte serialization           | ataraxis-base-utilities          | `ataraxis_base_utilities`          | axbu   |
| Worker/job resolution        | ataraxis-base-utilities          | `ataraxis_base_utilities`          | axbu   |
| Precision timing, delays     | ataraxis-time                    | `ataraxis_time`                    | axt    |
| Timestamps, conversion       | ataraxis-time                    | `ataraxis_time`                    | axt    |
| YAML config, shared memory   | ataraxis-data-structures         | `ataraxis_data_structures`         | axds   |
| Data logging                 | ataraxis-data-structures         | `ataraxis_data_structures`         | axds   |
| Camera interfaces            | ataraxis-video-system            | `ataraxis_video_system`            | axvs   |
| Serial communication (PC)    | ataraxis-transport-layer-pc      | `ataraxis_transport_layer_pc`      | axtlpc |
| MC communication             | ataraxis-communication-interface | `ataraxis_communication_interface` | axci   |
| Dev automation               | ataraxis-automation              | `ataraxis_automation`              | axa    |
