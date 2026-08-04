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
- Driving exploration through the CodeGraph index when the repository provides one
- Producing a structured summary of findings

**Does not cover:**
- Modifying code or configuration files
- Applying coding conventions (see `/python-style`)
- Writing commit messages (see `/commit`)
- Creating or modifying skill files (see `/skill-design`)

---

## Workflow

You MUST follow these steps when this skill is invoked.

### Step 1: Detect the CodeGraph index

Check whether the repository root contains a `.codegraph/` directory. Many ataraxis and sollertia
library projects carry one, and the index answers most exploration questions in a single call, so
this check comes first because it changes how every later step is executed.

When `.codegraph/` exists, CodeGraph becomes the primary exploration tool, and the file and search
tools cover what it does not return. Call `codegraph_explore` with a question or symbol names when
the codegraph MCP server is connected, and run `codegraph explore "<question or symbol names>"` when
only the codegraph CLI is installed.

The MCP tool may be listed as deferred rather than loaded, in which case load its schema by name
through tool search before calling it. Both access paths return the same output, which is the
verbatim line-numbered source of the matching symbols grouped by file, the call paths connecting
them, and a summary of what depends on them.

When `.codegraph/` is absent, explore with the file and search tools alone. Indexing a repository is
the user's decision, so do NOT create an index as part of exploration.

### Step 2: Determine project archetype and size

First detect the project archetype (Python-only, Python + C++ extension, C++ PlatformIO library or
firmware, or C# Unity) using the indicator table in `/project-layout`. Do not re-derive those
indicators here. The archetype determines which entry points, manifests, source extensions, public
API source, and test directory the phases target (see "Archetype-specific signals" after the phases).

Then select the exploration tier. Count first-party source modules with a scoped command rather than
listing the whole tree, for example `find src -name "*.py" | wc -l` (Python) or
`find . -name "*.cpp" -o -name "*.h" | wc -l` (C++/PlatformIO), paired with a top-level directory
listing.

ataraxis source uses GENERATED `.pyi` stubs (one per `.py`). They are purged during development
(`tox -e lint`) and only present at release (`tox -e stubs`), so a dev tree often has none. Treat
their absence as normal, never a gap. When present, exclude `.pyi` from file counts, dependency maps,
and test-coverage mapping, and always read the `.py` module (not the stub) for docstrings and the
documented API. Do not spend tokens reading or editing stubs.

| Tier   | Indicators                                               | Approach                      |
|--------|----------------------------------------------------------|-------------------------------|
| Small  | Single package, < 10 source files, no subpackages        | Single-pass exploration       |
| Medium | Multiple packages, or 10-50 source files                 | Structured four-phase         |
| Large  | 50+ source files AND (monorepo OR multiple entry points) | Parallel subagent exploration |

An MCP server alone does NOT force the Large tier, because MCP servers are common on single-package
ataraxis libraries. When indicators conflict, choose the lower tier (fewer subagents) unless the
source-file count alone clearly exceeds 50.

A CodeGraph index shifts the tier down by one, because the index already holds the structure that
the extra readers would otherwise reconstruct file by file. An indexed Large project is explored as
a Medium one, and an indexed Medium project is explored in a single pass. Spend the saved effort on
follow-up queries into the areas the first pass surfaced.

### Step 3: Execute exploration

Follow the approach for the determined tier. When a CodeGraph index is present, open each phase with
the queries listed in "CodeGraph-assisted exploration" below. Fall back to the file and search
tools only for what the index does not return, such as configuration files, environment variables,
and directory layout.

**Small projects:** Execute all four exploration phases yourself in a single pass.

**Medium projects:** Execute all four exploration phases sequentially, giving each phase focused
attention. Use the Agent tool with the `Explore` agent type for any phase that requires reading
many files. An indexed project rarely needs a subagent here, since one CodeGraph call returns what
a file-reading subagent would spend many calls collecting.

**Large projects:** Launch 2-3 Explore subagents in parallel using the Agent tool with the
`Explore` agent type. Assign each subagent a disjoint focus area so no surface is explored twice.
Tell each subagent that the repository is indexed and that it MUST query CodeGraph before reading
files, since subagents do not inherit that context:

- **Subagent 1** covers structure, entry points, and configuration: Phase 1 (feature discovery and
  configuration), excluding the public API and MCP surfaces assigned to Subagent 3
- **Subagent 2** covers architecture and code flow: Phase 2 (code flow tracing) and Phase 3
  (architecture analysis including import mapping and central component identification)
- **Subagent 3** covers the API surface, MCP tools, and quality: public API enumeration, MCP-tools
  enumeration, and Phase 4 (test coverage, error handling patterns, technical debt indicators)

Synthesize the subagent findings into a unified summary.

### Step 4: Present findings

Present the structured summary following the output format below. Do NOT make code changes during
exploration. Wait for user direction before proceeding.

---

## CodeGraph-assisted exploration

This section applies when Step 1 found a `.codegraph/` directory. Each phase below opens with a
CodeGraph query, and the file and search tools fill the remaining gaps.

| Phase                     | Open with this query                                                       |
|---------------------------|----------------------------------------------------------------------------|
| 1: Feature discovery      | The entry-point symbols named in `[project.scripts]`, plus each `__init__` |
| 2: Code flow tracing      | The entry-point symbol together with the core symbols it reaches           |
| 3: Architecture analysis  | The central symbols by name, read for their call paths and dependents      |
| 4: Implementation details | Each public symbol, read for its listed tests and callers                  |

Query with the symbol and file names that matter, or with the plain question itself. One capped call
usually answers a whole phase. Read these rules before querying:

- Name several related symbols in one query rather than issuing one query per symbol. The index
  returns them together with the call paths that connect them, which is the part a per-symbol query
  loses.
- Use the blast-radius summary for Phase 3. It reports the dependents of each symbol directly, which
  is the central-component ranking that would otherwise require reading every importer.
- Use the reported test references for Phase 4 test-coverage mapping, then confirm the gaps against
  the `tests/` tree, since the index reports the tests that exist rather than the modules that lack
  them.
- Do NOT re-read a file whose source a query already returned. The output is the current on-disk
  source, so a follow-up Read of the same file returns the same bytes at full token cost.
- Fall back to the file and search tools for what the index does not model, which includes
  `pyproject.toml` and `tox.ini` settings, environment variables, documentation, and the directory
  layout itself.
- Treat the index as lagging the working tree by about a second. After an edit, re-query rather than
  trusting an earlier result for the changed file.

---

## Exploration phases

Every exploration follows four phases regardless of project size. For small projects, phases may be
combined. For large projects, phases may be distributed across parallel subagents.

### Phase 1: Feature discovery

Identify the project's entry points, boundaries, and configuration.

1. **Orientation documents**: Read the repo-root `CLAUDE.md` (and `README.md`) for project purpose,
   architecture summary, and conventions before tracing source. Treat them as orientation, not ground
   truth for API details.
2. **Entry points**: CLI commands, API endpoints, main functions, GUI entry points. Check
   `pyproject.toml` for `[project.scripts]` and `[project.entry-points]` sections.
3. **Configuration mechanisms**: `pyproject.toml` settings, environment variables, CLI argument
   defaults, YAML/JSON/TOML config files, `.env` files, and dataclass-based configuration objects.
4. **Public API surface**: Classes, functions, and constants exported from `__init__.py` files.
   Note which modules use `__all__` to restrict exports.
5. **CLI command signatures**: For CLI-based projects, document each command's name, arguments,
   options, and purpose.
6. **MCP tools**: When an MCP server exists (an `interfaces/` package, `*_tools.py` modules, or an
   `<cli> mcp` script), enumerate each MCP tool with its name, parameters, and return/output shape.
   These tools are a major public surface for AI agents.

### Phase 2: Code flow tracing

Trace execution paths from entry points through the codebase layers.

1. **Call chain tracing**: Starting from each entry point identified in Phase 1, follow the call
   chain through to core logic. Document the path as a sequence of `file:function` references.
2. **Data transformations**: At each step in the call chain, note what data is passed, transformed,
   or returned.
3. **Layer identification**: Identify the abstraction layers the call chain passes through
   (e.g., CLI → validation → business logic → data access → output).
4. **Side effects**: Note where the code interacts with external systems (file I/O, network calls,
   subprocess invocations, environment variable reads).

### Phase 3: Architecture analysis

Analyze the structural relationships between components.

1. **Import/dependency mapping**: For each source module, identify what it imports from other
   project modules. Note the direction of dependencies (which modules depend on which).
2. **Central component identification**: Identify which files or classes are imported by the most
   other modules. These are the architecturally central components that changes will most likely
   ripple through.
3. **Design patterns**: Identify patterns in use (dataclasses, protocols, factories, decorators,
   context managers, abstract base classes, etc.).
4. **Cross-cutting concerns**: Note how logging, error handling, configuration, and validation are
   handled across the codebase.

### Phase 4: Implementation details

Examine specifics that inform future modifications.

1. **Test coverage mapping**: For each source module, identify the corresponding test file(s). Note
   any source modules that lack test coverage. ataraxis test files use the `<module>_test.py` suffix
   (e.g. `camera_test.py` for `camera.py`), not the `test_` prefix. See `/project-layout`.
2. **Key algorithms and data structures**: Document non-trivial algorithms, important data
   structures (dataclasses, TypedDicts, NamedTuples), and their locations.
3. **Error handling patterns**: How errors are raised, caught, and reported to the user. Note
   whether the project uses custom exception classes, logging, or a console utility.
4. **Technical debt and complexity**: Areas with high complexity, TODO/FIXME comments, known
   limitations, type: ignore suppressions, or noqa markers.

---

## Archetype-specific signals

The four phases are language-agnostic, and only the concrete signals change by archetype (see
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

1. **Project purpose**: 1-2 sentence summary
2. **Entry points and CLI commands**: Table of entry points with locations and descriptions
3. **Key components**: Table of components with locations and purposes
4. **Call chain summary**: Entry point → layer → core logic flow for primary paths
5. **Import dependency map**: Which modules depend on which, with central components highlighted
6. **Public API surface**: Exported classes, functions, and constants
7. **MCP tools**: When an MCP server exists, each tool with its parameters and return/output shape
8. **Configuration**: All configuration mechanisms discovered
9. **Test coverage**: Source module → test file mapping, noting gaps
10. **Notable patterns**: Design patterns, conventions, and cross-cutting concerns
11. **Areas of concern**: Technical debt, complexity hotspots, missing coverage

### Example output

For a complete worked example of this output format, see
[example-output.md](references/example-output.md).

---

## Related skills

| Skill                   | Relationship                                                                         |
|-------------------------|--------------------------------------------------------------------------------------|
| `/explore-dependencies` | Explores ataraxis dependency APIs, invoked alongside this skill                      |
| `/python-style`         | Provides Python coding conventions discovered during exploration                     |
| `/cpp-style`            | Provides C++ coding conventions discovered during exploration                        |
| `/csharp-style`         | Provides C# coding conventions discovered during exploration                         |
| `/readme-style`         | Provides README conventions when exploration reveals README issues                   |
| `/commit`               | Should be invoked after completing code changes informed by context                  |
| `/audit-project`        | Runs the change-mode gate over new code, after exploration informs the changes       |
| `/skill-design`         | Provides skill conventions when exploration reveals skill files                      |
| `/project-layout`       | Provides project directory and test-naming conventions referenced during exploration |
| `/pr`                   | Drafts the pull request summary for the branch this exploration informed             |
| `/release`              | Drafts the release notes that aggregate the merged pull requests                     |

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

Judgment items. No tool inspects the exploration output, so this checklist is its only enforcement.
Walk every one against the summary you are about to present.
- [ ] CodeGraph queried first for symbols, call paths, and dependents when an index was present
- [ ] No file re-read with Read after a CodeGraph query already returned its source
- [ ] No CodeGraph index created during exploration
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
- [ ] Archetype resolved from /project-layout's indicator table, with that archetype's entry points,
      manifest, and public API source used by every phase
- [ ] Exploration depth matches project size tier (small/medium/large), shifted down one tier when indexed
- [ ] Generated .pyi stubs excluded from file counts, dependency maps, and coverage mapping, with the
      .py read for the API and no missing stub reported as a gap

Command-settled item. `ls -d .codegraph` settles whether the repository carries an index, so run it
rather than inferring the index state from a partial directory listing.
- [ ] Repository root checked for a .codegraph/ directory before exploration began
```
