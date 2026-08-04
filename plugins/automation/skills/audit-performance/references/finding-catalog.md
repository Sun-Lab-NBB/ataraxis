# Finding catalog

The eighteen finding categories of `/audit-performance`. Each entry gives the category's definition, its mechanical
detection procedure, the evidence a report entry must carry, and its impact guidance. Classify every candidate against
exactly one category, choosing the most specific one, and list any others as tags.

Detection runs from the sweep passes in `detection-passes.md`, which Step 3 of `SKILL.md` loads before any candidate is
classified. Each entry's detection section therefore names the pass that supplies the procedure and adds only the checks
specific to that category.

---

## Contents

- DTYPE_UNPINNED_AT_CREATION
- DTYPE_SILENT_PROMOTION
- DTYPE_CHURN_AND_UNSTABLE_CONTRACT
- NATIVE_WIDTH_AND_CONVERSION
- MISSED_VECTORIZATION
- ALGORITHMIC_COMPLEXITY_BLOWUP
- REDUNDANT_RECOMPUTATION
- HOT_LOOP_ALLOCATION
- PEAK_MEMORY_FOOTPRINT
- REDUNDANT_COPY_OR_TEMPORARY
- MEMORY_LAYOUT_HOSTILE
- NUMBA_KERNEL_OPPORTUNITY
- NUMBA_CONFIGURATION_DEFECT
- PYTHON_INTERPRETER_OVERHEAD
- IO_AND_SERIALIZATION_COST
- CONCURRENCY_AND_GIL
- CPP_RUNTIME_COST
- CSHARP_RUNTIME_COST

---

## DTYPE_UNPINNED_AT_CREATION

**Definition.** A value is created without its numeric width pinned in the source, so NumPy's default rules decide it
rather than the author. The width is therefore untraceable, is frequently twice what the computation needs, and silently
sets the promotion floor for every downstream operation the value joins. This covers arrays created with no `dtype=`
keyword, and equally covers SCALARS, because a Python `int` or `float` reaching array code carries the interpreter's
width rather than a chosen one.

**Detection.** Grep every array-producing constructor. Read each call and sort it into one of three states: PINNED,
where a `dtype=` keyword is present, POSITIONAL, where a dtype is passed by position, and UNPINNED, where no dtype
argument appears. Report UNPINNED only. POSITIONAL keeps the width explicit and traceable, so it belongs to
`/audit-style`.

Sweep the scalar surface with equal care, because it is the half auditors skip. Report a module-level numeric constant
consumed by array code and left as a bare Python literal where a NumPy scalar of the matching width belongs. Report
equally a dataclass or configuration attribute annotated `int` or `float` whose value crosses into an array expression
or a wire format. Report a threshold or bound compared against a narrow array, and a scalar returned from a public
function whose annotation names no width. In each case name the width the value actually carries and the width the
consumer needs.

For each UNPINNED hit, write down the dtype NumPy actually produces. `np.zeros`, `np.ones`, `np.empty`, `np.full` with a
float fill, and `np.linspace` produce float64. `np.arange` with integer arguments produces the platform default integer,
which is int64 on every 64-bit platform under NumPy 2.0 and above, and int32 on 64-bit Windows under NumPy 1.x.
`np.array` on a Python sequence produces the `np.result_type` of the elements. The `*_like` constructors inherit from
their source, so trace the source instead of reporting the call.

Compare the produced dtype against the function's annotation, against the dtype of every array it is combined with, and
against the width the on-disk or wire format requires.

**Evidence.** STATIC. Cite the creation call's `<path>:<line>`, quote the call verbatim showing no `dtype=` keyword, and
name the dtype NumPy produces. Prove the width is wrong by quoting either the conflicting annotation with its line or
the dtype and creation line of the array it combines with. Cite no style rule for this category. `/python-style`
prohibits an unparameterized `NDArray` annotation and a positional dtype argument, and says nothing about an absent
constructor dtype, so this audit is its sole owner. Anchor the finding on the produced width and the consumer it
conflicts with.

**Impact.** MEDIUM by default. HIGH when the array exceeds roughly a million elements, when it feeds a jitted kernel, or
when it is the declared return value of a public function whose annotation claims a narrower width.

---

## DTYPE_SILENT_PROMOTION

**Definition.** A value's dtype widens or changes kind partway through a computation with no explicit cast in the
source. Causes include arithmetic against a Python literal, mixing two arrays of different widths, true division of
integers, a reduction whose default accumulator is wider than its input, and indexing or `np.where` whose branches
disagree. The dtype at the end of the pipeline becomes unpredictable by reading, and the computation runs wider, often
at half the SIMD throughput, than intended.

Scalars promote on the same rules and belong here too. An element extracted with `arr[i]` is a NumPy scalar carrying the
array's dtype. A reduction such as `sum` or `mean` returns a scalar whose width often exceeds its input's, and a scalar
that later re-enters an array expression carries whatever width it accumulated. Trace a scalar's width across a function
boundary exactly as you trace an array's.

**Detection.** Run the DTYPE TRACE procedure from `detection-passes.md`, per array binding. Report every row the ledger
marks IMPLICIT whose width change is unjustified.

**Evidence.** STATIC. Paste the completed dtype ledger into the finding, every row from origin to the flagged step, each
carrying `<path>:<line>`. The flagged row must show input dtype, output dtype, and the operator responsible. Name the
promotion rule that fires and the NumPy regime under which it fires, citing the version pin.

**Impact.** HIGH when the promotion precedes a large reduction, a jitted kernel, or serialization, where it changes the
on-disk width. MEDIUM on a mid-size intermediate.

---

## DTYPE_CHURN_AND_UNSTABLE_CONTRACT

**Definition.** Two related defects in dtype discipline. CHURN casts the same data back and forth, each cast paying a
full array copy. It covers `astype` round-trips, a cast that restores a width an earlier implicit promotion destroyed,
and a cast that is a no-op because the array already holds that dtype. UNSTABLE CONTRACT lets a function's output width
be decided by its input or by runtime data, so no caller can predict it, which covers an unparameterized array
annotation, a dtype taken from a variable, and branches that return different widths.

**Detection.** Grep every cast. For each `.astype(X)`, walk the DTYPE TRACE upward to the receiver's immediately
preceding dtype, then classify. A receiver already holding X under an unguarded call is an unconditional no-op copy, so
report it. A receiver already holding X under a guarded call is the prescribed no-copy pattern, so leave it alone. A
receiver that reached its dtype through an implicit promotion this cast now undoes is a pair, so report it and point the
fix at the promotion site. Two or more casts of one buffer with no intervening operation requiring the intermediate
width is a round-trip, so report it with the copy count.

For UNSTABLE CONTRACT, find every signature whose array width is generic, every `dtype=` bound to a variable, and every
function with two `return` statements of differing width.

**Evidence.** STATIC. For churn, cite every cast in the round-trip with the dtype before and after each, the count of
full-array copies, and the array's element count or shape. For a no-op cast, quote the producing line proving the
receiver already holds the target dtype. For an unstable contract, quote the signature verbatim, cite the line binding
the dtype to a variable or the two divergent returns, and cite at least one caller forced to cast defensively.

**Impact.** MEDIUM by default. HIGH when the churn sits on a per-record or per-frame path, when a round-trip copies an
array above roughly a million elements, or when the unstable contract forces defensive casts at three or more call
sites.

---

## NATIVE_WIDTH_AND_CONVERSION

**Definition.** The C++ and C# counterpart of the three dtype categories. A value's numeric width is untraceable from
the source, or it changes without an explicit cast. Covers a bare `int`, `short`, `long`, `unsigned`, or `size_t`
declaration whose width varies by platform, and integer promotion that computes an expression at a width neither operand
declares. Also covers a signed against unsigned comparison that converts the signed side, an implicit widening
conversion, a narrowing assignment that truncates, and boxing that moves a value type onto the heap. The last case is a
boundary where the traced width disagrees with the declared wire, packed-struct, or on-disk width.

**Detection.** Run the WIDTH TRACE procedure from `detection-passes.md`, per variable and per expression. Report every
UNPINNED origin, every unjustified IMPLICIT row, and every boundary disagreement.

**Evidence.** STATIC. Paste the completed width ledger into the finding, every row from declaration to the flagged step,
each carrying `<path>:<line>`. The flagged row must show operand widths, result width, and the operator responsible.
Name the platform whenever the result is platform-dependent, such as the difference between a 16-bit and a 32-bit `int`.
Quote the declared boundary width with its line for a boundary disagreement, and cite the fixed-width-type rule from
`/cpp-style` or the numeric-type rule from `/csharp-style` for an UNPINNED origin.

**Impact.** HIGH for a width that reaches a wire buffer, a packed struct, or persisted data, and for a promotion whose
result differs across the project's target boards. MEDIUM for boxing on a data-scaled path and for a narrowing
assignment on an internal path.

---

## MISSED_VECTORIZATION

**Definition.** Element-wise work over array data runs in the Python interpreter one element at a time, where a single
NumPy call would express the same computation as one C-level loop. Covers explicit loops indexing an array, list
comprehensions over array elements that are rebuilt into an array, `np.vectorize` used in the belief that it accelerates
anything, per-element function calls, and manual accumulation that a reduction already provides.

**Detection.** Follow Pass 4 of `detection-passes.md`. Apply the jitted-function gate first, classify every statement in
the body, and name the exact replacement expression.

**Evidence.** STATIC. Cite the loop or comprehension span, and quote the header and body verbatim. State the trip count
with the variable that bounds it and the line where that variable is set, and write out the proposed NumPy expression in
full. State explicitly that every statement in the body was checked and performs no I/O, no mutation of external state,
and no logging.

**Impact.** HIGH when the trip count scales with a data dimension and the body is pure. MEDIUM when the trip count is
moderate or the replacement reads worse than the loop.

---

## ALGORITHMIC_COMPLEXITY_BLOWUP

**Definition.** Code whose asymptotic cost exceeds what the task requires. Covers nested iteration producing quadratic
behavior where a hash join, a sort, or a vectorized set operation is linear or n-log-n. Also covers a linear search
performed inside a loop over another collection, repeated sorting of data that is already sorted, and mappings rebuilt
on every call.

**Detection.** Follow Pass 5 of `detection-passes.md`. Distinguish a join, where the inner loop searches for a match
against the outer element and belongs here, from a loop-invariant inner body, which belongs to REDUNDANT_RECOMPUTATION.

**Evidence.** STATIC. Cite each loop header in the nest, and give the trip-count expression for each with its bound
traced to where it is set. Write the cost as a product, and for membership findings give the container's declared type
and its construction site. State the current and the proposed complexity in big-O with the variables bound to real
quantities.

**Impact.** HIGH whenever both bounds scale with data volume. MEDIUM when one bound is data-scaled and the other is
bounded by configuration in the low hundreds.

---

## REDUNDANT_RECOMPUTATION

**Definition.** The same result is computed more than once, or computed and then discarded. Covers loop-invariant
expressions evaluated every iteration, and pure functions called repeatedly with identical arguments over a small
bounded domain. Also covers derived values recomputed on each access, validation repeated per element that belongs once
at the function boundary, and work whose result is written and never read.

**Detection.** Follow Pass 6 of `detection-passes.md`.

**Evidence.** STATIC. Cite the redundant computation, and quote the enclosing loop header with its trip count so the
repetition factor is explicit. Quote the expression verbatim, and prove invariance by naming every name the expression
depends on and citing the lines checked to show none is rebound inside the loop. For a memoization proposal, state the
argument domain, its bound, the source of that bound, and that the arguments are hashable.

**Impact.** HIGH when the repetition factor is data-scaled and the repeated work is an allocation or a full array scan.
MEDIUM for attribute-chain and `len()` invariants in hot loops.

---

## HOT_LOOP_ALLOCATION

**Definition.** Memory is allocated on a path that executes many times, where one buffer could be allocated once and
reused. Covers array construction inside a loop body, arrays grown by repeated `np.append`, `np.concatenate`, or
`np.vstack`, each of which copies the whole accumulated result, temporaries created per iteration, and computations that
accept an `out=` destination and go without.

**Detection.** Follow Pass 3 of `detection-passes.md` and report the GROWTH and REUSABLE buckets. Verify loop membership
before calling any growth pattern quadratic. Confirm the correct pattern is absent before reporting, because
accumulating into a Python list and calling `np.concatenate` once after the loop is the fix this category recommends.

**Evidence.** STATIC. Cite the allocation, quote the enclosing loop header with its trip count and the source of the
bound, give the per-iteration allocation size as shape times dtype width with both stated, and give the total bytes
churned. For a GROWTH finding, show the arithmetic producing the quadratic total and quote the loop header proving the
call sits inside a loop. For an `out=` proposal, confirm the destination shape and dtype stay invariant across
iterations.

**Impact.** HIGH for growth patterns, which are quadratic and always worth fixing, and for per-iteration allocation with
a data-scaled trip count. MEDIUM for a fixed small allocation in a moderately hot loop.

---

## PEAK_MEMORY_FOOTPRINT

**Definition.** The bytes resident at one moment scale with the size of the input, where a bounded form would hold a
window instead. Covers a reader that materializes a whole file where a memory-mapped handle or a chunked read serves,
and a list or array built from every record before any record is processed. Also covers a concatenate that holds every
chunk and the joined result at once, a full table read where a column or row subset is used, and a transform that keeps
its source alive while building a full-size destination.

This is the category that counts RESIDENT BYTES. `HOT_LOOP_ALLOCATION` counts allocation EVENTS, so a single allocation
performed once per file belongs here rather than there, and a small buffer allocated a million times belongs there
rather than here.

**Detection.** Run the footprint half of Pass 3 in `detection-passes.md`. Write the high-water expression as element
count times element width, then add every full-size binding alive across the same statement.

**Evidence.** STATIC. Cite the materializing call and quote it verbatim. Give the high-water arithmetic with the element
count and the element width stated separately, and cite the line that bounds the count together with what sets it in
practice, meaning the configuration field, the acquisition rate, or the on-disk size. Enumerate every full-size binding
alive at the peak with its line. Name the bounded replacement concretely, and state the window or chunk size it would
hold.

**Impact.** HIGH when the high-water mark scales with an acquisition dimension that has no ceiling in the configuration,
and when two full-size copies are alive at once. MEDIUM when the input is bounded by a configured maximum that leaves
the footprint large but predictable.

---

## REDUNDANT_COPY_OR_TEMPORARY

**Definition.** Data is copied where a view, a slice, or an in-place operation serves. Covers `.copy()` on data that is
never mutated, `np.array(existing_array)` where `np.asarray` avoids the copy, and list-then-array construction that
materializes a Python list purely as scaffolding. Also covers defensive copies at boundaries the callee never writes to,
`.tolist()` round-trips, and materializing a full intermediate where a slice would do.

**Detection.** Grep every copy. For each `.copy()` or `np.array(x)` where `x` is already an array, find every subsequent
use of the copy. A copy with no mutating use is removable, where mutating means item assignment, an in-place operator, a
ufunc with `out=` targeting it, or storage beyond the caller's lifetime. A copy with any mutating use is defensive and
correct.

Trace the binding to confirm the input type before proposing `np.asarray`. For list scaffolding, a list or comprehension
that exists only to be converted immediately allocates and frees one boxed Python object per element, so propose the
direct array expression or `np.fromiter`.

**Evidence.** STATIC. Cite the copy, give the array's shape and dtype and therefore its byte size, enumerate every
downstream use of the copied binding with line numbers, and state explicitly that none of them mutates it. State the
aliasing analysis, naming which other bindings would alias the buffer after the change and whether any of them is
mutated.

**Impact.** HIGH when the copied array is large or the copy sits inside a loop. MEDIUM for mid-size copies and for list
scaffolding over data-scaled element counts.

---

## MEMORY_LAYOUT_HOSTILE

**Definition.** Data is laid out or traversed in an order that fights the cache and the prefetcher. Covers iterating a
C-contiguous array along its slowest-varying axis, reductions applied along the wrong axis, and transposes producing
non-contiguous views used in bulk. Also covers array-of-structures layouts where structure-of-arrays would let each
field be processed contiguously, chunk sizes chosen without reference to the working set, and non-contiguous arrays
passed into Numba or C extensions.

**Detection.** Follow Pass 8 of `detection-passes.md`.

**Evidence.** STATIC for the layout claim itself. Cite the layout-changing operation, give the array's shape and dtype,
state the resulting layout explicitly as C-contiguous, F-contiguous, or non-contiguous with its stride pattern, cite
every consumer that traverses it, and name the axis each consumer walks. For array-of-structures findings, give the
object count, the number of distinct fields extracted, and each extraction line. MEASUREMENT-PENDING for the payoff,
except where the consumer is a jitted kernel or a C extension.

**Impact.** MEDIUM by default, because payoff is machine-dependent. HIGH when a non-contiguous array is passed directly
into a jitted kernel or a C extension.

---

## NUMBA_KERNEL_OPPORTUNITY

**Definition.** A scalar, branchy, or sequentially dependent computation runs in the interpreter, or as a chain of NumPy
calls each allocating a full temporary, where a `@numba.njit` kernel would compile it to machine code and fuse it into
one pass. This is the complement to MISSED_VECTORIZATION, covering the work NumPy expresses poorly.

**Detection.** Confirm the project already depends on Numba before proposing any kernel. Take candidates from the loops
that failed the Pass 4 vectorization test. Keep those whose body is numeric and carries a loop-carried dependency, an
early exit a vectorized form would not short-circuit, per-element branching that `np.where` would express only by
computing both branches, or a running state machine over samples. Also treat as candidates the NumPy expression chains
of three or more operations over the same large arrays, counting one temporary per binary operator and per ufunc without
`out=`. Verify nopython compatibility by searching the body for constructs Numba rejects.

**Evidence.** STATIC facts plus MEASUREMENT-PENDING payoff. Cite the candidate span and quote the body verbatim. Name
the specific reason it resists vectorization by naming the loop-carried dependency variable, the early-exit line, or the
branch, and give the trip count with its source. Enumerate which unsupported constructs you searched for and did not
find, give every argument dtype from the DTYPE TRACE, and state the number of kernel entries per run.

**Impact.** MEDIUM as a proposal, never HIGH on static evidence alone.

---

## NUMBA_CONFIGURATION_DEFECT

**Definition.** Numba is already in use but configured so its benefit is forfeited. Covers a missing `cache=True`, an
explicit `forceobj=True` kernel on a hot path, and `parallel=True` without `prange` or with `prange` on an inner loop.
Also covers `prange` over a loop carrying a cross-iteration race, signature churn from unpinned argument dtypes, a
kernel entered so often that dispatch and unboxing dominate, and `nogil` used without threading.

**Detection.** Follow the JIT half of Pass 7 in `detection-passes.md`. A missing `cache=True` is also a style finding,
so report it here with its runtime cost, which is recompilation on every fresh process, and note the overlap rather than
duplicating the style rule.

**Evidence.** STATIC. Quote the decorator verbatim with its `<path>:<line>` and name the function. For a race finding,
name the written variable, cite the line that writes it, and state why its index differs from the parallel loop
variable. For signature churn, cite every call site with the argument dtypes it passes. For boundary cost, cite the call
site, its enclosing loop, the entry count, and the per-entry work.

**Impact.** HIGH for an explicit `forceobj=True` kernel on a hot path. MEDIUM for a missing `cache=True`, for `prange`
on an inner loop, and for signature churn. A `prange` race is a correctness defect that `/audit-correctness` owns as
CONCURRENCY_DEFECT, so carry it here only as a tag recording that the parallelization this kernel enables is invalid.

---

## PYTHON_INTERPRETER_OVERHEAD

**Definition.** Costs imposed by the CPython object model and interpreter dispatch on paths that execute many times.
Covers repeated attribute and global-name lookups inside loops, call overhead for trivial bodies in hot paths, and
exceptions used as ordinary control flow. Also covers string building by repeated concatenation, layers of abstraction
on hot paths, function-local imports executed per call, and objects carrying a `__dict__` where `__slots__` would shrink
them.

**Detection.** Inside each data-scaled loop body, find dotted expressions of depth two or more and flag the ones
invariant across the loop. Flag module-level and builtin name references only when the loop is data-scaled and the name
is read many times per iteration. For each function called from inside a data-scaled loop, read its body and judge
whether call overhead exceeds it. Flag `try` blocks inside loops whose `except` clause is reached on a substantial
fraction of iterations by design. Flag string concatenation accumulating inside a data-scaled loop, which is genuinely
quadratic. For slots findings, confirm the documented exemptions were checked before proposing the change.

**Evidence.** STATIC. Cite the construct, quote the enclosing loop header with its trip count and the source of the
bound, and state the per-iteration repetition count. For slots findings, quote the class definition line, give the
instance count with the line that creates the instances, name the base class checked against the exemptions, and give
the estimated per-instance byte saving.

**Impact.** HIGH only for string concatenation in a data-scaled loop. MEDIUM for slots on classes with large instance
counts, and for exceptions raised on a majority of iterations. LOW for attribute and global lookups, which aggregate
into one note per function.

---

## IO_AND_SERIALIZATION_COST

**Definition.** Input and output structured so that per-call overhead, syscall count, or format choice dominates the
useful work. Covers file operations executed once per record or per element, repeated opening of the same path, and
unbatched reads and writes. Also covers an on-disk format or compression setting that mismatches the access pattern,
serialization round-tripping through Python objects, and I/O interleaved into a compute function.

**Detection.** Follow the DISK half of Pass 7 in `detection-passes.md`. Any I/O call at per-record or per-element
multiplicity is a finding, so state its call count. Group hits by the path expression, and report two or more opens of
the same path within one logical operation. For writes inside a loop, check whether the target API accepts a batch, and
cite the project's logging stack where the writes sit on an acquisition path.

**Evidence.** STATIC for call-count findings. Cite the I/O call, derive its multiplicity from the Pass 1 table with the
enclosing loop quoted, give the resulting total call count, and give the path expression. For repeated-open findings,
cite every line that opens the same path. For I/O-in-compute findings, name the function and cite the lines performing
each role. MEASUREMENT-PENDING is required for every format and compression recommendation.

**Impact.** HIGH for I/O at per-record or per-element multiplicity, and for repeated opens of one path inside a loop.
MEDIUM for unbatched writes and for I/O mixed into compute.

---

## CONCURRENCY_AND_GIL

**Definition.** Parallelism absent where the work is embarrassingly parallel, or present but structured so the GIL,
oversubscription, or data transfer cancels the benefit. Covers serial loops over independent work units, thread pools
wrapped around pure-Python work, process pools whose payload cost exceeds the task's work, nested parallelism causing
oversubscription, and blocking calls in extension code that hold the GIL.

**Detection.** Follow the PROCESS AND THREAD half of Pass 7 in `detection-passes.md`. For a parallelism opportunity,
verify independence mechanically by listing every variable the loop body writes and confirming each write targets either
a loop-local name or a distinct index of a shared output array. Any write to a shared accumulator disqualifies the loop.

**Evidence.** STATIC. For GIL findings, cite the task function and the specific line proving whether the work releases
or holds the GIL. For independence claims, enumerate every variable written in the loop body with each write's line and
its index expression. For oversubscription, give both parallelism factors with their sources.

**Impact.** HIGH for blocking without a GIL release in extension code. MEDIUM for GIL-serialized thread pools and for
oversubscription. Every proposal to add parallelism is MEASUREMENT-PENDING.

---

## CPP_RUNTIME_COST

**Definition.** C++ constructs that cost time, determinism, or binary size, evaluated against the archetype the file
belongs to. In embedded firmware this covers blocking waits in runtime code, heap allocation, RTTI, exceptions, STL heap
containers, `<iostream>`, virtual inheritance, floating-point microsecond timing, and platform-dependent integer widths.
In nanobind extension code it covers by-value parameter copies, blocking calls holding the GIL, and Python-object access
with the GIL released. In both it covers non-`const` members and methods, stateless methods left non-`static`, and
missing `final` or `override` that block devirtualization.

**Detection.** Follow the C++ half of Pass 9 in `detection-passes.md`. Establish the archetype first, because the
constraint sets differ and misapplying them is the primary source of C++ false positives.

**Evidence.** STATIC. State the file's archetype with the evidence establishing it, which is the presence of
`platformio.ini`, the includes, and the base class. Cite the construct and quote it verbatim, name the enclosing method,
which decides the setup and calibration exemption for blocking waits, and quote the style rule with its skill and
reference file. For a missing `override`, cite the base class's virtual method proving an override was intended.

**Impact.** HIGH for blocking waits in embedded runtime code, heap allocation in firmware, missing `override`, and a GIL
held across a blocking extension call. MEDIUM for bare integer types and for floating-point microsecond timing.

---

## CSHARP_RUNTIME_COST

**Definition.** C# constructs that allocate or perform avoidable work on a per-frame path, pressuring the garbage
collector and causing frame-time spikes. Covers LINQ operators inside the per-frame set, `GetComponent<T>()` per frame
instead of a cached reference, and string concatenation and interpolation per frame. Also covers `new
WaitForSeconds(...)` allocated inside a coroutine loop, multiple enumeration of a deferred query, boxing and defensive
copies from non-`readonly` structs, `static readonly` where `const` belongs, and instance methods that never touch
`this`.

**Detection.** Follow the C# half of Pass 9 in `detection-passes.md`, working strictly inside the per-frame reachability
set Pass 1 built. Resolve every receiver's declared type before reporting a named method.

**Evidence.** STATIC. Cite the construct, write out the per-frame call chain that reaches it as `Update() -> Method() ->
line` with a `<path>:<line>` for each hop, and quote the style rule with its skill and reference file. Every LINQ
finding additionally requires the receiver's declared type quoted from its declaration line and confirmation that the
file imports `System.Linq`. For struct findings, give the field list and the computed size.

**Impact.** HIGH for LINQ and `GetComponent<T>()` genuinely inside the per-frame set, and for per-frame string
allocation. MEDIUM for coroutine allocation and for boxing.
