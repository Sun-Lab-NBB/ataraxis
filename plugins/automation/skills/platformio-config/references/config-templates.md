# PlatformIO configuration templates

Full annotated templates for `platformio.ini` and `library.json`. Adapt the board set, headers, and dependencies to the
library, and keep field ordering and the mirroring rule.

---

## platformio.ini

One `[env:<board>]` section per supported board. The Teensy env needs no `build_unflags`. The AVR (`mega`) and SAM
(`due`) envs add `build_unflags = -std=gnu++11` so `build_flags = -std=c++17` takes effect. `lib_deps` is omitted from a
board that has no dependencies. The `check_flags` header filter carries the repository name, and `--target` carries the
board's clang triple, so both change per repository and per board.

```ini
; The clang-tidy check requires the --config flag, as PlatformIO appends '--checks=*' when it is absent and that
; discards the curated check list in .clang-tidy. The header filter narrows reporting to this repository's own headers,
; keeping out the framework, toolchain, and dependency headers that the '.*' filter in .clang-tidy admits.
; --target names the board's architecture, so clang applies the pointer and integer widths the firmware compiles
; against. -ferror-limit=0 keeps clang parsing to the end of the translation unit. Without it clang stops at the
; twentieth error and analyses a truncated syntax tree, which reports findings the source does not contain.
; The two -Wno flags silence the GCC libstdc++ headers, which the bundled clang 15 cannot fully parse. A compiler
; diagnostic bypasses the header filter, so it has to be turned off at source.

[env:teensy41]
platform = teensy
board = teensy41
framework = arduino
monitor_speed = 115200
test_framework = unity
upload_protocol = teensy-cli
build_flags = -std=c++17
check_tool = clangtidy
check_flags =
    clangtidy: --config-file=.clang-tidy
    clangtidy: --header-filter=.*/<repository>/src/.*
    clangtidy: --extra-arg=--target=arm-none-eabi
    clangtidy: --extra-arg=-ferror-limit=0
    clangtidy: --extra-arg=-Wno-invalid-constexpr
    clangtidy: --extra-arg=-Wno-unusable-partial-specialization
lib_deps =
    arminjo/digitalWriteFast@^1.3.1
    inkaros/ataraxis-transport-layer-mc@^4.0.1

[env:due]
platform = atmelsam
board = due
framework = arduino
monitor_speed = 5250000
test_framework = unity
build_unflags = -std=gnu++11
build_flags = -std=c++17
check_tool = clangtidy
check_flags =
    clangtidy: --config-file=.clang-tidy
    clangtidy: --header-filter=.*/<repository>/src/.*
    clangtidy: --extra-arg=--target=arm-none-eabi
    clangtidy: --extra-arg=-ferror-limit=0
    clangtidy: --extra-arg=-Wno-invalid-constexpr
    clangtidy: --extra-arg=-Wno-unusable-partial-specialization
lib_deps =
    pfeerick/elapsedMillis@^1.0.6
    arminjo/digitalWriteFast@^1.3.1
    inkaros/ataraxis-transport-layer-mc@^4.0.1

[env:mega]
platform = atmelavr
board = megaatmega2560
framework = arduino
monitor_speed = 1000000
test_framework = unity
build_unflags = -std=gnu++11
build_flags = -std=c++17
check_tool = clangtidy
check_flags =
    clangtidy: --config-file=.clang-tidy
    clangtidy: --header-filter=.*/<repository>/src/.*
    clangtidy: --extra-arg=--target=avr
    clangtidy: --extra-arg=-ferror-limit=0
    clangtidy: --extra-arg=-Wno-invalid-constexpr
    clangtidy: --extra-arg=-Wno-unusable-partial-specialization
lib_deps =
    pfeerick/elapsedMillis@^1.0.6
    arminjo/digitalWriteFast@^1.3.1
    inkaros/ataraxis-transport-layer-mc@^4.0.1
```

---

## library.json

The published registry manifest. `dependencies` mirrors the union of all `lib_deps` above. Each dependency's `platforms`
array names the boards whose `lib_deps` list it (`elapsedMillis` is only in `due`/`mega`, so it is scoped to
`atmelsam`/`atmelavr`, while `digitalWriteFast` and the ataraxis library are in every board, so they list all three).
`export.exclude` drops `./src/main.cpp`. `headers` lists only the public consumer headers. Use a bare string for a
single-header library.

```json
{
  "$schema": "https://raw.githubusercontent.com/platformio/platformio-core/develop/platformio/assets/schema/library.json",
  "name": "ataraxis-micro-controller",
  "version": "4.0.2",
  "description": "Provides the framework for integrating custom hardware modules with a centralized PC control interface.",
  "keywords": "arduino, teensy, ataraxis, communication, serial, integration, hardware modules",
  "repository": {
    "type": "git",
    "url": "https://github.com/Sun-Lab-NBB/ataraxis-micro-controller"
  },
  "homepage": "https://ataraxis-micro-controller-api-docs.netlify.app/",
  "authors":
  [
    {
      "name": "Ivan Kondratyev",
      "url": "https://github.com/Inkaros",
      "email" : "ik278@cornell.edu",
      "maintainer": true
    }
  ],
  "license": "Apache-2.0",
  "frameworks": ["arduino"],
  "platforms": ["atmelavr", "atmelsam", "teensy"],
  "headers": ["kernel.h", "communication.h", "module.h", "axmc_shared_assets.h"],
  "dependencies":
  [
    {
      "owner": "pfeerick",
      "name": "elapsedMillis",
      "version": "^1.0.6",
      "platforms": ["atmelsam", "atmelavr"]
    },
    {
      "owner": "arminjo",
      "name": "digitalWriteFast",
      "version": "^1.3.1",
      "platforms": ["atmelsam", "atmelavr", "teensy"]
    },
    {
      "owner": "inkaros",
      "name": "ataraxis-transport-layer-mc",
      "version": "^4.0.1",
      "platforms": ["atmelsam", "atmelavr", "teensy"]
    }
  ],
  "export": {
    "include":
    [
      "./examples/*",
      "./src/*"
    ],
    "exclude":
    [
      "./src/main.cpp"
    ]
  },
  "build": {
    "flags": "-std=c++17"
  }
}
```

For a single-header library, `headers` is a bare string instead of an array:

```json
"headers": "transport_layer.h",
```
