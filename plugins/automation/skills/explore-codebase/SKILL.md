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

### Step 1: Determine project archetype and size

First detect the project archetype (Python-only, Python + C++ extension, C++ PlatformIO library or
firmware, or C# Unity) using the indicator table in `/project-layout`; do not re-derive those
indicators here. The archetype determines which entry points, manifests, and source extensions the
phases target (see "Archetype-specific signals" after the phases).

Then select the exploration tier. Count first-party source modules with a scoped command rather than
listing the whole tree — e.g. `find src -name "*.py" | wc -l` (Python) or
`find . -name "*.cpp" -o -name "*.h" | wc -l` (C++/PlatformIO) — paired with a top-level directory
listing.

Ataraxis source uses GENERATED `.pyi` stubs (one per `.py`). They are purged during development
(`tox -e lint`) and only present at release (`tox -e stubs`), so a dev tree often has none — treat
their absence as normal, never a gap. When present, exclude `.pyi` from file counts, dependency maps,
and test-coverage mapping, and always read the `.py` module (not the stub) for docstrings and the
documented API. Do not spend tokens reading or editing stubs.

| Tier   | Indicators                                               | Approach                      |
|--------|----------------------------------------------------------|-------------------------------|
| Small  | Single package, < 10 source files, no subpackages        | Single-pass exploration       |
| Medium | Multiple packages, or 10-50 source files                 | Structured four-phase         |
| Large  | 50+ source files AND (monorepo OR multiple entry points) | Parallel subagent exploration |

An MCP server alone does NOT force the Large tier — MCP servers are common on single-package ataraxis
libraries. When indicators conflict, choose the lower tier (fewer subagents) unless the source-file
count alone clearly exceeds 50.

### Step 2: Execute exploration

Follow the approach for the determined tier.

**Small projects:** Execute all four exploration phases yourself in a single pass. Combine phases
where appropriate to keep exploration concise.

**Medium projects:** Execute all four exploration phases sequentially, giving each phase focused
attention. Use the Agent tool with the `Explore` agent type for any phase that requires reading
many files.

**Large projects:** Launch 2-3 Explore subagents in parallel using the Agent tool with the
`Explore` agent type. Assign each subagent a disjoint focus area so no surface is explored twice:

- **Subagent 1: Structure, entry points, and configuration** — Phase 1 (feature discovery and
  configuration), excluding the public API and MCP surfaces assigned to Subagent 3
- **Subagent 2: Architecture and code flow** — Phase 2 (code flow tracing) and Phase 3 (architecture
  analysis including import mapping and central component identification)
- **Subagent 3: API surface, MCP tools, and quality** — public API enumeration, MCP-tools
  enumeration, and Phase 4 (test coverage, error handling patterns, technical debt indicators)

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

## Archetype-specific signals

The four phases are language-agnostic; only the concrete signals change by archetype (see
`/project-layout` for archetype indicators). Substitute the following, then apply the matching style
skill (`/python-style`, `/cpp-style`, or `/csharp-style`):

| Archetype          | Entry points / manifest                          | Public API source              |
|--------------------|--------------------------------------------------|--------------------------------|
| Python (+ C++ ext) | `pyproject.toml` scripts, `__init__.py`          | `__all__` exports              |
| C++ PlatformIO     | `library.json`, `platformio.ini`, `src/main.cpp` | public classes in header files |
| C# Unity           | `Assets/`, `ProjectSettings/`, `*.slnx`          | `MonoBehaviour` entry points   |

For C++ projects, enumerate the public surface from header files rather than `__all__`, read the
version from `library.json` (not `pyproject.toml`), and map tests to the PlatformIO `test/` directory.

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

For a complete worked example of this output format, see
[example-output.md](references/example-output.md).

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
(check `pyproject.toml`), also invoke `/explore-dependencies` to build a live API snapshot of each
dependency, including reconciliation of any local or editable checkouts against the latest GitHub
release.

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
- [ ] MCP tools enumerated with parameters and return/output shape when an MCP server exists
- [ ] Configuration mechanisms documented (pyproject.toml, env vars, config files, dataclasses)
- [ ] Test files mapped to source modules with coverage gaps noted
- [ ] Design patterns and cross-cutting concerns documented
- [ ] Areas of concern noted (technical debt, complexity, missing coverage)
- [ ] Output uses structured format (headings, tables, lists)
- [ ] No code modifications were made during exploration
- [ ] Exploration depth matches project size tier (small/medium/large)
```
