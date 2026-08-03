# Detection passes

The ordered sweep passes of `/audit-correctness`, and the named CEAI procedure they call. Each pass
asks one question of every line in scope. Pass 1 builds the ledgers every later pass consumes, so it
runs first and to completion on the main agent.

## Contents

- The coverage tiers
- Pass 1: Contract, state, and callgraph harvest
- Pass 2: Adversarial instantiation, and the CEAI procedure
- Pass 3: Domain boundary sweep
- Pass 4: Type and nullability sweep
- Pass 5: Unwind, resource, and durability sweep
- Pass 6: Sharing and interleaving sweep
- Pass 7: Call contract and sequence sweep
- Pass 8: Language-specific defect sweep
- Pass 9: Coverage-ranked hunt
- Pass 10: Test-oracle analysis

## One traversal, ten questions

Passes 2 through 10 are a CHECKLIST OF QUESTIONS rather than a schedule of re-reads. Read each file
ONCE and answer every applicable pass during that single traversal, carrying the pass list beside you.
Re-reading the file set once per pass costs nine extra traversals of every line in scope and surfaces
nothing the single traversal misses.

Pass 1 is the one exception. It runs to completion across the whole file set before any other pass
starts, because every later pass consumes the ledgers it builds.

---

## The coverage tiers

Pass 9 ranks the sweep by these tiers, and the skill's Step 2 assigns one to every line in scope.

| Tier | Meaning                                                                                        |
|------|------------------------------------------------------------------------------------------------|
| T0   | A file measured nowhere, whether by an `omit` entry or by carrying no test suite at all        |
| T1   | A statement deliberately excluded, by a coverage pragma or by an `exclude_lines` corpus entry  |
| T2   | A statement, or a partial branch where branch coverage is on, the report lists as missing      |
| T3   | A branch outcome the report cannot see, which is every branch arm while branch coverage is off |

Where `branch` is unset or false, T3 is the richest hunting ground this skill has, covering every `if`
without an `else`, every short-circuit operand, every ternary arm, every loop that can run zero times,
and every `except` whose `try` never actually raised, all while the gate reports success. Where
`branch = true`, those same outcomes are measured, so the unexercised ones surface as T2 and T3 stays
empty.

---

## Pass 1: Contract, state, and callgraph harvest

**Question:** What does this line promise, and where is that promise written down?

Read every source file top to bottom once and build three ledgers before hunting any defect. Read the
test suite and the build configuration as authority rather than auditing them, which means
`pyproject.toml` and `tox.ini` for Python, `platformio.ini` or `CMakeLists.txt` for C++, and the
assembly definitions for C#.

**CONTRACT ledger.** One row per callable, class, and module, holding its preconditions and
postconditions with each tagged by source:

| Tag  | Source of the promise                                                       |
|------|-----------------------------------------------------------------------------|
| NAME | The identifier itself, such as `validate_`, `_checked`, `is_`, or `ensure_` |
| SIG  | The parameter list, defaults, and keyword-only separators                   |
| ANN  | Type annotations, including the dtype inside an array annotation            |
| DOC  | The docstring, Doxygen block, or XML doc comment                            |
| ENF  | An adjacent enforcement point, such as a guard, an assertion, or a constant |

**STATE ledger.** One row per stateful attribute with its creating line, every reader, and every
destroyer. Add each class's CREATE, USE, and DESTROY method classification.

**CALLGRAPH ledger.** One row per call site with the callee, whether the callee is in-repo or
external, and whether its return value is consumed.

Read the FULL body of every callable while building these. A summary that looks wrong at the signature
is often satisfied deeper in the body, and behavior a docstring describes is often delegated to a
helper. Delegated behavior still belongs to the documented callable.

---

## Pass 2: Adversarial instantiation

**Question:** Is there an input that satisfies every stated precondition of this callable yet makes
one of its stated postconditions false?

Run the CEAI procedure below against every row of the contract ledger, highest fan-in callables first.
Apply the ownership ladder before keeping any finding.

### The CEAI procedure

Contract Extraction then Adversarial Instantiation. Several categories call it by name.

**CE1.** Read the full body before judging anything, and follow every call it delegates to.

**CE2.** Write the contract as two lists. PRECONDITIONS states what the caller must supply.
POSTCONDITIONS states the return value, its type, units, range, shape, and dtype, plus side effects,
exceptions raised, state left behind, idempotence, and thread-safety. Tag every item with its source
from the Pass 1 table.

**CE3.** For each postcondition, search four catalogs in this fixed order for a witness that satisfies
every precondition and falsifies that postcondition: domain boundary, branch arm, state sequence, and
interleaving.

**CE4.** Write the witness as a callable expression with concrete literal arguments.

**CE5.** Hand-trace the witness through the body, writing the concrete value after each transforming
statement, and stop at the first contradiction.

**CE6.** Record the witness expression and the trace verbatim. They ARE the evidence, and a candidate
without them is discarded.

---

## Pass 3: Domain boundary sweep

**Question:** What is the most extreme legal value that reaches this line, and what does the line do
with it?

For every line, take each value it consumes and instantiate the boundary catalog for that value's
kind.

| Kind            | Boundary values                                                                                 |
|-----------------|-------------------------------------------------------------------------------------------------|
| Sequence, array | Length 0, 1, 2 equal elements, all equal, all NaN, one non-finite, a zero axis, ndim off by one |
| Integer         | 0, 1, -1, dtype minimum, dtype maximum, and maximum plus one                                    |
| Float           | 0.0, -0.0, NaN, positive and negative infinity, a denormal, exactly the boundary constant       |
| String, path    | Empty, whitespace only, a missing path, a missing parent, a directory, a case clash, a symlink   |
| Optional        | None                                                                                            |
| Time            | Zero elapsed, a clock that moved backwards, a fixed-width counter at rollover                   |

Then check the four boundary constructs on the line: comparison strictness against the documented open
or closed interval, slice and index arithmetic where start equals stop and where stop equals the
length, every reduction or first and last access reachable with an empty input, and every loop
evaluated at zero and at one iteration, including any variable read after the loop that only the loop
body assigns.

Finally verify DOMINANCE. A guard that would exclude the boundary counts only when it runs on every
path reaching the line.

---

## Pass 4: Type and nullability sweep

**Question:** Can a value this line's declared type forbids arrive here, or can this line produce a
value its declared type forbids?

The question is the same in all three languages, and each declares its types differently. In Python
the declaration is the annotation, in C++ it is the parameter and return type together with `const`
and reference qualifiers, and in C# it is the declared type together with its nullability annotation.

For every declaration, enumerate its arms. In Python these are each union member, each `| None`, each
`Literal` value, each enum member, and the dtype inside an array annotation. In C++ they are each
alternative of a `std::variant` or a tagged union, each enumerator of an `enum class`, and the null
state of every pointer and `std::optional`. In C# they are each arm of a nullable reference or value
type, each enum member, and each pattern arm of a `switch` expression.

Locate the handler that dominates each arm. For a Python optional the handler is an identity check
against None rather than a truthiness check, because the style skills require identity for None. For a
C++ pointer or `std::optional` it is a null or `has_value` test on every path to the dereference. For a
C# nullable it is a null test, and a null-forgiving `!` operator asserts an arm the compiler could not
prove, so treat every one of them as an unhandled arm until a dominating test is found.

Walk every terminating path of every function declared to return a value and identify the paths that
fall off the end. In Python, remember that ruff RET503 reasons syntactically and does not follow
`NoReturn` through a call, so the sanctioned unreachable `return` after a terminating error call is
compliance rather than a defect. In C++, a path falling off the end of a non-`void` function is
undefined behavior, so route it to CPP_LOW_LEVEL_DEFECT with that consequence named.

Then sweep the boundaries no type system polices. For Python this is YAML and JSON deserialization,
CLI arguments, wire bytes, environment variables, `**kwargs`, `Any`, every `cast(...)`, and every
`# type: ignore`. For C++ it is every C-style cast, `reinterpret_cast`, `static_cast` that narrows,
`void*` round-trip, and byte buffer parsed into a struct. For C# it is every `as` cast, unboxing cast,
`dynamic` value, and deserialized payload.

Verify the actual dtype of every expression whose annotation names one, tracking NumPy promotion. True
division of integer arrays yields float64 and true division of float32 arrays stays float32. `np.mean`
returns float64 for integer input and the input's own dtype for floating input. `np.zeros` and
`np.empty` without `dtype=` default to float64, and a reduction returns a scalar rather than an array.

These NumPy rules apply to Python files. The C++ and C# width analysis lives in NUMERIC_DEFECT and
CPP_LOW_LEVEL_DEFECT, which cover integer promotion and implicit conversion.

---

## Pass 5: Unwind, resource, and durability sweep

**Question:** If control leaves this line abnormally, what is left half-done, what is left unreleased,
and what is left wrong on disk?

For every statement that can raise, which includes every call, index, conversion, and every terminating
error call, list the acquisitions and mutations already in flight above it in the same scope and the
obligations sitting below it. An obligation below a possible exit is a finding unless it lives in a
`finally` block or a context manager.

Walk every acquisition to its release and verify the release runs on EVERY path out: the normal return,
each early return, each `break` and `continue`, each raised exception, and each terminating error call.
A terminating error call is the highest-yield leak site in this framework, because a guard clause placed
after an `open()` never reaches the `close()`.

Check the loop dimension, where an acquisition inside a loop whose release sits outside holds every
handle but the last.

Then walk each stateful class's CREATE, USE, and DESTROY table and test every ordered pair the public
API permits: USE before CREATE, USE after DESTROY, DESTROY twice, CREATE twice, and DESTROY without
CREATE.

Finally sweep the durability dimension, which asks what a reader finds on disk after a crash, a kill,
or a second writer. Enumerate every write to a persisted artifact, which covers session markers,
descriptors, hardware-state and configuration files, feather and NPZ outputs, log archives, and
checksum files. For each one establish four things.

**Atomicity.** A write straight to the destination path leaves a truncated or half-written file when
the process dies mid-write. The durable form writes a temporary file in the same directory and renames
it into place, so name the destination and state which form the code uses. A rename across filesystems
is not atomic, so check that the temporary file shares the destination's directory.

**Ordering against the marker.** Where one artifact records that another is complete, the recording
must land after the data it vouches for. Write down the actual order of the data write, the flush, and
the marker write, then name the crash point between them that leaves the marker claiming a file that
is absent or partial.

**Checksum subject.** A checksum computed over an in-memory buffer verifies the buffer rather than the
bytes that reached the disk. Trace what the checksum consumed and what the verifier later reads, and
report the pair when they differ.

**Second writer.** Name every context that can write the same path, taking the contexts from the Pass
6 enumeration, and give the interleaving that leaves the file holding one writer's header over another
writer's body. Where the code claims a lock or a marker prevents this, check that the claim covers the
whole write rather than its opening.

A durability candidate carries the same evidence floor as every other. The trigger is the concrete
crash point or interleaving with its line, and the result is the concrete on-disk state a later read
observes.

---

## Pass 6: Sharing and interleaving sweep

**Question:** Who else can observe or modify this object between this line and the next?

Enumerate the execution contexts that ACTUALLY exist before anything else, and proceed only when at
least two of them reach the same state. Contexts to look for are threads, processes, futures, asyncio
tasks, signal handlers, exit hooks, camera and serial callbacks, interrupt service routines, Unity
main-thread callbacks, and the pytest-xdist workers implied by a distributed test run.

Build the shared-state set from module-level mutables, class attributes, shared-memory arrays and
values, queues, and the files and directories written by more than one context. For every item, list
every read and write with its owning context.

Apply the four concurrency tests: read-modify-write atomicity, publication ordering of data against
its flag, lock-ordering cycles across the whole file set, and reentrancy through callbacks. Write each
failing interleaving as a line-numbered trace.

In the same pass, sweep the single-threaded sharing forms, which need no second context at all:
mutable default arguments, class-body mutables used as if they were per-instance, mutated module-level
state, caches handing every caller the same mutable object, and returned references to internal state.

---

## Pass 7: Call contract and sequence sweep

**Question:** Does this call obey its callee's contract, with the right arguments in the right order
and the right units, and is its result consumed?

For every row of the callgraph ledger, open the callee and compare parameter by parameter: name
binding, positional order, units, applied defaults, and documented usage constraints such as "must be
called after", "not thread-safe", and "only once". Read in-repo source directly, and read the installed
package for an external callee. Verifying from memory is prohibited.

Flag every discarded return that carries a result or a status, and every ignored `[[nodiscard]]`.

Step up a level and recover the required operation order from its in-repo authority, which may be a
docstring stating an ordering requirement, an ordered command list, numbered stage comments, an ordered
enum, or a wire-format table. Build the actual order, diff the two, and check the state machine for
unguarded transitions, inescapable states, and silently returning default arms.

Finally run the unit ledger. Record the unit each variable carries from its name suffix, its
documentation, and the constant it derives from, compare declared units across every assignment and
call boundary, and verify each conversion's direction by substituting one concrete value.

---

## Pass 8: Language-specific defect sweep

**Question:** Does this line hold a defect specific to its language and archetype?

This pass runs over `.h`, `.hpp`, `.cpp`, and `.cs` files only.

**C++.** Determine the archetype first, because the embedded prohibitions bind embedded firmware alone
and a nanobind extension may use every construct they forbid. Sweep integer promotion, where arithmetic
on operands narrower than `int` promotes to a width that differs across boards. Sweep signed against
unsigned comparisons, object lifetime and dangling references, uninitialized reads, strict-aliasing
violations, misaligned access, packed-struct and endianness assumptions, static initialization order,
and interrupt-service-routine and `volatile` misuse.

**C#.** Build a lifecycle table per component listing what each lifecycle method reads and writes, then
check three ordering hazards: a field read in one component's initialization that another component
assigns in its own, a field assigned late but read in an earlier-running method, and a field assigned
in a method that never runs for an inactive prefab instance. Check event symmetry, so every
subscription has its matching unsubscription in the paired lifecycle method. Sweep Unity's overloaded
null, where a destroyed object compares equal to null while the underlying reference is live, and sweep
coroutines that outlive their component.

---

## Pass 9: Coverage-ranked hunt

**Question:** This line or branch outcome has never executed under test, so what happens the first time
it does?

Consume the ranking built in the workflow's Step 2 and work through the tiers in order, T0 first. Never
run bare `tox`, which would execute the source-mutating lint environment.

For every T0 region, confirm the module qualifies for exclusion. The qualifying kinds are CLI and other
command modules, the MCP server module with its tool modules, and process entry points. A module in the
omit list that matches none of those is real logic made invisible to the coverage gate, and that alone
is worth reporting.

For every T1 region, confirm the exclusion annotates the narrowest construct that covers the excluded
code, then hunt inside it with the ordinary category procedures.

For every T3 region, name the specific branch outcome that never runs and trace what happens when it
does. Every `if` without an `else`, every short-circuit operand, every ternary arm, every loop that can
run zero times, and every `except` whose `try` never actually raised belongs here.

An unexercised region is a place to look rather than a finding. It becomes a finding only when the
ordinary category procedures produce a concrete trigger and a concrete result inside it.

---

## Pass 10: Test-oracle analysis

**Question:** Which behaviors does the suite execute without pinning, so a wrong value would ship
undetected?

This is read-only reasoning over the assertion text. Editing source or tests to try a hypothesis is
prohibited.

Enumerate the test files for the languages in scope and build a symbol-to-tests index from their
imports and calls. In Python the suite is pytest under `tests/`, and it is the one place the style
skills permit private access and direct submodule imports, so expect both. In C++ it is the PlatformIO
or CTest suite. In C# it is the Unity Test Framework assemblies. A language whose suite does not exist
produces no findings in this pass, because Pass 9 already placed all of its code at T0.

Classify every assertion touching each symbol:

| Class        | Meaning                                                                    |
|--------------|----------------------------------------------------------------------------|
| PINNED-VALUE | Compares a concrete expected value, array, or structure against the result |
| PINNED-ERROR | Asserts a raised exception or an error return with its identifying text    |
| PINNED-STATE | Asserts on an attribute, a file's contents, or a recorded call's arguments |
| SMOKE        | Calls and asserts nothing, or asserts only that something is truthy        |

Flag the weak-oracle shapes: an exception assertion without a message match, truthiness assertions on
numeric or array results, approximate comparisons whose tolerance exceeds the effect under test, mocks
asserted only as called, and assertions on values the function returns unchanged from its input.

For each statement, name the smallest bug-shaped mutation and identify the assertion that would catch
it. Where none exists, the behavior is UNPINNED.

Cross-check every parametrized arm list against the code's own branch arms, because statement coverage
without branch coverage reports full coverage for a half-exercised conditional. Note fixtures whose
teardown or global reset explains why an otherwise obvious defect survives the suite.
