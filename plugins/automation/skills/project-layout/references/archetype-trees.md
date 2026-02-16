# Archetype directory trees

Annotated directory trees for each Sun Lab project archetype. Each tree is verified against the
canonical example repository listed in its section heading.

---

## Python-only

Based on `ataraxis-automation` and `ataraxis-base-utilities`.

```text
project-root/
├── .github/
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.md             # GitHub bug report template
│       └── feature_request.md        # GitHub feature request template
├── docs/
│   ├── source/
│   │   ├── api.rst                   # API reference directives (automodule, click)
│   │   ├── conf.py                   # Sphinx configuration
│   │   ├── index.rst                 # Main page with toctree
│   │   └── welcome.rst              # Landing page content
│   ├── make.bat                      # Windows Sphinx wrapper
│   └── Makefile                      # Unix Sphinx wrapper (delegates to tox)
├── envs/
│   ├── {abbr}_dev_lin.yml            # Linux conda environment specification
│   ├── {abbr}_dev_lin_spec.txt       # Linux explicit package list
│   ├── {abbr}_dev_osx.yml            # macOS conda environment specification
│   ├── {abbr}_dev_osx_spec.txt       # macOS explicit package list
│   ├── {abbr}_dev_win.yml            # Windows conda environment specification
│   └── {abbr}_dev_win_spec.txt       # Windows explicit package list
├── src/
│   └── package_name/                 # Main package (underscore-separated)
│       ├── submodule/                # (optional) Subpackage directories
│       │   ├── __init__.py
│       │   └── module.py
│       ├── __init__.py               # Package init with public API re-exports
│       ├── __init__.pyi              # (optional) Stub file for re-exports
│       ├── module.py                 # Source modules
│       └── py.typed                  # PEP 561 marker for typed packages
├── tests/
│   ├── submodule/                    # (optional) Mirrors src/ subpackage structure
│   │   └── module_test.py
│   └── module_test.py                # Test files use _test.py suffix
├── .gitignore
├── CLAUDE.md                         # Claude Code project instructions
├── LICENSE                           # GPL-3.0 license
├── pyproject.toml                    # Build config, metadata, tool settings
├── README.md                         # Project documentation
└── tox.ini                           # Automation orchestration (lint, type, test, docs)
```

### Notes

- The `{abbr}` placeholder in `envs/` files is a short project abbreviation (e.g., `axa` for
  ataraxis-automation, `axbu` for ataraxis-base-utilities).
- The `tests/` directory mirrors the `src/package_name/` structure. Test files use the `_test.py`
  suffix (e.g., `automation_test.py`).
- The `.pypirc` file may exist locally but is not committed to version control.
- Build artifacts (`dist/`, `reports/`, `coverage.xml`) are gitignored.

---

## Python + C++ extension

Based on `ataraxis-time`.

```text
project-root/
├── .github/
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.md             # GitHub bug report template
│       └── feature_request.md        # GitHub feature request template
├── docs/
│   ├── source/
│   │   ├── doxygen/                  # Doxygen-generated XML for Breathe
│   │   │   └── xml/                  # XML output consumed by Sphinx
│   │   ├── api.rst                   # API reference (automodule + doxygenfile)
│   │   ├── conf.py                   # Sphinx config with Breathe integration
│   │   ├── index.rst                 # Main page with toctree
│   │   └── welcome.rst              # Landing page content
│   ├── make.bat                      # Windows Sphinx wrapper
│   └── Makefile                      # Unix Sphinx wrapper
├── envs/
│   ├── {abbr}_dev_lin.yml            # Linux conda environment specification
│   ├── {abbr}_dev_lin_spec.txt       # Linux explicit package list
│   ├── {abbr}_dev_osx.yml            # macOS conda environment specification
│   ├── {abbr}_dev_osx_spec.txt       # macOS explicit package list
│   ├── {abbr}_dev_win.yml            # Windows conda environment specification
│   └── {abbr}_dev_win_spec.txt       # Windows explicit package list
├── src/
│   ├── c_extensions/                 # C++ extension sources
│   │   └── module_ext.cpp            # nanobind extension (snake_case_ext.cpp)
│   ├── python_wrapper/               # Pure Python wrapper around C++ extension
│   │   ├── __init__.py
│   │   ├── wrapper_module.py         # Python class wrapping C++ class
│   │   └── wrapper_module.pyi        # Stub file for wrapper
│   ├── pure_python_module/           # (optional) Additional pure Python subpackages
│   │   ├── __init__.py
│   │   └── module.py
│   ├── __init__.py                   # Top-level package init
│   ├── __init__.pyi                  # Top-level stub file
│   ├── module_ext.pyi                # Stub for compiled C++ extension
│   └── py.typed                      # PEP 561 marker
├── tests/
│   ├── python_wrapper/               # Tests for Python wrapper
│   │   └── wrapper_test.py
│   └── pure_python_module/           # Tests for pure Python modules
│       └── module_test.py
├── .clang-format                     # C++ formatting configuration
├── .clang-tidy                       # C++ linting configuration
├── .gitignore
├── CLAUDE.md                         # Claude Code project instructions
├── CMakeLists.txt                    # CMake build config for nanobind extension
├── Doxyfile                          # Doxygen documentation configuration
├── LICENSE                           # GPL-3.0 license
├── pyproject.toml                    # scikit-build-core backend + metadata
├── README.md                         # Project documentation
└── tox.ini                           # Automation orchestration
```

### Notes

- The `src/` layout uses a flat namespace: `c_extensions/`, wrapper subpackages, and pure Python
  subpackages are all direct children of `src/`.
- C++ extension stubs (`module_ext.pyi`) live at the `src/` top level alongside `py.typed`.
- The `CMakeLists.txt` at the project root drives the nanobind build via scikit-build-core.
- Build artifacts (`build/`) contain per-Python-version subdirectories and are gitignored.

---

## C++ PlatformIO library

Based on `ataraxis-transport-layer-mc`.

```text
project-root/
├── docs/
│   ├── source/
│   │   ├── doxygen/                  # Doxygen-generated XML for Breathe
│   │   │   └── xml/                  # XML output consumed by Sphinx
│   │   ├── api.rst                   # API reference (doxygenfile directives)
│   │   ├── conf.py                   # Sphinx config with Breathe integration
│   │   ├── index.rst                 # Main page with toctree
│   │   └── welcome.rst              # Landing page content
│   ├── make.bat                      # Windows Sphinx wrapper
│   └── Makefile                      # Unix Sphinx wrapper
├── examples/                         # PlatformIO example sketches
│   └── example_sketch.cpp            # Runnable usage examples
├── src/                              # Header-only library source
│   ├── main.cpp                      # Development entry point (excluded from library)
│   ├── primary_header.h              # Primary library header
│   ├── supporting_header.h           # Supporting module headers
│   └── shared_assets.h              # Shared types and constants
├── test/                             # PlatformIO native tests
│   └── test_component.cpp            # Unity framework test files
├── .clang-format                     # C++ formatting configuration
├── .clang-tidy                       # C++ linting configuration
├── .gitignore
├── Doxyfile                          # Doxygen documentation configuration
├── library.json                      # PlatformIO library manifest
├── LICENSE                           # GPL-3.0 license
├── platformio.ini                    # PlatformIO build configuration
├── README.md                         # Project documentation
└── tox.ini                           # Documentation build automation
```

### Notes

- All library code is **header-only** (`.h` files only, no `.cpp` implementation files).
- The `main.cpp` in `src/` is a development entry point used for testing during development. It
  is excluded from the distributed library via `library.json` configuration.
- The `test/` directory (not `tests/`) follows PlatformIO's native test convention.
- The `examples/` directory contains runnable example sketches for library consumers.
- No `envs/` directory — PlatformIO manages its own toolchain environment.
- No `pyproject.toml` — this is a pure C++ project.

---

## C++ PlatformIO firmware

Based on `sl-micro-controllers`.

```text
project-root/
├── docs/
│   ├── source/
│   │   ├── doxygen/                  # Doxygen-generated XML for Breathe
│   │   │   └── xml/                  # XML output consumed by Sphinx
│   │   ├── api.rst                   # API reference (doxygenfile directives)
│   │   ├── conf.py                   # Sphinx config with Breathe integration
│   │   ├── index.rst                 # Main page with toctree
│   │   └── welcome.rst              # Landing page content
│   ├── make.bat                      # Windows Sphinx wrapper
│   └── Makefile                      # Unix Sphinx wrapper
├── src/                              # Firmware source
│   ├── main.cpp                      # Firmware entry point (setup/loop)
│   ├── custom_module.h               # Hardware module headers
│   └── another_module.h              # Each module is a single .h file
├── .clang-format                     # C++ formatting configuration
├── .clang-tidy                       # C++ linting configuration
├── .gitignore
├── Doxyfile                          # Doxygen documentation configuration
├── LICENSE                           # GPL-3.0 license
├── platformio.ini                    # PlatformIO build configuration
├── README.md                         # Project documentation
└── tox.ini                           # Documentation build automation
```

### Notes

- Firmware projects have no `examples/`, `test/`, or `library.json` — the firmware itself is the
  final artifact.
- All custom modules are header-only `.h` files in `src/` alongside `main.cpp`.
- The `main.cpp` is the actual firmware entry point (not a development stub like in library
  projects).
- Uses `#define` / `#ifdef` conditional compilation for hardware variant selection.
- No `envs/` directory — PlatformIO manages its own toolchain environment.
- No `pyproject.toml` — this is a pure C++ project.

---

## C# Unity

Based on `sl-unity-tasks`.

```text
project-root/
├── Assets/                           # Unity project assets (root of all content)
│   ├── TaskName/                     # Task-specific asset folder
│   │   ├── Configurations/           # JSON configuration files
│   │   ├── Materials/                # Unity materials
│   │   ├── Models/                   # 3D models
│   │   ├── Prefabs/                  # Unity prefabs
│   │   ├── Scripts/                  # C# source code
│   │   │   ├── TaskScript.cs         # Task logic scripts
│   │   │   └── UtilityScript.cs      # Helper scripts
│   │   ├── Sounds/                   # Audio assets
│   │   └── Textures/                 # Texture assets
│   ├── Plugins/                      # Third-party Unity plugins
│   ├── Scenes/                       # Unity scene files
│   ├── Textures/                     # Shared textures
│   └── UI-*/                         # UI-related asset folders
├── Packages/                         # Unity Package Manager configuration
│   └── manifest.json                 # Package dependencies
├── ProjectSettings/                  # Unity project configuration
│   ├── ProjectSettings.asset         # Core project settings
│   ├── QualitySettings.asset         # Graphics quality tiers
│   ├── InputManager.asset            # Input configuration
│   └── *.asset                       # Other Unity settings files
├── .csharpierignore                  # CSharpier formatter ignore patterns
├── .csharpierrc.yaml                 # CSharpier formatter configuration
├── .editorconfig                     # Editor configuration (indentation, encoding)
├── .gitignore
├── CLAUDE.md                         # Claude Code project instructions
├── LICENSE                           # GPL-3.0 license
├── README.md                         # Project documentation
└── *.slnx                            # Unity solution file
```

### Notes

- Unity projects use `Assets/` as the root for all content, not `src/`.
- Each task or feature gets its own folder under `Assets/` containing all related assets
  (scripts, prefabs, materials, etc.).
- C# source files live in `Assets/TaskName/Scripts/` directories. Every `.cs` file has a
  corresponding `.cs.meta` file managed by Unity (committed to version control).
- The `ProjectSettings/` directory contains Unity engine configuration files. These are
  `.asset` files managed by the Unity Editor.
- No `pyproject.toml`, `tox.ini`, `envs/`, `docs/`, or `tests/` — Unity has its own build and
  test infrastructure.
- Formatting is managed by CSharpier (`.csharpierrc.yaml`) and EditorConfig (`.editorconfig`),
  not by Python-based tools.
- The `Library/`, `Logs/`, `Temp/`, and `UserSettings/` directories are gitignored.
