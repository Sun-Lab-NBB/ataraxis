---
name: audit-performance
description: >-
  Performs a thorough optimization audit of Python, C++, and C# source, hunting for untraceable
  numeric widths, algorithmic blowup, redundant allocation, hostile memory layout, and boundary,
  concurrency, and per-archetype language costs. Reports only findings whose cost is proven by
  execution multiplicity, with verbatim citations. Use when auditing a package, firmware, or Unity
  code for speed, memory, or numeric predictability, or when the user invokes /audit-performance.
user-invocable: true
---

# Performance optimization audit

Audits source code against its measured execution frequency, reporting only optimization findings
whose cost is proven by verbatim source citations and explicit cost arithmetic.

You MUST read this entire skill and load the reference files it names before starting an audit. The
verification checklist at the end is mandatory before submitting findings.

---

## Scope

**Covers:**
- Auditing single files, directories, or full project trees for optimization opportunities
- Numeric width traceability for arrays and scalars alike, so every value's width is predictable by
  reading the source
- Algorithmic complexity, allocation, copying, redundant work, and memory layout
- Boundary crossings, which covers compiled kernels, disk, threads, processes, and extensions
- Per-language and per-archetype runtime cost in Python, C++, and C#
- Proposing a benchmark for a finding whose payoff needs measurement, and asking before running it

**Does not cover:**
- Active and latent bugs, and behavior that disagrees with the stated contract (see `/audit-correctness`)
- Style, formatting, naming, and convention compliance, which includes a positional `dtype` argument,
  a bare `NDArray` annotation, and a missing `cache=True` (see `/audit-style`)
- Factual accuracy of documentation against source code (see `/audit-facts`)
- Code modifications and optimization work (this skill produces findings only)
- Codebase exploration (see `/explore-codebase`)

The boundary against `/audit-style` decides most disputes. That skill owns the FORM of a construct,
this skill owns its NUMERIC OR TEMPORAL CONSEQUENCE. A positional `dtype` argument keeps the width
explicit and traceable, so it stays a style finding. An ABSENT `dtype` hands the width to NumPy and
hides it from the source, so it belongs here. A construct that both skills can see enters this
report only with a cited runtime consequence attached.

---

## Language coverage

The audit applies to Python, C++, and C# equally, and a project holding none of NumPy, Numba, or a
Python package is fully in scope. Passes 1 and 3 through 8 are language-neutral, because execution
multiplicity, allocation, complexity, redundancy, boundary crossings, and memory layout are properties
of any program. Pass 2 traces numeric width and dispatches to a per-language procedure. Pass 9 holds
the constructs whose cost only one language's style skill names.

Numeric width traceability is the audit's central concern in every language, and each language
expresses it differently. In Python it is the NumPy `dtype`, which the source frequently leaves for
NumPy to choose. In C++ it is the fixed-width type from `<cstdint>`, which integer promotion and
implicit conversion silently change mid-expression. In C# it is the declared numeric type, which
implicit conversion and boxing silently change. The question is identical across the three, that a
reader can predict every value's width from the source alone.

---

## Evidence model

Three canonical terms carry every finding in this audit. Use these names consistently.

**Execution multiplicity** is how many times a line runs per program run, drawn from a fixed
vocabulary and always traced to the expression that bounds it.

| Multiplicity     | Meaning                                                                      |
|------------------|------------------------------------------------------------------------------|
| ONCE_PER_PROCESS | Module scope, `__init__`, setup, and calibration routines                    |
| ONCE_PER_CALL    | A function body whose caller runs once                                       |
| PER_SESSION      | Once per acquisition session or per processing run                           |
| PER_FILE         | Once per input file or per archive                                           |
| PER_CHUNK        | Once per batch, block, or window                                             |
| PER_RECORD       | Once per row, frame, packet, or event                                        |
| PER_ELEMENT      | Once per array element or per innermost loop iteration                       |
| PER_FRAME        | A Unity `Update`, `FixedUpdate`, `LateUpdate`, or coroutine, and its callees |

**Evidence class** records whether inspection settles the payoff.

| Evidence class      | Meaning                                                                 |
|---------------------|-------------------------------------------------------------------------|
| STATIC              | Cost is provable by reading, such as copy counts and complexity classes |
| MEASUREMENT-PENDING | Payoff needs a benchmark, such as layout, chunking, and new kernels     |

**Impact** orders the report and derives from multiplicity, per-execution cost, and payoff
confidence. Assign HIGH, MEDIUM, or LOW, and state the arithmetic that produced it.

---

## Workflow

You MUST follow these steps when this skill is invoked.

Copy this progress checklist into your response and check off items as you complete them:

```text
Audit Progress:
- [ ] Step 0: Plan produced and confirmed
- [ ] Step 1: Target resolved, tier selected, prerequisites recorded
- [ ] Step 2: Hot-path census complete
- [ ] Step 3: Sweep passes 2 through 9 complete
- [ ] Step 4: Findings categorized and rated
- [ ] Step 5: False-positive guards applied
- [ ] Step 6: Benchmark proposals presented
- [ ] Step 7: Sample verification complete
- [ ] Step 8: Coverage ledger assembled
- [ ] Step 9: Report produced
```

### Step 0: Produce audit plan and pause

Emit a plan before any sweep work fires. The plan must list:

- Files in scope (resolved absolute paths), grouped by language
- The prerequisites from Step 1, with the values you read
- Tier classification (small, medium, or large)
- Expected finding categories

Pause for user confirmation or a "proceed" signal. This catches misidentified targets before tokens
burn on the wrong scope. A user who narrows the scope here has that narrowing recorded in the Step 8
coverage ledger.

### Step 1: Resolve target and record prerequisites

Resolve the target into the set of files in scope. For a directory or project-root target, every
source file under the target is in scope. There is no "covered area" reduction. Enumerate with the
version control index, which reports the files that actually ship:

```bash
git ls-files '*.py' '*.pyi' '*.h' '*.hpp' '*.cpp' '*.cs'
```

Bind each file to the style skill that supplies its citable authority:

| File pattern            | Authority       |
|-------------------------|-----------------|
| `*.py`, `*.pyi`         | `/python-style` |
| `*.h`, `*.hpp`, `*.cpp` | `/cpp-style`    |
| `*.cs`                  | `/csharp-style` |

Record the prerequisites that apply to the languages in scope, before any verdict. A C++-only or
C#-only target records only the last two, and a project carrying no `pyproject.toml` records the
Python rows as N/A rather than treating their absence as a finding:

1. Python only. The NumPy version pin from `pyproject.toml`. NumPy 2.0 and above applies NEP 50, and
   earlier versions apply value-based casting for scalars. A dtype verdict that omits its regime is
   unfalsifiable.
2. Python only. Whether Numba is already a dependency, confirmed in `pyproject.toml` and by an
   existing import.
3. C++ only. The archetype of every C++ file, embedded (a `platformio.ini` at the project root) or
   extension (nanobind headers under a CMake build).
4. C# only. The target framework and whether the project is a Unity project, which decides whether a
   per-frame reachability set exists at all.
5. Every language. The fixed-width numeric types the project uses at its boundaries, taken from the
   wire formats, packed structs, and on-disk schemas it declares. These pin the widths Pass 2 traces.
6. Every language. Whether a `.codegraph/` directory exists. When it does, prefer `codegraph explore`
   over grep for call-site discovery, because it follows the dynamic-dispatch hops that establish
   multiplicity.

Classify the audit tier:

| Tier   | Indicators                     | Execution                                                                  |
|--------|--------------------------------|----------------------------------------------------------------------------|
| Small  | 1 file, under 500 lines        | Main agent, sequential                                                     |
| Medium | 2-10 files                     | Main agent, file-by-file                                                   |
| Large  | 10+ files or full project root | Parallel `general-purpose` sub-agents, one per file or per directory group |

Specify agent type per subsequent step:

| Step                  | Agent type                   | Why                                       |
|-----------------------|------------------------------|-------------------------------------------|
| Hot-path census       | main                         | Every sub-agent consumes one shared table |
| Sweep passes (Small)  | main                         | Sequential preserves citation precision   |
| Sweep passes (Medium) | main                         | Sequential preserves citation precision   |
| Sweep passes (Large)  | `general-purpose` (parallel) | Parallelizes across files                 |
| Categorization        | main                         | Needs synthesis across findings           |
| Guard application     | main                         | Trust boundary                            |
| Benchmark proposals   | main                         | Owns the question put to the user         |
| Sample verification   | main                         | Trust boundary                            |
| Coverage ledger       | main                         | Owns the record of what was swept         |
| Final report          | main                         | Format and ordering live here             |

Do NOT use the `Explore` agent type for sweep work. Explore returns summaries rather than verbatim
citations and breaks the "verbatim quote" discipline.

### Step 2: Hot-path census

Run Pass 1 from [detection-passes.md](references/detection-passes.md) to completion on the main
agent, before any fan-out. It annotates every line in scope with an execution multiplicity and
traces every loop bound to its source. Every later pass consumes this table, and no finding in this
audit is reportable without it.

Running the census per file inside a sub-agent would show each sub-agent only its own loops and
would lose the cross-file call-site evidence that establishes heat.

### Step 3: Run the sweep passes

Run passes 2 through 9 from [detection-passes.md](references/detection-passes.md) in order. Each
pass asks one question of every line, and the file also holds the two named width procedures that
Pass 2 dispatches to, DTYPE TRACE for Python and WIDTH TRACE for C++ and C#.

For every candidate, classify it against
[finding-catalog.md](references/finding-catalog.md), which supplies each category's definition,
mechanical detection procedure, required evidence, and severity guidance.

List ALL candidates in each pass. Do NOT stop at the first.

For Large-tier audits, spawn one `general-purpose` sub-agent per file or per directory group. Each
sub-agent receives the file paths, the Step 2 multiplicity table, the reference files, and the
output format. The main agent synthesizes after all sub-agents complete.

### Step 4: Categorize and rate

Assign every candidate a category from the catalog, an impact, an evidence class, and a confidence
tier. State the cost arithmetic that produced the impact rating.

| Confidence | Meaning                                                           |
|------------|-------------------------------------------------------------------|
| HIGH       | Verbatim source quote present, the cost arithmetic is mechanical  |
| MEDIUM     | Source quote present, the cost estimate requires interpretation   |
| LOW        | Pattern detected but the cost or multiplicity mapping is inferred |

### Step 5: Apply the false-positive guards

Walk every candidate through every guard in
[false-positive-guards.md](references/false-positive-guards.md), in order. The cold-path gate runs
first and removes the most candidates. Discard everything a guard rejects, and record the count of
discarded candidates for the coverage ledger.

### Step 6: Benchmark proposals

Collect every MEASUREMENT-PENDING finding into its own report section. For each, preserve the static
facts, then state the specific benchmark that would settle it: what to vary, what to measure, and on
what input. Ask the user whether to run them.

Run a benchmark only after the user agrees. Write any harness into a scratch directory outside the
project tree, leave the audited source untouched, and report measured numbers only when they came
from a run you actually performed. State an unmeasured speedup figure nowhere in the report.

### Step 7: Sample verification

Before emitting the report:

1. Sample 3 random findings from the candidate list (if fewer than 3, sample all).
2. Re-read the cited file line(s) and the cited multiplicity without looking at the original finding.
3. Re-derive whether the finding holds from scratch.
4. Discard any finding that does not survive re-verification.

This step catches the most common audit failure mode: asserting heat that the call sites deny.

### Step 8: Assemble the coverage ledger

Build the ledger that opens the report. It records what was swept, so a thin pass is visible rather
than silent:

```text
| Language | Files in scope | Files swept | Files skipped | Passes run |
|----------|----------------|-------------|---------------|------------|
| Python   | 24             | 24          | 0             | 1-8        |
| C++      | 6              | 6           | 0             | 1, 9       |
```

List every skipped file by path with its reason, and state the count of candidates the Step 5 guards
discarded. Skipping is allowed only when the user narrowed the scope in Step 0, when a file is
generated, or when a file is unreadable.

### Step 9: Produce the findings report

Use the output format below. Report HIGH and MEDIUM confidence findings by default. Report LOW
confidence findings only when the user explicitly requests them via `--include-low` or equivalent
invocation.

---

## Finding categories

Each category is defined in full in [finding-catalog.md](references/finding-catalog.md). Load that
file before classifying any candidate.

| Category                          | One-line definition                                                      |
|-----------------------------------|--------------------------------------------------------------------------|
| DTYPE_UNPINNED_AT_CREATION        | An array or scalar is created with its width left for NumPy to pick      |
| DTYPE_SILENT_PROMOTION            | A dtype widens partway through a computation with no explicit cast       |
| NATIVE_WIDTH_AND_CONVERSION       | A C++ or C# width is untraceable, promoted, narrowed, or boxed           |
| DTYPE_CHURN_AND_UNSTABLE_CONTRACT | Casts round-trip, or an output width is decided by runtime data          |
| MISSED_VECTORIZATION              | Element-wise work runs in the interpreter where one NumPy call would do  |
| ALGORITHMIC_COMPLEXITY_BLOWUP     | Cost is asymptotically higher than the task requires                     |
| REDUNDANT_RECOMPUTATION           | The same result is computed more than once, or computed and discarded    |
| HOT_LOOP_ALLOCATION               | Memory is allocated per iteration where one buffer would be reused       |
| REDUNDANT_COPY_OR_TEMPORARY       | Data is copied where a view, a slice, or an in-place operation serves    |
| MEMORY_LAYOUT_HOSTILE             | Memory is traversed in an order that fights the cache                    |
| NUMBA_KERNEL_OPPORTUNITY          | Branchy or sequentially dependent numeric work stays in the interpreter  |
| NUMBA_CONFIGURATION_DEFECT        | Numba is in use but configured so its benefit is forfeited               |
| PYTHON_INTERPRETER_OVERHEAD       | CPython object-model and dispatch costs on a data-scaled path            |
| IO_AND_SERIALIZATION_COST         | Per-call overhead, syscall count, or format choice dominates the work    |
| CONCURRENCY_AND_GIL               | Parallelism is absent where work is independent, or cancelled by the GIL |
| CPP_RUNTIME_COST                  | C++ constructs costing time, determinism, or size for their archetype    |
| CSHARP_RUNTIME_COST               | C# constructs allocating or working on a per-frame path                  |

---

## Output format

Open the report with the Step 8 coverage ledger. Report STATIC findings first, ordered by impact,
then the MEASUREMENT-PENDING section from Step 6. Group findings hierarchically: file, then
category, then impact.

Each finding uses this structure:

```text
[Category]: <category name from the catalog>
[Impact]: <HIGH | MEDIUM | LOW>
[Evidence]: <STATIC | MEASUREMENT-PENDING>
[Confidence]: <HIGH | MEDIUM | LOW>
Location: <path>:<line>-<line>
Multiplicity: <class> (bound: <expression>, source: <path>:<line>)
Current state: "<verbatim quote from the file>"
Cost: <explicit arithmetic or complexity class, stated for current and proposed>
Suggested fix: <concrete code change, described rather than applied>
```

When the same category is triggered multiple times within a file by one root cause, collapse to a
single finding with a count and representative line citations:

```text
[Category]: HOT_LOOP_ALLOCATION
[Impact]: MEDIUM
[Evidence]: STATIC
[Confidence]: HIGH
Location: processor.py:88, 141, 203 (3 occurrences)
Multiplicity: PER_RECORD (bound: len(records), source: processor.py:84)
Current state: "buffer = np.empty(shape=(window,), dtype=np.float32)"
Cost: 3 allocations x 4 KB x 50000 records, 600 MB churned, versus one hoisted buffer
Suggested fix: Hoist each buffer above its loop and pass it through the `out=` parameter.
```

For a MEASUREMENT-PENDING finding, replace the Cost line with the benchmark that would settle it,
stating what to vary, what to measure, and on what input.

---

## Discipline

You MUST adhere to the following discipline during every audit.

- Establish heat from an actual call site with a `<path>:<line>`. A function whose call sites resolve
  nowhere in the package is UNKNOWN and stays out of the report.
- Anchor every finding to a verbatim source quote and to explicit cost arithmetic.
- Cite the authority for a rule by skill name and reference file name, together with a verbatim quote
  of the rule.
- Report a construct that `/audit-style` also sees only with its runtime consequence established and
  cited.
- Keep every proposal inside the project's own conventions. A speedup that requires breaking a
  documented convention is reported with that conflict stated, and the decision left to the user.
- Preserve MEASUREMENT-PENDING findings with their static facts intact, and ask before running any
  benchmark.
- Never restructure, refactor, or optimize. This skill produces findings only.
- Treat `console.enable()` and `console.disable()` calls as correct at every library tier.

---

## Related skills

| Skill                   | Relationship                                                                               |
|-------------------------|--------------------------------------------------------------------------------------------|
| `/audit-correctness`    | Sibling audit for active and latent bugs, and for behavior that breaks its stated contract |
| `/audit-style`          | Sibling audit for style, formatting, documentation quality, and convention compliance      |
| `/audit-facts`          | Sibling audit for factual accuracy of documentation and docstrings against source code     |
| `/python-style`         | Supplies the dtype, NumPy, and Numba rules this skill cites as authority                   |
| `/cpp-style`            | Supplies the fixed-width type and per-archetype embedded rules this skill cites            |
| `/csharp-style`         | Supplies the Unity per-frame allocation rules this skill cites                             |
| `/pyproject-style`      | Defines where the NumPy and Numba version pins live, read as a Step 1 prerequisite         |
| `/explore-dependencies` | Provides ataraxis API snapshots, invoke before judging a library call's cost               |
| `/explore-codebase`     | Provides project structure context, invoke first when auditing an unfamiliar codebase      |

---

## Proactive behavior

Invoke this skill when the user asks to optimize, profile, or audit code for speed, memory use, or
dtype predictability. A request that names a directory or a repository covers every source file
under it by default, and the Step 0 plan is where the user narrows that scope.

Run after `/audit-facts` and `/audit-correctness`, and before `/audit-style`, when auditing the same
file end to end. Correctness fixes change the code an optimization would otherwise target, and style
compliance is settled last against the final form of the code.

Do NOT make code changes during the audit. Present findings and wait for user direction.

---

## Verification checklist

You MUST verify the audit output against this checklist before presenting it to the user.

```text
Performance Optimization Audit Compliance:
- [ ] Step 0 plan produced and confirmed by user before sweep began
- [ ] Step 1 prerequisites recorded for every language in scope, with the rows for absent languages marked N/A
- [ ] For Python in scope, the NumPy version pin recorded together with its promotion regime
- [ ] Pass 2 dispatched to DTYPE TRACE for Python files and WIDTH TRACE for C++ and C# files
- [ ] Scalar widths traced alongside array widths, including constants, reduction results, and extracted elements
- [ ] Tier classified (small/medium/large) and agent allocation matched the table
- [ ] For Large tier, parallel `general-purpose` sub-agents used and findings synthesized
- [ ] Hot-path census completed on the main agent before any fan-out
- [ ] Every loop bound traced to the expression and line that sets it
- [ ] Sweep passes 2 through 9 run in order, with pass 9 restricted to C++ and C# files
- [ ] Every finding carries an execution multiplicity of PER_CHUNK or hotter, or is a categorical prohibition
- [ ] Every finding assigned a category, an impact, an evidence class, and a confidence tier
- [ ] Every finding cites a file location <path>:<line> and quotes the source verbatim
- [ ] Every finding states explicit cost arithmetic for the current and the proposed form
- [ ] Every false-positive guard applied in order, with the discarded-candidate count recorded
- [ ] No hot-path speculation present, every heat claim backed by a call site <path>:<line>
- [ ] MEASUREMENT-PENDING findings collected into their own section with a concrete proposed benchmark
- [ ] Benchmarks run only after explicit user consent, with harnesses written outside the project tree
- [ ] Every speedup figure in the report came from a run that actually happened
- [ ] Micro-findings aggregated to at most one note per function
- [ ] Repeated instances of one root cause collapsed with counts
- [ ] Sample verification (3 random findings re-derived) complete
- [ ] Coverage ledger present, with every skipped file listed by path and reason
- [ ] LOW-confidence findings excluded unless explicitly requested
- [ ] No style, formatting, or convention findings appear (those belong to /audit-style)
- [ ] No correctness or bug findings appear (those belong to /audit-correctness)
- [ ] No proposal violates a documented project convention without that conflict being stated
- [ ] Findings ordered by impact, STATIC section before MEASUREMENT-PENDING section
- [ ] Suggested fixes are concrete code changes, not abstract recommendations
- [ ] No file modifications made during the audit
```
