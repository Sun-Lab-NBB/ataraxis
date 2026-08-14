---
name: platformio-config
description: >-
  Applies platformio.ini and library.json conventions when creating or modifying PlatformIO C++ project and library
  configuration files. Covers per-board environments, build flags, pinned lib_deps, library.json metadata, the
  lib_deps<->dependencies mirroring rule, the main.cpp export exclusion, and library.json as the single source of the
  C++ library version. Use when creating or modifying these files, adding a board or dependency, or when asked about
  PlatformIO conventions.
user-invocable: false
---

# PlatformIO configuration style guide

Applies conventions for the PlatformIO configuration files a C++ microcontroller project ships: `platformio.ini` (local
build, test, and upload configuration) and, for a library, `library.json` (the published PlatformIO registry manifest).
This is the C++ PlatformIO analogue of `/pyproject-style` and `/tox-config`.

You MUST read this skill and load the reference templates before creating or modifying either file. You MUST verify your
changes against the checklist before submitting.

---

## Scope

**Covers:**
- `platformio.ini` per-board `[env:<board>]` sections, the standard field set, build flags/unflags
- `lib_deps` dependency pinning and the registry-owner naming convention
- `library.json` metadata, `headers`, `dependencies`, `export`, and `build` fields
- The `lib_deps` <-> `dependencies` mirroring rule and per-dependency `platforms` scoping
- `library.json` `version` as the single source of the C++ library version

**Does not cover:**
- C++ source code style (see `/cpp-style`)
- Firmware Module/Kernel implementation (see `microcontroller:firmware-module`)
- Project directory structure and which files belong where (see `/project-layout`)
- Python `pyproject.toml` / `tox.ini` configuration (see `/pyproject-style`, `/tox-config`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Read this skill

Read this entire file. The conventions below apply to ALL ataraxis PlatformIO libraries.

### Step 2: Load the reference templates

Load [config-templates.md](references/config-templates.md) for the full annotated `platformio.ini` and `library.json`
examples when creating either file from scratch or adding a board/dependency.

### Step 3: Apply conventions

Write or modify the file following all conventions from this file and the loaded template. When adding or changing a
dependency, apply the mirroring rule (below) to BOTH files in the same change.

### Step 4: Verify compliance

Complete the verification checklist at the end of this file. Every item must pass.

---

## platformio.ini conventions

`platformio.ini` declares one build environment per supported board. It governs local builds, unit tests, and uploads.
It carries no version field, because the library version lives only in `library.json`.

### Environment sections

Declare one `[env:<board>]` section per supported board, named for the board (`teensy41`, `due`, `mega`). Each section
uses this field set:

| Field             | Convention                                                                       |
|-------------------|----------------------------------------------------------------------------------|
| `platform`        | The board's PlatformIO platform (`teensy`, `atmelsam`, `atmelavr`)               |
| `board`           | The PlatformIO board id (`teensy41`, `due`, `megaatmega2560`)                    |
| `framework`       | `arduino`                                                                        |
| `monitor_speed`   | Serial monitor baud rate for that board (board-specific, e.g. `115200`)          |
| `test_framework`  | `unity`                                                                          |
| `upload_protocol` | Set when the board needs a non-default uploader (e.g. `teensy-cli` for Teensy)   |
| `build_unflags`   | Add `-std=gnu++11` on boards whose Arduino core pins an older standard (AVR/SAM) |
| `build_flags`     | `-std=c++17` (the project C++ standard)                                          |
| `lib_deps`        | Pinned third-party and ataraxis dependencies (see below)                         |

Every `[env:<board>]` section sets `platform`, `board`, `framework`, `monitor_speed`, `test_framework`, and
`build_flags`. `upload_protocol`, `build_unflags`, and `lib_deps` appear only on the boards that need them, so a board
with no dependencies omits `lib_deps` entirely. `build_unflags = -std=gnu++11` is required only where the board's
default toolchain would otherwise override `build_flags` (the AVR `mega` and SAM `due` cores). Teensy needs no unflag.
Sections appear in the order of the board table below (`teensy41`, `due`, `mega`), and the keys inside a section follow
the field-table order above. `build_unflags` comes before `build_flags`, because the unflag is what lets the flag take
effect.

### Board / platform mapping

| Board id          | platform   |
|-------------------|------------|
| `teensy41`        | `teensy`   |
| `due`             | `atmelsam` |
| `megaatmega2560`  | `atmelavr` |

### lib_deps pinning

List each dependency as `registry_owner/name@^MAJOR.MINOR.PATCH`. The registry owner is the lowercase PlatformIO
registry account, NOT the GitHub org, so ataraxis libraries are published under `inkaros` (e.g.
`inkaros/ataraxis-transport-layer-mc@^4.0.1`). Third-party deps use their own owners (`arminjo/digitalWriteFast@^1.3.1`,
`pfeerick/elapsedMillis@^1.0.6`). Use the caret (`^`) range so patch/minor updates are accepted. List a dependency only
under the boards that need it.

---

## library.json conventions

`library.json` is the PlatformIO registry manifest, the file consumers receive when they add the library via `lib_deps`.
Pin `$schema` to the official PlatformIO schema URL and order fields as in the template.

| Field          | Convention                                                                                                   |
|----------------|--------------------------------------------------------------------------------------------------------------|
| `$schema`      | `https://raw.githubusercontent.com/platformio/platformio-core/develop/platformio/assets/schema/library.json` |
| `name`         | The library name (matches the repository)                                                                    |
| `version`      | The library version, the SINGLE source of truth (see Versioning)                                             |
| `description`  | One sentence, matching the README/repository description                                                     |
| `keywords`     | Comma-separated discovery keywords                                                                           |
| `repository`   | `{ "type": "git", "url": "https://github.com/Sun-Lab-NBB/<name>" }`                                          |
| `homepage`     | The Netlify API-docs URL                                                                                     |
| `authors`      | Author objects, where the primary author carries `"maintainer": true`                                        |
| `license`      | `Apache-2.0`                                                                                                 |
| `frameworks`   | `["arduino"]`                                                                                                |
| `platforms`    | Every platform the library supports (`["atmelavr", "atmelsam", "teensy"]`)                                   |
| `headers`      | The PUBLIC header(s) consumers include, as a string for one and an array for several                         |
| `dependencies` | Published dependency list that MUST mirror `lib_deps` (see mirroring rule)                                   |
| `export`       | `include` (`./examples/*`, `./src/*`) and `exclude` (`./src/main.cpp`)                                       |
| `build`        | `{ "flags": "-std=c++17" }`, mirroring `platformio.ini` `build_flags`                                        |

`$schema`, `name`, `version`, `description`, `repository`, `authors`, `license`, `frameworks`, `platforms`, `headers`,
`export`, and `build` are required in every manifest. `keywords` is included for registry discovery, `homepage` when
hosted API documentation exists, and `dependencies` exactly when some `[env]` section carries `lib_deps`. The
`description` opens with a bare third-person imperative verb and repeats the README one-line description and the GitHub
repository description verbatim, with no library-name prefix and no "This library..." opener.

### headers

`headers` lists only the PUBLIC headers a consumer includes (e.g. `["kernel.h", "communication.h", "module.h",
"axmc_shared_assets.h"]`), not every file under `src/`. Use a bare string when the library exposes a single header
(`"transport_layer.h"`).

### export and the main.cpp exclusion

`export.include` ships `./examples/*` and `./src/*`, and `export.exclude` MUST list `./src/main.cpp`. `src/main.cpp` is
the local build/test harness (see `microcontroller:firmware-module`) and must never be shipped to consumers, even though
it lives under `src/`.

### Comment restraint

A `platformio.ini` key states its own name and value, so a comment beside it is justified only by a question the key
leaves open. Such a question is the reason a dependency is pinned to an exact version, the hardware constraint behind a
build flag, or a coupling to `library.json` that a reader would otherwise break. Do not comment a key by restating it.
The `description` field stays at the single sentence its row above prescribes, matching the README and repository
description rather than expanding on them. Comments describe the configuration as it currently stands, never the edit
that produced it. A comment's claim must be true of the key and value it sits beside and of the effect the tool actually
produces, so a value change rewrites or deletes its comment in the same edit. Comments must not carry stale references
to closed issues, removed keys, sections, environments, boards, or dependencies, superseded tool versions, or outdated
TODOs, and a comment whose referent is removed is removed or rewritten with it. Align inline comments vertically within
a section, so a run of commented settings shares one comment column.

### Line width

Every line in `platformio.ini` and `library.json` stays within the 120 character limit the project sets for its Python
code, and a single unbreakable value such as a URL, a requirement string, or the `$schema` URL is exempt.

Break a comment line only where it would otherwise pass 120 characters, and fill each line to that limit before
breaking. Comment prose wrapped at a narrower width reads as a rigid block and advertises a limit the file does not set.
The test is mechanical: a wrapped line that ends before column 100 while its next word would still fit under 120 is
re-flowed. A line ending early because the sentence or the comment block ends is already correct.

### Prose punctuation and positive description

The `description` field and any configuration comments follow the project prose rules. Prose uses only the full stop and
the comma to separate clauses. Do not use a semicolon or an em-dash (`--`, `—`, or `–`) as a separator, and use a colon
only where it is lexically appropriate. A single hyphen stays available as a list marker, in tables, and in compound
words. State what the subject does and what is currently true. Do not frame it by what it is not or what it used to be,
and keep a "not Y" contrast only when it is load-bearing because it corrects a counter-intuitive assumption, giving its
reason. Sentences over 40 words must be broken into smaller sentences at natural clause boundaries, because a long
sentence in a comment or a `description` field signals over-explanation. Every comment body and `description` field is
free of typos and grammatical errors.

---

## The lib_deps <-> dependencies mirroring rule

`platformio.ini` `lib_deps` (used for local builds/tests) and `library.json` `dependencies` (shipped to consumers) MUST
describe the same dependency set with the same owner, name, and version. Whenever you add, remove, or re-pin a
dependency, update BOTH files in the same change.

Scope each `library.json` dependency to the boards that need it with a `platforms` array that matches which `[env]`
sections list it in `lib_deps`:

- A dependency in every board's `lib_deps` -> `"platforms": ["atmelsam", "atmelavr", "teensy"]`.
- A dependency only in the `due`/`mega` `lib_deps` -> `"platforms": ["atmelsam", "atmelavr"]`.

```json
{ "owner": "pfeerick", "name": "elapsedMillis", "version": "^1.0.6", "platforms": ["atmelsam", "atmelavr"] }
```

---

## Versioning

`library.json` `version` is the SINGLE source of truth for the C++ library version. Releases are cut from it, and there
is no Python-style single-sourcing helper. `platformio.ini` carries no version field. When releasing, bump
`library.json` `version` (see `/release`).

---

## pio command reference

These are the PlatformIO development-automation commands (agent-runnable, the C++ analogue of `tox` envs). The PR gate
for a PlatformIO library is `tox` (docs) plus `pio check` and `pio test`.

| Command                | Purpose                                                       |
|------------------------|---------------------------------------------------------------|
| `pio project init`     | Scaffold/refresh the PlatformIO project from `platformio.ini` |
| `pio project metadata` | Emit resolved per-env build metadata (IDE/index integration)  |
| `pio run`              | Build the firmware/library for the selected environment(s)    |
| `pio test`             | Run the Unity unit tests on the selected environment(s)       |
| `pio check`            | Run static analysis on the source                             |

---

## Related skills

| Skill              | Relationship                                                         |
|--------------------|----------------------------------------------------------------------|
| `/project-layout`  | Owner of where platformio.ini/library.json sit in the C++ archetypes |
| `/cpp-style`       | C++ source style (scopes out these config files)                     |
| `/pyproject-style` | The Python metadata analogue that library.json mirrors               |
| `/tox-config`      | The Python automation analogue that the pio commands mirror          |
| `/readme-style`    | C++ PlatformIO README Installation/Developers sections               |
| `/release`         | Uses library.json version as the C++ library release version         |
| `/commit`          | Should be invoked after platformio.ini/library.json changes          |

---

## Proactive behavior

You should proactively offer to invoke this skill when:
- Creating a new PlatformIO C++ library or firmware project that needs a platformio.ini or library.json
- Adding a board environment, or adding, removing, or re-pinning a lib_deps dependency
- Releasing a C++ library (library.json version is the single source of the release version)
- The user asks about PlatformIO configuration or the lib_deps<->dependencies mirroring rule

---

## Verification checklist

**You MUST verify your edits against this checklist before submitting any changes to platformio.ini or library.json
files.**

```text
PlatformIO Configuration Compliance:

Judgment items. No tool inspects these, so this checklist is their only enforcement. Walk every one against the file you
wrote. Only parse validity, the library.json $schema, and what pio check and pio test reject are tool-settled.
- [ ] platformio.ini has one [env:<board>] section per supported board, named for the board
- [ ] Each env sets platform, board, framework=arduino, monitor_speed, test_framework=unity, build_flags=-std=c++17
- [ ] library.json carries every required field ($schema, name, version, description, repository, authors, license,
      frameworks, platforms, headers, export, build)
- [ ] build_unflags=-std=gnu++11 present on AVR/SAM boards that need it; upload_protocol set where required
- [ ] [env:<board>] sections follow the board-table order, and keys inside each section follow the field-table order
      (build_unflags before build_flags)
- [ ] lib_deps entries are pinned as registry_owner/name@^X.Y.Z (ataraxis libs under the inkaros owner)
- [ ] library.json pins $schema and orders fields per the template
- [ ] headers lists only the public consumer-facing header(s)
- [ ] export.include ships ./examples/* and ./src/*; export.exclude lists ./src/main.cpp
- [ ] library.json dependencies mirror platformio.ini lib_deps (same owner/name/version)
- [ ] each library.json dependency's platforms array matches the boards that list it in lib_deps
- [ ] library.json version is set (the single source of the C++ library version); platformio.ini has no version
- [ ] description is the single sentence matching the README and repository description
- [ ] Every description field opens with a bare third-person imperative verb (no name prefix, no
      "This environment/library..." opener)
- [ ] Every comment answers a question its key leaves open (no comment restating the key)
- [ ] Every comment's claim is true of the value it sits beside and of the effect the tool actually produces
- [ ] Comments record current configuration only, never the edit that produced it
- [ ] No stale references in comments (closed issues, removed keys or environments, superseded tool versions,
      outdated TODOs)
- [ ] Inline comments aligned vertically within their section
- [ ] Sentences in comments and description fields stay under 40 words
- [ ] Comments and description fields free of typos and grammar errors
- [ ] Lines stay under 120 characters, with unbreakable single values (URLs, requirement strings, $schema) exempt
- [ ] Comments and descriptions fill each line to 120 characters, with no line ending before column 100 while its next
      word would still fit
- [ ] Prose separators are full stops and commas only, no semicolons or em-dashes (colons, hyphen bullets, and code
      syntax exempt)
- [ ] Prose states what the configuration does, not what it is not or used to be (contrast only when load-bearing)
```
