# Archetype directory trees

Annotated directory trees for each project archetype. A tree whose section heading names a canonical example repository
is verified against that repository. The firmware and Unity headings name none, because no repository exemplifies either
archetype in its modernized form yet, so those two trees state the target layout rather than a verified one.

---

## Python-only

Based on `ataraxis-automation` and `ataraxis-base-utilities`.

```text
project-root/
├── .codegraph/                       # (optional) Generated code index, gitignored except its own .gitignore
├── .github/
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.yml            # Structured bug report form
│       ├── config.yml                # Template chooser configuration
│       └── feature_request.yml       # Structured feature request form
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
│   ├── {abbr}_dev_osx.yml            # (optional) macOS specification, for projects supporting macOS
│   └── {abbr}_dev_win.yml            # (optional) Windows specification, for projects supporting Windows
├── examples/                         # (optional) Runnable usage examples for library consumers
│   └── example_usage.py              # Standalone script demonstrating the public API
├── src/
│   └── package_name/                 # Main package (underscore-separated)
│       ├── submodule/                # (optional) Subpackage directories
│       │   ├── __init__.py
│       │   └── module.py
│       ├── __init__.py               # Package init with public API re-exports
│       ├── __init__.pyi              # Generated stub for re-exports (present in releases only)
│       ├── module.py                 # Source modules
│       └── py.typed                  # PEP 561 marker for typed packages
├── tests/                            # Present for projects defining a test environment
│   ├── submodule/                    # (optional) Mirrors src/ subpackage structure
│   │   └── module_test.py
│   ├── fixtures/                     # (optional) Non-Python test data and vendored binary test doubles
│   ├── conftest.py                   # (optional) Fixtures pytest discovers for the whole test directory
│   ├── module_test.py                # Test files use _test.py suffix
│   └── shared_builders.py            # (optional) Test-support module, named for what it provides
├── .gitattributes                    # (optional) Line-ending normalization for tracked text files
├── .gitignore
├── .netlify-site                  # (optional) Netlify site identifier, present with the deploy environment
├── CLAUDE.md                         # Claude Code project instructions
├── LICENSE                           # Apache-2.0 license
├── pyproject.toml                    # Build config, metadata, tool settings
├── README.md                         # Project documentation
└── tox.ini                           # Automation orchestration (lint, type, test, docs)
```

### Notes

- The `{abbr}` placeholder in `envs/` files is a short project abbreviation (e.g., `axa` for ataraxis-automation, `axbu`
  for ataraxis-base-utilities).
- The `tests/` directory mirrors the `src/package_name/` structure. Test files use the `_test.py` suffix (e.g.,
  `automation_test.py`).
- Test-support modules under `tests/` hold the fixtures, builders, and fakes shared across test modules. Each one is
  named for what it provides rather than with the `_test.py` suffix, which the rule above reserves for test modules,
  and `conftest.py` is the support module pytest discovers fixtures from automatically.
- The `examples/` directory is optional and holds runnable scripts that demonstrate the public API for library
  consumers. It is source-distribution content. A project that also ships it in the wheel maps it to a
  distribution-qualified package name. A bare top-level `examples` package installs into a shared
  `site-packages/examples/` namespace that every distribution shipping that directory name would share.
- `.pyi` stub files and the `py.typed` marker are generated artifacts whose presence is release-phase-dependent, so a
  missing `.pyi` is not a layout violation. See `/python-style` for the stub-file rule.
- The `.pypirc` file may exist locally but is not committed to version control.
- Build artifacts (`dist/`, `reports/`, `coverage.xml`) are gitignored.

---

## Python + C++ extension

Based on `ataraxis-time`.

```text
project-root/
├── .codegraph/                       # (optional) Generated code index, gitignored except its own .gitignore
├── .github/
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.yml            # Structured bug report form
│       ├── config.yml                # Template chooser configuration
│       └── feature_request.yml       # Structured feature request form
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
│   ├── {abbr}_dev_osx.yml            # (optional) macOS specification, for projects supporting macOS
│   └── {abbr}_dev_win.yml            # (optional) Windows specification, for projects supporting Windows
├── examples/                         # (optional) Runnable usage examples for library consumers
│   └── example_usage.py              # Standalone script demonstrating the public API
├── src/
│   ├── c_extensions/                 # C++ extension sources
│   │   └── module_ext.cpp            # nanobind extension (snake_case_ext.cpp)
│   ├── python_wrapper/               # Pure Python wrapper around C++ extension
│   │   ├── __init__.py
│   │   ├── wrapper_module.py         # Python class wrapping C++ class
│   │   └── wrapper_module.pyi        # Generated stub for wrapper (present in releases only)
│   ├── pure_python_module/           # (optional) Additional pure Python subpackages
│   │   ├── __init__.py
│   │   └── module.py
│   ├── __init__.py                   # Top-level package init
│   ├── __init__.pyi                  # Generated top-level stub (present in releases only)
│   ├── module_ext.pyi                # Generated stub for compiled C++ extension (releases only)
│   └── py.typed                      # PEP 561 marker
├── tests/
│   ├── python_wrapper/               # Tests for Python wrapper
│   │   └── wrapper_test.py
│   ├── pure_python_module/           # Tests for pure Python modules
│   │   └── module_test.py
│   ├── conftest.py                   # (optional) Fixtures pytest discovers for the whole test directory
│   └── shared_builders.py            # (optional) Test-support module, named for what it provides
├── tools/                            # Build-support scripts called by CMake and the wheel repair step
│   ├── check_build_env.py            # Verifies the toolchain before a build starts
│   └── repair_windows_wheel.py       # Repairs Windows wheels cibuildwheel cannot repair natively
├── .clang-format                     # C++ formatting configuration
├── .clang-tidy                       # C++ linting configuration
├── .gitattributes                    # (optional) Line-ending normalization for tracked text files
├── .gitignore
├── .netlify-site                  # (optional) Netlify site identifier, present with the deploy environment
├── CLAUDE.md                         # Claude Code project instructions
├── CMakeLists.txt                    # CMake build config for nanobind extension
├── Doxyfile                          # Doxygen documentation configuration
├── LICENSE                           # Apache-2.0 license
├── pyproject.toml                    # scikit-build-core backend + metadata
├── README.md                         # Project documentation
└── tox.ini                           # Automation orchestration
```

### Notes

- The `src/` layout uses a flat namespace: `c_extensions/`, wrapper subpackages, and pure Python subpackages are all
  direct children of `src/`.
- C++ extension stubs (`module_ext.pyi`) live at the `src/` top level alongside `py.typed`.
- The `tests/` directory carries test-support modules alongside its test modules, under the rule the Python-only notes
  above state.
- The `examples/` directory is optional and follows the rule the Python-only notes above state.
- The `CMakeLists.txt` at the project root drives the nanobind build via scikit-build-core.
- Build artifacts (`build/`) contain per-Python-version subdirectories and are gitignored.

---

## C++ PlatformIO library

Based on `ataraxis-transport-layer-mc`.

```text
project-root/
├── .codegraph/                       # (optional) Generated code index, gitignored except its own .gitignore
├── .github/
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.yml            # Structured bug report form
│       ├── config.yml                # Template chooser configuration
│       └── feature_request.yml       # Structured feature request form
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
│   └── {abbr}_shared_assets.h       # Shared types and constants, abbreviation-prefixed
├── test/                             # PlatformIO native tests
│   └── test_component.cpp            # Unity framework test files
├── .clang-format                     # C++ formatting configuration
├── .clang-tidy                       # C++ linting configuration
├── .gitattributes                    # (optional) Line-ending normalization for tracked text files
├── .gitignore
├── .netlify-site                  # (optional) Netlify site identifier, present with the deploy environment
├── CLAUDE.md                         # Claude Code project instructions
├── Doxyfile                          # Doxygen documentation configuration
├── library.json                      # PlatformIO library manifest
├── LICENSE                           # Apache-2.0 license
├── platformio.ini                    # PlatformIO build configuration
├── README.md                         # Project documentation
└── tox.ini                           # Documentation build automation
```

### Notes

- All library code is **header-only** (`.h` files only, no `.cpp` implementation files).
- The `main.cpp` in `src/` is a development entry point used for testing during development. It is excluded from the
  distributed library by the `export.exclude` rule that `/platformio-config` owns.
- The `test/` directory (not `tests/`) follows PlatformIO's native test convention.
- The `examples/` directory contains runnable example sketches for library consumers.
- No `envs/` directory, because PlatformIO manages its own toolchain environment.
- No `pyproject.toml`, because this is a pure C++ project.

---

## C++ PlatformIO firmware

Based on a microcontroller firmware project.

```text
project-root/
├── .codegraph/                       # (optional) Generated code index, gitignored except its own .gitignore
├── .github/
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.yml            # Structured bug report form
│       ├── config.yml                # Template chooser configuration
│       └── feature_request.yml       # Structured feature request form
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
├── .gitattributes                    # (optional) Line-ending normalization for tracked text files
├── .gitignore
├── .netlify-site                  # (optional) Netlify site identifier, present with the deploy environment
├── CLAUDE.md                         # Claude Code project instructions
├── Doxyfile                          # Doxygen documentation configuration
├── LICENSE                           # Apache-2.0 license
├── platformio.ini                    # PlatformIO build configuration
├── README.md                         # Project documentation
└── tox.ini                           # Documentation build automation
```

### Notes

- Firmware projects have no `examples/`, `test/`, or `library.json`, because the firmware itself is the final artifact.
- All custom modules are header-only `.h` files in `src/` alongside `main.cpp`.
- The `main.cpp` is the actual firmware entry point (not a development stub like in library projects).
- Uses `#define` / `#ifdef` conditional compilation for hardware variant selection.
- No `envs/` directory and no `pyproject.toml`, for the reasons the C++ PlatformIO library notes above state.

---

## C# Unity

Based on a Unity behavioral-task project.

```text
project-root/
├── .codegraph/                       # (optional) Generated code index, gitignored except its own .gitignore
├── .github/
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.yml            # Structured bug report form
│       ├── config.yml                # Template chooser configuration
│       └── feature_request.yml       # Structured feature request form
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
│   ├── VRSettings/                   # Actor and display scriptable-object settings
│   └── UI-*/                         # UI-related asset folders
├── Packages/                         # Unity Package Manager configuration
│   └── manifest.json                 # Package dependencies
├── imgs/                             # (optional) Screenshots referenced by the README
├── ProjectSettings/                  # Unity project configuration
│   ├── ProjectSettings.asset         # Core project settings
│   ├── QualitySettings.asset         # Graphics quality tiers
│   ├── InputManager.asset            # Input configuration
│   └── *.asset                       # Other Unity settings files
├── .csharpierignore                  # CSharpier formatter ignore patterns
├── .csharpierrc.yaml                 # CSharpier formatter configuration
├── .editorconfig                     # Editor configuration (indentation, encoding)
├── .gitattributes                    # (optional) Line-ending normalization for tracked text files
├── .gitignore
├── CLAUDE.md                         # Claude Code project instructions
├── LICENSE                           # Apache-2.0 license
├── README.md                         # Project documentation
└── *.slnx                            # Unity solution file
```

### Notes

- Unity projects use `Assets/` as the root for all content, not `src/`.
- Each task or feature gets its own folder under `Assets/` containing all related assets (scripts, prefabs, materials,
  etc.).
- C# source files live in `Assets/TaskName/Scripts/` directories. Every `.cs` file has a corresponding `.cs.meta` file
  managed by Unity (committed to version control).
- The `ProjectSettings/` directory contains Unity engine configuration files. These are `.asset` files managed by the
  Unity Editor.
- Unity has its own build and test infrastructure, so a Unity project carries no `pyproject.toml`, `tox.ini`, `envs/`,
  `docs/`, or `tests/`.
- Formatting is managed by CSharpier (`.csharpierrc.yaml`) and EditorConfig (`.editorconfig`), not by Python-based
  tools.
- The `Library/`, `Logs/`, `Temp/`, and `UserSettings/` directories are gitignored.
