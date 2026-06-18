---
name: explore-codebase
description: >-
  Performs in-depth codebase exploration at the start of a coding session. Builds comprehensive
  understanding of project structure, architecture, key components, and patterns. Use when starting
  a new session, when asked to understand or explore the codebase, when asked "what does this project
  do", when exploring unfamiliar code, or when the user asks about project structure or architecture.
user-invocable: true
---

# Codebase exploration

Performs thorough, structured codebase exploration to build deep understanding before coding work begins.

---

## Scope

**Covers:**
- Exploring project structure, architecture, and directory layout
- Tracing execution paths from entry points through core logic
- Mapping import dependencies and identifying central components
- Enumerating public API surfaces and CLI commands
- Discovering configuration mechanisms (pyproject.toml, env vars, config files)
- Mapping test coverage to source modules
- Producing a structured summary of findings

**Does not cover:**
- Modifying code or configuration files
- Applying coding conventions (see `/python-style`)
- Writing commit messages (see `/commit`)
- Creating or modifying skill files (see `/skill-design`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Determine project size

Quickly assess the project to select the appropriate exploration tier. Run a file count and directory
listing to make this determination before proceeding.

Ataraxis source uses GENERATED `.pyi` stubs (one per `.py`). They are purged during development
(`tox -e lint`) and only present at release (`tox -e stubs`), so a dev tree often has none — treat
their absence as normal, never a gap. When present, exclude `.pyi` from file counts, dependency maps,
and test-coverage mapping, and always read the `.py` module (not the stub) for docstrings and the
documented API. Do not spend tokens reading or editing stubs.

| Tier   | Indicators                                                    | Approach                      |
|--------|---------------------------------------------------------------|-------------------------------|
| Small  | Single package, < 10 source files, no subpackages             | Single-pass exploration       |
| Medium | Multiple packages or 10-50 source files                       | Structured four-phase         |
| Large  | Monorepo, 50+ source files, multiple entry points, MCP server | Parallel subagent exploration |

### Step 2: Execute exploration

Follow the approach for the determined tier.

**Small projects:** Execute all four exploration phases yourself in a single pass. Combine phases
where appropriate to keep exploration concise.

**Medium projects:** Execute all four exploration phases sequentially, giving each phase focused
attention. Use the Task tool with `subagent_type: Explore` for any phase that requires reading
many files.

**Large projects:** Launch 2-3 Explore subagents in parallel using the Task tool with
`subagent_type: Explore`. Assign each subagent a different focus area:

- **Subagent 1: Structure and entry points** — Phase 1 (feature discovery, including configuration)
  and Phase 4 (test coverage and implementation details)
- **Subagent 2: Architecture and dependencies** — Phase 2 (code flow tracing) and Phase 3
  (architecture analysis including import mapping and central component identification)
- **Subagent 3: API surface and quality** — Public API enumeration, error handling patterns,
  technical debt indicators

Synthesize the subagent findings into a unified summary.

### Step 3: Present findings

Present the structured summary following the output format below. Do NOT make code changes during
exploration. Wait for user direction before proceeding.

---

## Exploration phases

Every exploration follows four phases regardless of project size. For small projects, phases may be
combined. For large projects, phases may be distributed across parallel subagents.

### Phase 1: Feature discovery

Identify the project's entry points, boundaries, and configuration.

1. **Orientation documents** — Read the repo-root `CLAUDE.md` (and `README.md`) for project purpose,
   architecture summary, and conventions before tracing source. Treat them as orientation, not ground
   truth for API details.
2. **Entry points** — CLI commands, API endpoints, main functions, GUI entry points. Check
   `pyproject.toml` for `[project.scripts]` and `[project.entry-points]` sections.
3. **Configuration mechanisms** — `pyproject.toml` settings, environment variables, CLI argument
   defaults, YAML/JSON/TOML config files, `.env` files, and dataclass-based configuration objects.
4. **Public API surface** — Classes, functions, and constants exported from `__init__.py` files.
   Note which modules use `__all__` to restrict exports.
5. **CLI command signatures** — For CLI-based projects, document each command's name, arguments,
   options, and purpose.
6. **MCP tools** — When an MCP server exists (an `interfaces/` package, `*_tools.py` modules, or an
   `<cli> mcp` script), enumerate each MCP tool with its name, parameters, and return/output shape.
   These tools are a major public surface for AI agents.

### Phase 2: Code flow tracing

Trace execution paths from entry points through the codebase layers.

1. **Call chain tracing** — Starting from each entry point identified in Phase 1, follow the call
   chain through to core logic. Document the path as a sequence of `file:function` references.
2. **Data transformations** — At each step in the call chain, note what data is passed, transformed,
   or returned.
3. **Layer identification** — Identify the abstraction layers the call chain passes through
   (e.g., CLI → validation → business logic → data access → output).
4. **Side effects** — Note where the code interacts with external systems (file I/O, network calls,
   subprocess invocations, environment variable reads).

### Phase 3: Architecture analysis

Analyze the structural relationships between components.

1. **Import/dependency mapping** — For each source module, identify what it imports from other
   project modules. Note the direction of dependencies (which modules depend on which).
2. **Central component identification** — Identify which files or classes are imported by the most
   other modules. These are the architecturally central components that changes will most likely
   ripple through.
3. **Design patterns** — Identify patterns in use (dataclasses, protocols, factories, decorators,
   context managers, abstract base classes, etc.).
4. **Cross-cutting concerns** — Note how logging, error handling, configuration, and validation are
   handled across the codebase.

### Phase 4: Implementation details

Examine specifics that inform future modifications.

1. **Test coverage mapping** — For each source module, identify the corresponding test file(s). Note
   any source modules that lack test coverage. Ataraxis test files use the `<module>_test.py` suffix
   (e.g. `camera_test.py` for `camera.py`), not the `test_` prefix — see `/project-layout`.
2. **Key algorithms and data structures** — Document non-trivial algorithms, important data
   structures (dataclasses, TypedDicts, NamedTuples), and their locations.
3. **Error handling patterns** — How errors are raised, caught, and reported to the user. Note
   whether the project uses custom exception classes, logging, or a console utility.
4. **Technical debt and complexity** — Areas with high complexity, TODO/FIXME comments, known
   limitations, type: ignore suppressions, or noqa markers.

---

## Output format

Present findings using the following structure. Include all sections for medium and large projects.
For small projects, omit sections that do not apply.

### Required sections

1. **Project purpose** — 1-2 sentence summary
2. **Entry points and CLI commands** — Table of entry points with locations and descriptions
3. **Key components** — Table of components with locations and purposes
4. **Call chain summary** — Entry point → layer → core logic flow for primary paths
5. **Import dependency map** — Which modules depend on which, with central components highlighted
6. **Public API surface** — Exported classes, functions, and constants
7. **MCP tools** — When an MCP server exists, each tool with its parameters and return/output shape
8. **Configuration** — All configuration mechanisms discovered
9. **Test coverage** — Source module → test file mapping, noting gaps
10. **Notable patterns** — Design patterns, conventions, and cross-cutting concerns
11. **Areas of concern** — Technical debt, complexity hotspots, missing coverage

### Example output

```markdown
## Project purpose

Provides a camera interface library for OpenCV and GeniCam cameras with real-time FFMPEG video
encoding, including an MCP server and CLI tools for camera management.

## Entry points and CLI commands

| Entry Point    | Location               | Description                         |
|----------------|------------------------|-------------------------------------|
| `axvs`         | pyproject.toml scripts | Main CLI entry point                |
| `list-cameras` | cli.py:list_cameras    | Enumerates connected cameras        |
| `live`         | cli.py:live            | Opens a live camera preview session |
| `mcp`          | cli.py:mcp             | Starts the MCP server               |

## Key components

| Component       | Location                                  | Purpose                                |
|-----------------|-------------------------------------------|----------------------------------------|
| VideoSystem     | src/ataraxis_video_system/video_system.py | Top-level camera acquisition lifecycle |
| Camera backends | src/ataraxis_video_system/camera/         | OpenCV and GeniCam camera interfaces   |
| Savers          | src/ataraxis_video_system/saver/          | FFMPEG-based video and image encoders  |
| Configuration   | src/ataraxis_video_system/configuration/  | Acquisition and encoding settings      |
| MCP Server      | src/ataraxis_video_system/mcp/            | AI agent integration via MCP           |

## Call chain summary

`axvs live` → `cli.py:live` → `video_system.py:VideoSystem.start`
→ `camera/opencv_camera.py:OpenCVCamera.connect` → `saver/video_saver.py:VideoSaver.create_encoder`
→ writes encoded frames to the output directory.

## Import dependency map

Central components (imported by 5+ modules):
- `configuration/acquisition_config.py` — imported by video_system, camera, saver
- `camera/camera_base.py` — base class imported by all camera backends
- `types.py` — imported by all modules for shared type definitions

Dependency direction: cli → video_system → camera + saver → configuration.

## Public API surface

Exported from `__init__.py` via `__all__`:
- `VideoSystem` (class)
- `CameraBackends` (enum)
- `SaverBackends` (enum)

## MCP tools

| Tool                | Location               | Parameters       | Returns              |
|---------------------|------------------------|------------------|----------------------|
| `list_cameras_tool` | mcp/discovery_tools.py | (none)           | list of camera dicts |
| `check_camera_tool` | mcp/discovery_tools.py | `camera_id: int` | camera status dict   |

## Configuration

| Mechanism           | Location                            | Purpose                                  |
|---------------------|-------------------------------------|------------------------------------------|
| `AcquisitionConfig` | configuration/acquisition_config.py | Dataclass with camera + encoder settings |
| CLI arguments       | cli.py                              | Override config values at runtime        |
| YAML config         | User-provided path                  | Full acquisition configuration file      |

## Test coverage

| Source Module           | Test File                   | Coverage |
|-------------------------|-----------------------------|----------|
| video_system.py         | tests/video_system_test.py  | Yes      |
| camera/opencv_camera.py | tests/opencv_camera_test.py | Yes      |
| saver/video_saver.py    | tests/video_saver_test.py   | Yes      |
| mcp/server.py           | (none)                      | Gap      |

## Notable patterns

- Multiprocessing for concurrent acquisition and encoding
- Configuration dataclasses with validation
- MyPy strict mode with full type annotations
- Camera backends follow a consistent connect → grab → release interface

## Areas of concern

- `mcp/server.py` lacks test coverage
- GeniCam backend requires a vendor SDK that is unavailable in CI
- Saver module has high cyclomatic complexity
- Several `# type: ignore` suppressions in the camera backend
```

---

## Related skills

| Skill                   | Relationship                                                                         |
|-------------------------|--------------------------------------------------------------------------------------|
| `/explore-dependencies` | Explores ataraxis dependency APIs; invoke alongside this skill                       |
| `/python-style`         | Provides Python coding conventions discovered during exploration                     |
| `/cpp-style`            | Provides C++ coding conventions discovered during exploration                        |
| `/csharp-style`         | Provides C# coding conventions discovered during exploration                         |
| `/readme-style`         | Provides README conventions when exploration reveals README issues                   |
| `/commit`               | Should be invoked after completing code changes informed by context                  |
| `/skill-design`         | Provides skill conventions when exploration reveals skill files                      |
| `/project-layout`       | Provides project directory and test-naming conventions referenced during exploration |

---

## Proactive behavior

Invoke at session start to ensure full context before making changes. Prevents blind modifications
and ensures understanding of existing patterns. When the project has ataraxis dependencies
(check `pyproject.toml`), also invoke `/explore-dependencies` to build a live API
snapshot of each dependency. Ataraxis dependencies may exist locally in the parent directory; when
they do, `/explore-dependencies` should reconcile the local version against the latest GitHub release
(`gh api repos/.../releases/latest`) before trusting its API.

Do NOT make code changes during exploration. Present findings and wait for user direction.

---

## Verification checklist

**You MUST verify the exploration output against this checklist before presenting it to the user.**

```text
Exploration Output Compliance:
- [ ] Project purpose summarized (1-2 sentences)
- [ ] Entry points identified with locations (pyproject.toml scripts, CLI commands)
- [ ] Key components identified with locations and purposes
- [ ] Call chains traced from entry points through core logic (file:function references)
- [ ] Import dependencies mapped with central components highlighted
- [ ] Public API surface enumerated (exported classes, functions, constants)
- [ ] Configuration mechanisms documented (pyproject.toml, env vars, config files, dataclasses)
- [ ] Test files mapped to source modules with coverage gaps noted
- [ ] Design patterns and cross-cutting concerns documented
- [ ] Areas of concern noted (technical debt, complexity, missing coverage)
- [ ] Output uses structured format (headings, tables, lists)
- [ ] No code modifications were made during exploration
- [ ] Exploration depth matches project size tier (small/medium/large)
```
