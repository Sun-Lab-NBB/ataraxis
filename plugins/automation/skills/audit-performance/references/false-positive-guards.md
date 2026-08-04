# False-positive guards

Every candidate finding of `/audit-performance` passes through these guards in order before it enters
the report. The cold-path gate runs first and removes the most candidates. Record the count of
discarded candidates for the report's triage header.

The codebase this audit runs against already applies most of the patterns the audit recommends, so a
guard that rejects a plausible-looking candidate is doing its job.

## Contents

- Guard 1: Cold-path gate
- Guard 2: No hot-path speculation
- Guard 3: Style boundary
- Guard 4: Readability and convention floor
- Guard 5: Guarded casts and defensive conversions are correct
- Guard 6: Loop membership must be proven for growth claims
- Guard 7: Leave compiled code alone
- Guard 8: No premature Numba
- Guard 9: No premature parallelism
- Guard 10: Resolve the receiver before reporting a named method
- Guard 11: Promotion must be unjustified to be a finding
- Guard 12: Archetype gate for C++
- Guard 13: MEASUREMENT-PENDING is a report state
- Guard 14: Aggregate the micro-findings
- Guard 15: Footprint requires an unbounded input dimension

---

## Guard 1: Cold-path gate

A finding is reportable only when Pass 1 assigned its line a multiplicity of PER_CHUNK or hotter, or
when it is a categorical prohibition that holds at any frequency. The categorical prohibitions are
heap allocation in embedded firmware, a missing `final` on a leaf class or leaf override, and LINQ
inside a Unity per-frame method.

`PEAK_MEMORY_FOOTPRINT` is exempt from this gate, because it is gated by SIZE rather than by
frequency. A whole-file materialization that runs once per process still exhausts the machine when the
file is large, so judge it against Guard 15 instead.

A `prange` race and `os.cpu_count()` arithmetic without a None guard stay off that list, because the
payload of each is a wrong value or a crash rather than a cost. `/audit-correctness` owns the race as
CONCURRENCY_DEFECT, and `/audit-style` owns the worker arithmetic through the `/python-style`
anti-pattern that prescribes `resolve_worker_count()`. Carry the race here only as a tag on the
NUMBA_CONFIGURATION_DEFECT finding recording that the parallelization it enables is invalid, and keep
the worker arithmetic out of this report.

Constructor bodies, module-level code, setup and calibration routines, CLI entry points, MCP tool
wrappers, viewer and GUI code, Unity editor scripts, and `tests/` are COLD by default, so report
nothing there on constant-factor grounds. A loop bounded by a literal, an enum length, or a
configuration value in the low tens stays COLD however deeply it nests.

---

## Guard 2: No hot-path speculation

Hotness comes from an actual call site with a `<path>:<line>`, found through `codegraph explore` or
grep. Phrases such as "probably called in a loop", "likely a hot path", and "could be
performance-critical" disqualify the finding.

A function whose call sites resolve nowhere inside the package is UNKNOWN and stays out of the report,
because an unused or externally called helper is no evidence of heat. A function whose only call sites
live in `tests/` is COLD.

---

## Guard 3: Style boundary

`/audit-style` owns the FORM of a construct, this audit owns its NUMERIC OR TEMPORAL CONSEQUENCE.

A positional `dtype` argument keeps the width explicit and traceable, which is all this audit cares
about, so it stays a style finding. A bare `NDArray` annotation, a missing `cache=True`, a
function-local import, and a missing `slots=True` are style findings first. Each enters this report
only with its runtime consequence established and cited, meaning an unresolvable dtype trace,
recompilation on every process start, a per-call module lookup on a hot path, or a data-scaled
instance count.

---

## Guard 4: Readability and convention floor

Never propose a change that breaks the project's own style rules in the name of speed. Specifically,
leave keyword arguments in place, keep full-word identifiers, keep comprehensions, which are both the
prescribed and the faster form, keep named intermediates that document a step, and keep well-named
helpers inline-free. Loop unrolling and replacing a dataclass with a tuple stay out of the report.

Treat `console.enable()` and `console.disable()` calls as correct at every library tier.

When the only available speedup requires breaking a documented convention, report the finding with
that conflict stated plainly and leave the decision to the user.

---

## Guard 5: Guarded casts and defensive conversions are correct

The pattern `x.astype(T) if x.dtype != T else x` is the prescribed no-copy form.
`np.ascontiguousarray(...)` placed immediately before a jitted kernel or a C extension call is a
required guard. A `.copy()` whose result is subsequently mutated, or that keeps a caller's buffer from
being aliased, is defensive and correct.

Before reporting any copy, enumerate every downstream use and state that none mutates the buffer. When
the binding escapes into code you cannot read, report UNVERIFIABLE rather than guessing.

---

## Guard 6: Loop membership must be proven for growth claims

`np.append` and `np.concatenate` are quadratic only when they accumulate across iterations. A single
`np.append` under an `if`, or one `np.concatenate` after a loop, costs one linear copy and is correct.
The list-accumulate-then-concatenate-once shape is the very fix this audit recommends, so flagging it
inverts the advice.

Quote the enclosing loop header as evidence before calling any growth pattern quadratic, and check for
the post-loop single-concatenate form first.

---

## Guard 7: Leave compiled code alone

Explicit `for` loops inside `@njit`, `@numba.njit`, and `@guvectorize` bodies are the correct form,
because Numba compiles them to machine code and rewriting them as NumPy calls would allocate
temporaries and slow them down. A loop inside a jitted function never enters MISSED_VECTORIZATION.

Scalar per-element code inside a kernel is likewise correct rather than interpreter overhead, and
`range` on inner loops under an outer `prange` is the prescribed nesting.

---

## Guard 8: No premature Numba

Propose a kernel only in a project that already depends on Numba, confirmed in `pyproject.toml` and by
an existing import.

Leave out any kernel whose body touches dicts of objects, dataframe objects, logging, or I/O, because
it cannot compile in nopython mode. Unicode strings compile in nopython mode, and so does a raise
whose exception constructor arguments are compile-time constants, so neither is on its own a reason to
drop a candidate. Leave out only the kernel that raises with a message assembled from runtime values.
State which of these you checked. Leave out any kernel entered many times over small arrays, where dispatch and unboxing
dominate. Every new-kernel proposal is MEASUREMENT-PENDING.

---

## Guard 9: No premature parallelism

Parallelism adds real complexity, new failure modes, and machine-dependent payoff, so it is never
recommended on static evidence alone.

Prove independence mechanically first by listing every variable the loop body writes and showing each
targets a loop-local name or a distinct index of a shared output array. Any shared accumulator or
order-dependent append disqualifies the loop and the proposal is dropped. A surviving proposal is
MEASUREMENT-PENDING, and the user is asked.

Report an existing thread pool as GIL-bound only after reading the task body and citing the specific
line showing the work is pure Python, because threads are correct for array work, extension calls, and
blocking I/O.

---

## Guard 10: Resolve the receiver before reporting a named method

A method name matching a known-expensive API is no evidence that the expensive API is being called.

The canonical case is a Unity component whose per-frame path calls `movement.Sum()`, which greps as a
LINQ violation. When `movement` is a project type whose own `Sum()` is a hand-written zero-allocation
loop over a fixed array, and the file imports only `UnityEngine`, the call allocates nothing and is
exactly the prescribed form. Before reporting LINQ inside a per-frame method, find the receiver's
declaration, confirm its type is an `IEnumerable<T>` rather than a project type, and confirm the file
imports `System.Linq`.

Apply the same resolution discipline to `in`, where a set differs from a list, to `.copy()`, where a
dict differs from a NumPy array, and to `.append`, where a list differs from `np.append`.

---

## Guard 11: Promotion must be unjustified to be a finding

A dtype change is a defect only when it is unintended.

Accumulator widening that prevents overflow, float64 used where the algorithm needs the precision, a
bool mask, and a width mandated by an external API or an on-disk schema are all correct. Before
reporting a promotion, check whether the narrower width would overflow or lose required precision
given the value ranges visible in the code, and state what you checked.

Record the NumPy version pin in the finding, because NEP 50 and legacy value-based casting give
different verdicts for the same expression.

---

## Guard 12: Archetype gate for C++

Establish embedded against extension before evaluating a single C++ construct. STL containers,
exceptions, dynamic allocation, and RTTI are prohibited in embedded firmware and allowed in nanobind
extension code. Applying the embedded prohibitions to extension source is the most common C++ false
positive.

Blocking `delay()` and `delayMicroseconds()` calls are permitted in setup and calibration routines, so
resolve the enclosing method before reporting a blocking call.

---

## Guard 13: MEASUREMENT-PENDING is a report state

When payoff genuinely resists static proof, which happens for cache and layout effects, chunk sizing,
compression and format choices, new Numba kernels, new parallelism, constant-factor interpreter
overheads, and cache hit rates, report the finding with its static facts intact, mark it
MEASUREMENT-PENDING, state the specific benchmark that would settle it, and ask the user whether to
run it.

Silently dropping such a finding, silently running a benchmark, and asserting an unmeasured speedup
figure are all prohibited.

The opposite error matters equally. Allocation counts, copy counts, complexity classes, and dtype
widths are provable by reading, so attaching MEASUREMENT-PENDING to them would make the audit useless.

---

## Guard 14: Aggregate the micro-findings

Attribute-lookup hoisting, global-name binding, small-call inlining, and similar constant-factor items
are collapsed into at most one aggregated note per function, citing representative lines with a count,
and reported only for loops whose bound Pass 1 proved data-scaled. A report padded with dozens of them
buries the real findings and pressures the reader toward unreadable code.

The same collapsing applies to repeated instances of one root cause within a file, which become one
finding carrying a count and representative line citations.

---

## Guard 15: Footprint requires an unbounded input dimension

A whole-input materialization is a finding only when the input dimension that bounds its size has no
ceiling the project sets. Trace that dimension to what fixes it, which is a configuration field, an
acquisition rate and duration, a wire-format field count, or the on-disk size of a file the pipeline
accepts, and state the value or the range you found.

Four shapes produce no finding. A materialization bounded by a literal or a configuration value in the
low tens holds a trivial buffer. A whole-file read of an artifact the project itself writes at a
bounded size, such as a session marker, a descriptor, or a configuration file, is bounded by
construction. A full read that the very next statement reduces to a scalar or a small summary still
peaks, so it IS reportable, while a full read the consumer genuinely needs in one piece, such as an
array a single vectorized call transforms end to end, is not.

The fourth is the inverted recommendation. Accumulating chunks in a list and concatenating once is the
prescribed fix for quadratic growth, so report its peak only when the joined result and the chunk list
are both alive and both full size, and say so explicitly rather than flagging the pattern itself.

Never propose chunking, windowing, or memory mapping without naming the window size and confirming the
consumer tolerates a partial view. A proposal that would change the result the current code produces
belongs to no report.
