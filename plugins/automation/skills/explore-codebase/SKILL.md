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

| Tier   | Indicators                                                    | Approach                       |
|--------|---------------------------------------------------------------|--------------------------------|
| Small  | Single package, < 10 source files, no subpackages             | Single-pass exploration        |
| Medium | Multiple packages or 10-50 source files                       | Structured four-phase          |
| Large  | Monorepo, 50+ source files, multiple entry points, MCP server | Parallel subagent exploration  |

### Step 2: Execute exploration

Follow the approach for the determined tier.

**Small projects:** Execute all four exploration phases yourself in a single pass. Combine phases
where appropriate to keep exploration concise.

**Medium projects:** Execute all four exploration phases sequentially, giving each phase focused
attention. Use the Task tool with `subagent_type: Explore` for any phase that requires reading
many files.

**Large projects:** Launch 2-3 Explore subagents in parallel using the Task tool with
`subagent_type: Explore`. Assign each subagent a different focus area:

- **Subagent 1: Structure and entry points** — Phase 1 (feature discovery) and Phase 4
  (configuration and test coverage)
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

1. **Entry points** — CLI commands, API endpoints, main functions, GUI entry points. Check
   `pyproject.toml` for `[project.scripts]` and `[project.entry-points]` sections.
2. **Configuration mechanisms** — `pyproject.toml` settings, environment variables, CLI argument
   defaults, YAML/JSON/TOML config files, `.env` files, and dataclass-based configuration objects.
3. **Public API surface** — Classes, functions, and constants exported from `__init__.py` files.
   Note which modules use `__all__` to restrict exports.
4. **CLI command signatures** — For CLI-based projects, document each command's name, arguments,
   options, and purpose.

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
   any source modules that lack test coverage.
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
7. **Configuration** — All configuration mechanisms discovered
8. **Test coverage** — Source module → test file mapping, noting gaps
9. **Notable patterns** — Design patterns, conventions, and cross-cutting concerns
10. **Areas of concern** — Technical debt, complexity hotspots, missing coverage

### Example output

```markdown
## Project purpose

Provides a reimplemented suite2p library for neural imaging analysis with multi-day cell tracking
capabilities for the Sun Lab at Cornell University.

## Entry points and CLI commands

| Entry Point     | Location                  | Description                              |
|-----------------|---------------------------|------------------------------------------|
| `sl-suite2p`    | pyproject.toml scripts    | Main CLI entry point                     |
| `run`           | cli.py:run                | Executes single-day processing pipeline  |
| `run-multiday`  | cli.py:run_multiday       | Executes multi-day tracking pipeline     |
| `mcp`           | cli.py:mcp                | Starts the MCP server                    |

## Key components

| Component          | Location                       | Purpose                                           |
|--------------------|--------------------------------|---------------------------------------------------|
| Pipeline           | src/sl_suite2p/pipeline.py     | Main single-day processing pipeline orchestration |
| Multi-day Tracking | src/sl_suite2p/multiday/       | Cross-session cell tracking and alignment         |
| Registration       | src/sl_suite2p/registration/   | Image registration and motion correction          |
| Detection          | src/sl_suite2p/detection/      | Cell detection algorithms                         |
| Configuration      | src/sl_suite2p/configuration/  | Pipeline configuration management                 |
| MCP Server         | src/sl_suite2p/mcp/            | AI agent integration via MCP                      |

## Call chain summary

`sl-suite2p run` → `cli.py:run` → `pipeline.py:run_pipeline` → `registration/register.py:register`
→ `detection/detect.py:detect_cells` → `extraction/extract.py:extract_signals`
→ `classification/classify.py:classify_cells` → writes output to `suite2p/` directory.

## Import dependency map

Central components (imported by 5+ modules):
- `configuration/pipeline_config.py` — imported by pipeline, registration, detection, extraction
- `utils/io.py` — imported by all processing modules
- `types.py` — imported by all modules for shared type definitions

Dependency direction: cli → pipeline → processing modules → utils/io + configuration.

## Public API surface

Exported from `__init__.py` via `__all__`:
- `run_pipeline(config: PipelineConfig) -> PipelineResult`
- `PipelineConfig` (dataclass)
- `PipelineResult` (dataclass)

## Configuration

| Mechanism       | Location                        | Purpose                              |
|-----------------|---------------------------------|--------------------------------------|
| `PipelineConfig`| configuration/pipeline_config.py| Dataclass with all pipeline settings |
| CLI arguments   | cli.py                          | Override config values at runtime    |
| YAML config     | User-provided path              | Full pipeline configuration file     |

## Test coverage

| Source Module                | Test File                          | Coverage |
|------------------------------|------------------------------------|----------|
| pipeline.py                  | tests/test_pipeline.py             | Yes      |
| registration/register.py     | tests/test_registration.py         | Yes      |
| detection/detect.py          | tests/test_detection.py            | Yes      |
| mcp/server.py                | (none)                             | Gap      |

## Notable patterns

- Numba-accelerated computation for image processing
- Configuration dataclasses with validation
- MyPy strict mode with full type annotations
- Processing modules follow a consistent register → detect → extract → classify pipeline

## Areas of concern

- `mcp/server.py` lacks test coverage
- Large refactoring effort from original suite2p codebase still in progress
- Multi-day tracking module has high cyclomatic complexity
- Several `# type: ignore` suppressions in registration module
```

---

## Related skills

| Skill                   | Relationship                                                        |
|-------------------------|---------------------------------------------------------------------|
| `/explore-dependencies` | Explores ataraxis dependency APIs; invoke alongside this skill      |
| `/python-style`         | Provides Python coding conventions discovered during exploration    |
| `/cpp-style`            | Provides C++ coding conventions discovered during exploration       |
| `/csharp-style`         | Provides C# coding conventions discovered during exploration        |
| `/readme-style`         | Provides README conventions when exploration reveals README issues  |
| `/commit`               | Should be invoked after completing code changes informed by context |
| `/skill-design`         | Provides skill conventions when exploration reveals skill files     |

---

## Proactive behavior

Invoke at session start to ensure full context before making changes. Prevents blind modifications
and ensures understanding of existing patterns. When the project has ataraxis or Sun Lab
dependencies (check `pyproject.toml`), also invoke `/explore-dependencies` to build a live API
snapshot of each dependency.

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
