# Detection passes

The ordered sweep passes of `/audit-performance`, and the two named width procedures Pass 2 dispatches to. Each pass
asks one question of every line in scope. Run them in order, because the two cheapest and highest-yield passes gate
everything after them.

Passes 1 and 3 through 8 apply to Python, C++, and C# alike. Pass 2 dispatches by language, and Pass 9 runs over C++ and
C# files only.

---

## Contents

- One traversal, nine questions
- Pass 1: Hot-path census
- Pass 2: Numeric width trace, and the DTYPE TRACE and WIDTH TRACE procedures
- Pass 3: Allocation, copy, and footprint census
- Pass 4: Loop-body interpretation cost
- Pass 5: Complexity nesting
- Pass 6: Redundancy and invariance
- Pass 7: Boundary crossings
- Pass 8: Memory layout
- Pass 9: Language-specific cost

---

## One traversal, nine questions

Passes 2 through 9 are a CHECKLIST OF QUESTIONS rather than a schedule of re-reads. Read each file ONCE and answer every
applicable pass during that single traversal, carrying the pass list beside you. Re-reading the file set once per pass
costs seven extra traversals of every line in scope and surfaces nothing the single traversal misses.

Pass 1 is the one exception. It runs to completion across the whole file set before any other pass starts, because every
later pass consumes the multiplicity table it builds.

---

## Pass 1: Hot-path census

**Question:** How many times does this line execute per run, and what bounds that count?

Run this pass first and to completion. Every later pass consumes its output, and no finding in this audit is reportable
without it.

Walk each file top to bottom and annotate every line with an execution multiplicity from the fixed vocabulary in the
skill's evidence model. For every loop, record the header's `<path>:<line>` and its bound expression, then trace the
bounding variable to its source and write down what determines it: an array `.shape`, a configuration field, a file's
row count, or a literal. A bound that resolves to a literal, an enum length, or a configuration value in the low tens
marks the region COLD, however deeply the loop nests.

For each function, determine multiplicity from its call sites rather than from its body. Locate every call site with
`codegraph explore` where a `.codegraph/` directory exists, because it follows the callback, registry, and
message-dispatch hops that establish whether a function runs per frame or per record. Fall back to grep elsewhere. A
function whose call sites resolve nowhere inside the package is marked UNKNOWN. A function whose only call sites live in
`tests/` is COLD.

Read the distribution's top-level `__init__.py` `__all__` before assigning either mark, and mark every symbol it exports
PUBLIC_API instead. Such a symbol is reachable from repositories this audit cannot read, so record its per-call cost in
the multiplicity table in place of a frequency: the work one call performs, and the input dimension that work scales
with. Carry that row forward like any other, since the later passes consume it to size a saving one call receives.

Treat these regions as COLD by default: module-level code, constructors, `SetupModule()`, calibration routines, CLI
entry points, MCP tool wrappers, viewer and GUI code, Unity editor scripts, and `tests/`.

For C# files, build the per-frame reachability set explicitly. Start at every `Update`, `FixedUpdate`, `LateUpdate`,
`OnGUI`, and coroutine body, then follow every method they call transitively, recording the depth of each hop. A
construct qualifies as PER_FRAME only inside that set.

The output is a multiplicity table covering every line in scope. Hand the same table to every sub-agent in a Large-tier
audit.

---

## Pass 2: Numeric width trace

**Question:** Does this line create, change, or fix a value's numeric width, and is that width explicit in the source?

The question is the same in every language, and the procedure that answers it differs. Dispatch on the file's language,
run the matching trace below, and record the result in one shared ledger.

| Language | Procedure   | Feeds                                                                |
|----------|-------------|----------------------------------------------------------------------|
| Python   | DTYPE TRACE | The three DTYPE categories, and the `layout` column Pass 8 completes |
| C++, C#  | WIDTH TRACE | NATIVE_WIDTH_AND_CONVERSION                                          |

### The DTYPE TRACE procedure

Execute this per array binding rather than per line, for Python files. Read the NumPy version pin first. Grep the
array-producing and width-changing surface to find the seeds:

```bash
grep -nE 'np\.(zeros|ones|empty|full|array|arange|linspace|frombuffer|fromiter|identity|eye)\(|\.astype\(|\.view\(|np\.asarray\(|dtype='
```

**Step 0.** Establish the promotion regime from the NumPy pin in `pyproject.toml`. NumPy 2.0 and above applies NEP 50,
under which a Python `int`, `float`, or `complex` is weak and adopts the array's dtype, while a NumPy scalar is strong
and promotes exactly as a same-dtype array would. Earlier versions apply value-based casting, under which a NumPy
scalar's dtype gave way to the narrowest dtype holding its value, so it could be demoted to the array's width. Write the
regime at the top of the trace, because every promotion verdict depends on it.

**Step 1.** Pick a seed. Trace scalars as well as arrays, because a scalar carries a dtype and sets the promotion floor
for every array it touches. Seeds are assignment targets holding arrays, parameters annotated `NDArray[...]`, and loop
variables bound to array slices. Seeds also include every NumPy scalar construction such as `np.uint64(0)`, and every
binding that receives an element extraction like `arr[i]` or a reduction result like `arr.sum()`. The last two seed
kinds are every module-level numeric constant consumed by array code, and every dataclass or configuration attribute
annotated with a bare `int` or `float` whose value reaches an array expression.

**Step 2.** Open a ledger with one row per operation and six columns:

```text
| line | operation | input dtype(s) | output dtype | EXPLICIT or IMPLICIT | layout |
```

**Step 3.** Fill row 0, the origin. An origin is a constructor, in which case apply the DTYPE_UNPINNED_AT_CREATION
rules, or a parameter, in which case take the annotation. An unparameterized annotation makes the whole trace
UNRESOLVED, and that is itself the finding.

**Step 4.** Walk forward through every use, applying the promotion rules. Arithmetic between an integer array and a
Python `float` yields float64 under both regimes. A Python `float` against a float array keeps the array's width under
both regimes, unless the literal's value overflows that width, which widens only under legacy casting. Record the
operand kinds rather than the regime alone. Mixing two array widths promotes to the wider. True division of integer
arrays yields float64. A reduction's default accumulator is often wider than its input. Indexing and `np.where` promote
to the common type of their branches. Mark each row EXPLICIT when a cast appears in the source, and IMPLICIT otherwise.

**Step 5.** Terminate each trace at a `return`, a disk write, or a jitted-kernel boundary. Compare the final dtype
against the function's return annotation and against every consumer's expectation.

**Step 6.** Report every IMPLICIT row whose width change is unjustified, and paste the completed ledger into the
finding.

### The WIDTH TRACE procedure

Execute this per variable and per expression, for C++ and C# files. It answers the same question the DTYPE TRACE
answers, using each language's own width rules.

**Step 0.** Record the target's width model. For C++, record the board or platform, because `int` is 16 bits on AVR and
32 bits on ARM and x86, and that width decides every promotion result. For C#, record the target framework and whether
the assembly enables `checked` arithmetic by default.

**Step 1.** Pick a seed. Seeds are every declared numeric variable, every function parameter and return, every
packed-struct field, and every value read from or written to a wire buffer or a file.

**Step 2.** Open a ledger with one row per operation and five columns:

```text
| line | operation | operand width(s) | result width | EXPLICIT or IMPLICIT |
```

**Step 3.** Fill row 0 from the declaration. A `<cstdint>` fixed-width type, a C# `byte`, `short`, `int`, `long`,
`float`, or `double`, and a packed-struct field are all EXPLICIT origins. A bare `int`, `short`, `long`, `unsigned`,
`size_t`, or `auto` in C++ is an UNPINNED origin, which is itself the finding, because the width varies by platform.

**Step 4.** Walk forward through every use. In C++, any arithmetic on operands narrower than `int` promotes both to
`int`, so `uint8_t` plus `uint8_t` computes at `int` width and truncates only on assignment back. A comparison mixing a
signed and an unsigned operand converts the signed side. A shift result takes the promoted left operand's width. In C#,
arithmetic on `byte`, `sbyte`, `short`, and `ushort` promotes to `int`, integer division truncates, and any implicit
widening conversion is silent while a narrowing one requires a cast.

**Step 5.** Terminate each trace at a return, a wire write, a disk write, or a boundary crossing into another language.
Compare the final width against the declared type at that boundary.

**Step 6.** Report every IMPLICIT row whose width change is unjustified, every UNPINNED origin, and every boundary where
the traced width disagrees with the declared one. Paste the ledger into the finding, and name the platform whenever the
result is platform-dependent.

---

## Pass 3: Allocation, copy, and footprint census

**Question:** Does this line allocate memory or copy data, how many times does it do so, and how many bytes are resident
at once while it runs?

Grep every allocating construct for the languages in scope, then join each hit against the Pass 1 multiplicity table and
compute total bytes churned as multiplicity times size, stating element count and element width for the size.

```bash
# Python
grep -nE 'np\.(zeros|ones|empty|full|array|concatenate|append|vstack|hstack|stack|copy)\(|\.copy\(\)|\.tolist\(\)'
# C++
grep -nE '\bnew\b|\bmalloc\(|std::(vector|string|map|unordered_map|set|deque)|\.resize\(|\.push_back\(|std::make_(unique|shared)'
# C#
grep -nE '\bnew\b|\.ToList\(\)|\.ToArray\(\)|string\.(Format|Concat)|\$"|\+= *"|List<|Dictionary<'
```

The three buckets below apply to every language. In C++ the GROWTH pattern is a `push_back` or `resize` inside a loop
over a container whose capacity was never reserved, and the fix is a single `reserve` before the loop. In C# it is
repeated string concatenation and repeated `ToList` or `ToArray` materialization, and any allocation reachable from a
per-frame method is a finding on garbage-collection grounds regardless of its size.

Sort the hits into three buckets.

**GROWTH.** The binding being assigned also appears as the first argument, as in `x = np.append(x, ...)`, or a
`np.concatenate` accumulates into one binding inside a loop. This is quadratic total copying. Quote the enclosing loop
header as evidence, because the identical call outside a loop is one linear copy and is correct.

**REUSABLE.** The allocation's shape and dtype stay invariant across iterations, so a hoisted scratch buffer plus `out=`
removes it.

**REMOVABLE.** A copy whose result is never mutated. Enumerate every downstream use and confirm that none of them
mutates the buffer before reporting.

### The footprint half

The three buckets above count allocation EVENTS. This half counts RESIDENT BYTES, which is a separate question and the
one the buckets cannot answer. A single allocation performed once per file allocates once and still exhausts the machine
when the file is a stack of imaging frames.

Walk every whole-input materialization. The shapes are a reader that returns the complete array rather than an iterator
or a memory-mapped handle, and a `read()` or `load` over a path whose size the input decides. The rest are a list built
from every record before any record is processed, a concatenate over every chunk, and a `DataFrame` or feather read
without column or row selection.

For each one, write the high-water expression as element count times element width, taking the count from the input
dimension that bounds it and the width from the Pass 2 ledger. State what the input dimension is in practice, citing the
configuration field, the acquisition rate, or the on-disk size that sets it. A footprint whose bound is a literal or a
configuration value in the low tens is not a finding, exactly as a loop bounded that way is COLD.

Then count SIMULTANEOUS liveness. Two full-size arrays alive at once double the high-water mark, so name every binding
holding a full-size buffer across the same statement and add their sizes. The common shapes are a source array kept
alive while its transformed copy is built, and an accumulator concatenated into a second full-size result before the
first is released.

Every footprint candidate is STATIC, because the arithmetic is provable by reading. The proposed fix names the bounded
form, which is chunked or windowed processing, a memory-mapped read, an iterator, a column or row subset, or an in-place
transform that releases the source.

---

## Pass 4: Loop-body interpretation cost

**Question:** Is this line doing per-element work that the interpreter must dispatch, and could one call replace all of
it?

For every loop that Pass 1 gave a data-scaled bound, apply the gate first. A loop inside a function decorated `@njit` or
`@numba.njit` is compiled, so skip it entirely.

For surviving loops, classify each statement in the body as element-wise arithmetic, a reduction, a conditional select,
or a side effect such as I/O, mutation of external state, or logging. A body made only of the first three is fully
vectorizable, so write out the exact replacement expression and name the operation: `np.sum`, `np.cumsum`, `np.diff`,
`np.where`, boolean-mask indexing, `np.searchsorted`, or `np.maximum.accumulate`.

A body containing any side effect stays an explicit loop and is not reportable. A body that is numeric but carries a
loop-carried dependency or an early exit is not vectorizable either, so route it to Pass 7 as a Numba kernel candidate.

---

## Pass 5: Complexity nesting

**Question:** What is this line's cost as a function of input size, and what does it multiply against?

Build a nesting tree per function from the loop headers Pass 1 recorded, annotating each node with its bound expression
and the source of that bound. Walk every root-to-leaf path and write the cost as a product of the bounds. Flag any path
where two or more factors are data-scaled. A path where every factor except one is a small literal or an enum length is
not a finding, whatever its depth.

Then sweep for the hidden linear factors that make a visually single loop quadratic. The pattern is identical in all
three languages, and only the container names change. In Python, membership with `in` against a list, tuple, or array is
linear while the same test against a set or dict is constant time. In C++, `std::find` and lookup in a `std::vector` or
`std::list` are linear while `std::unordered_map` and `std::unordered_set` are constant time, and `std::map` is
logarithmic. In C#, `List<T>.Contains` and `IndexOf` are linear while `HashSet<T>` and `Dictionary<K,V>` are constant
time. Resolve the container's declared type at its declaration line before reporting.

Also sweep index and count searches, a sort applied to already-sorted data, and the linear NumPy scans `np.where`,
`np.isin`, `np.argmax`, and whole-array comparisons appearing inside a loop body. `np.searchsorted` is a binary search
costing logarithmic time per query, so it is the replacement for a linear scan rather than an instance of one, and it
enters this pass as a proposed fix.

State both the current and the proposed complexity in big-O, with the variables bound to real quantities.

---

## Pass 6: Redundancy and invariance

**Question:** Has this value already been computed, or will it be computed again unchanged?

Confirm each invariant candidate formally. List the names the expression depends on, then search the loop body for any
rebinding or mutation of those names, citing the lines you checked. A confirmed invariant that is non-trivial is a
hoisting finding, which covers a call, a member chain of depth two or more, an allocation, a size query, a regex
compile, and a container literal. In C++ this includes a `.size()` or `.end()` recomputed in a loop condition and a
non-inlinable call whose arguments never change. In C# it includes a per-frame `GetComponent` or `Find` and a property
whose getter recomputes.

Next, normalize every call expression in each function into a canonical string and build a frequency table. For each
expression appearing twice or more with identical arguments, read the callee and confirm purity, meaning no I/O, no
mutation, no randomness, and no clock read, before reporting.

Propose memoization only after establishing that the argument domain is bounded and small and that the arguments are
hashable. Caching on array arguments or unbounded domains stays out of the report.

---

## Pass 7: Boundary crossings

**Question:** Does this line cross a boundary, interpreter to native, memory to disk, or thread to thread, and is the
crossing amortized over enough work?

Enumerate all four boundary kinds in one sweep.

**JIT.** Grep every `@njit`, `@jit`, `@vectorize`, and `@guvectorize` decorator and record its full arguments. Check for
`cache=True`, for an explicit `forceobj=True`, for `parallel=True` paired with `prange`, for `prange` on the outer loop
with `range` inside, and for `nogil` without threading. Numba 0.59 removed object-mode fallback and made `nopython`
default to `True`, so a bare `@jit` compiles exactly as `@njit`, and `forceobj=True` is the one remaining route into
object mode. For every `prange`, list every variable written in the body and classify each as parallel-indexed, a
recognized reduction, or a race. Locate each kernel's call sites, take the argument dtypes from the Pass 2 ledger to
detect signature churn, and count entries per run to detect dispatch-dominated kernels. Route the Pass 4 leftovers in as
new-kernel candidates, and only where Numba is already a dependency.

**DISK.** Grep the I/O surface, join each hit against Pass 1 to get its call count, group hits by the path expression
they operate on to find repeated opens, and check whether the target API accepts a batch.

**PROCESS AND THREAD.** Grep `ThreadPoolExecutor`, `ProcessPoolExecutor`, `multiprocessing`, `threading`, and `prange`.
For each task function submitted to a thread pool, read its body and decide whether the work releases the GIL. Bulk
NumPy operations, compiled extension calls, `@njit(nogil=True)` kernels, and blocking I/O release it, so threads are
correct there. Pure-Python loops and object work hold it, which is the finding. Cite the specific line that decides the
classification.

**EXTENSION.** For nanobind code, check that blocking calls release the GIL and that Python objects are untouched while
it is released.

---

## Pass 8: Memory layout

**Question:** Does this line touch memory in the order it is physically laid out?

Complete the `layout` column of the Pass 2 ledger for every array binding. Constructors yield C-contiguous arrays. `.T`
and a no-argument `.transpose()` reverse every axis, so over a C-contiguous input they yield an F-contiguous view at any
rank, and over a one-dimensional array they are a no-op that stays C-contiguous. An explicit permutation given to
`.transpose`, `np.swapaxes`, or `np.moveaxis` matches `.T` on two axes and generally yields a view that is neither
C-contiguous nor F-contiguous on three or more. `.reshape` preserves contiguity where compatible and copies otherwise.
Fancy and boolean indexing yield fresh C-contiguous arrays. Record the exact state in the `layout` column.

For each binding whose layout stops matching its consumer's traversal, find every subsequent bulk consumer and flag the
four cases that matter: passed into a jitted kernel, passed into a C extension, reduced over, or written to disk. In
each the consumer walks memory with a large stride.

Check axis order against access order. For each `axis=` argument and each loop that slices an array, state which axis
varies fastest in memory and whether the traversal matches. A column-wise loop over a C-contiguous two-dimensional array
strides the entire buffer once per slice.

Sweep for array-of-structures layouts, where a list of small objects or a structured dtype holds fields that are each
processed contiguously in a structure-of-arrays form. Record the object count and every field-extraction line. In C++
this is a `std::vector` of structs whose consumers each touch one member, and it also covers struct field ordering that
wastes space on padding, which is checked by listing each member's size and alignment in declaration order. In C# it is
an array of reference types, where each element is a separate heap object reached through a pointer, against an array of
structs that is contiguous.

For Python files this pass reads the `layout` column of the Pass 2 ledger. For C++ and C# files it is driven by the
declarations directly, because those languages have no equivalent ledger.

Every finding from this pass is MEASUREMENT-PENDING, because payoff is machine-dependent. The one exception is a
non-contiguous array passed directly into a jitted kernel or a C extension, where the degradation of the inner loop is
provable by inspection.

---

## Pass 9: Language-specific cost

**Question:** Does this line incur a cost that its language's style skill names as prohibited or discouraged for its
archetype?

This pass runs over `.h`, `.hpp`, `.cpp`, and `.cs` files only.

**C++.** Gate on archetype before evaluating anything, embedded against extension. In embedded files sweep for `new` and
`delete`, STL heap containers, `<iostream>`, `dynamic_cast` and `typeid`, `throw` and `try`, virtual inheritance, bare
`int`, `short`, and `long`, floating-point microsecond timing, and blocking `delay()` or `while()` waits. Resolve the
enclosing method for every blocking hit, because setup and calibration routines are exempt. In extension files sweep for
blocking calls without a GIL release, Python-object access while the GIL is released, and unnecessary by-value parameter
copies. In both archetypes sweep for non-`const` members and methods, stateless methods that are not `static`, and a
missing `final` on a leaf class or a leaf override, which blocks devirtualization.

**C#.** Work inside the per-frame reachability set Pass 1 built. Sweep for LINQ operators, per-frame
`GetComponent<T>()`, string concatenation and interpolation, `new WaitForSeconds(...)` allocated inside a coroutine
loop, multiple enumeration of a deferred query, boxing and defensive copies from non-`readonly` structs, `static
readonly` where `const` belongs, and instance methods that never touch `this`. Resolve every receiver's declared type
before reporting a named method, because a project type carrying a same-named method is a different call entirely.
