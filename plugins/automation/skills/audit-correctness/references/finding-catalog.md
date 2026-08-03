# Finding catalog

The nineteen finding categories of `/audit-correctness`. Each entry gives the category's definition,
its mechanical detection procedure, the evidence a report entry must carry, and its severity guidance.
Classify every candidate against exactly one category, choosing the most specific one, and list any
others as tags.

Every entry inherits the skill's evidence floor. A finding needs a concrete trigger, written as an
executable expression, a numbered call sequence, or a line-numbered interleaving, and a concrete
result, written as a value, an exception, a corruption, or a hang.

## Contents

- The severity levels
- CONTRACT_VIOLATION
- UNHANDLED_EDGE_CASE
- TYPE_CONTRACT_BREACH
- NUMERIC_DEFECT
- STATE_LIFECYCLE_DEFECT
- RESOURCE_LEAK
- DURABILITY_DEFECT
- CONCURRENCY_DEFECT
- ERROR_HANDLING_DEFECT
- SHARED_MUTABLE_STATE_DEFAULT
- ALIASING_MUTATION
- CONTROL_FLOW_DEFECT
- API_MISUSE
- ORDERING_PROTOCOL_DEFECT
- UNIT_SCALE_CONSTANT_DEFECT
- UNTESTED_PATH_DEFECT
- CPP_LOW_LEVEL_DEFECT
- CSHARP_UNITY_DEFECT
- TEST_ORACLE_GAP

---

## The severity levels

Severity orders the report, and every entry below states which level its category takes by default.

| Severity | Meaning                                                                                             |
|----------|-----------------------------------------------------------------------------------------------------|
| CRITICAL | Silent corruption of persisted or transmitted data, unsafe actuation, a hang, or undefined behavior |
| HIGH     | A wrong value or wrong control flow reaching a consumer, an accumulating leak, a use-after-close    |
| MEDIUM   | A defect confined to messages and logs, or on a path that fails loudly and immediately              |
| LOW      | A defect reachable only from test helpers or debug entry points                                     |

---

## CONTRACT_VIOLATION

**Definition.** The implementation fails to deliver the behavior its own name, signature, annotations,
documentation, or an adjacent enforcement point states it delivers, AND the stated contract is the
intended behavior. The fix edits the CODE. This mirrors `/audit-facts`, which owns the same mismatch
when the documentation is the stale side, and the ownership ladder decides between them.

**Detection.** Run the CEAI procedure from `detection-passes.md` against the callable's contract ledger
row.

**Evidence.** All five parts are mandatory. Quote the contract text verbatim with its `<path>:<line>`
and its source tag. Quote the implementing statements verbatim with their `<path>:<line>`. Give a
named witness written as a callable expression with concrete literal arguments. Give the traced
concrete result of that witness. Give the postcondition it falsifies.

**Severity.** HIGH by default. CRITICAL when the violated postcondition is a data-integrity or safety
guarantee, such as checksum validity, packet framing, a hardware-actuation gate, or a persisted record.

---

## UNHANDLED_EDGE_CASE

**Definition.** A reachable input or state at the edge of the callable's declared-legal domain that the
body mishandles. The contract admits the value, and the body produces a wrong result, an unintended
exception, or a silently degenerate answer.

**Detection.** Run Pass 3 of `detection-passes.md`, instantiating the boundary catalog for each value's
kind and verifying guard dominance.

**Evidence.** Give the exact boundary value or collection literal that triggers it, cite the line it
reaches, name the concrete outcome exactly, such as a specific exception message, a `nan`, a `-1` used
as a valid index, or an infinite loop, and prove reachability by naming a caller that can supply the
value or a boundary the value crosses.

**Severity.** HIGH when the boundary yields a silently wrong value that flows into persisted data or a
hardware command. MEDIUM when it yields an immediate, loud exception.

---

## TYPE_CONTRACT_BREACH

**Definition.** A declared type promises or permits something the body cannot deliver or does not
handle. Covers an optional arm never checked before a dereference, an implicit `None` return on a path
in a function annotated to return a value, a union or `Literal` arm no branch handles, a declared dtype
the body cannot produce, and an annotation contradicted by a runtime value crossing a boundary the type
checker leaves unpoliced.

**Detection.** Run Pass 4 of `detection-passes.md`.

**Evidence.** Quote the annotation with its line, write the path that produces or receives the
unpermitted value as an ordered list of line numbers, give the concrete value that arrives or is
returned, and name the downstream statement that fails on it exactly, with its `<path>:<line>`.

**Severity.** HIGH for an implicit `None` return or an unhandled `None` dereference on a reachable path,
and for a dtype breach reaching persisted data or a wire buffer. MEDIUM for an unhandled union arm that
fails loudly.

---

## NUMERIC_DEFECT

**Definition.** A computation whose result is wrong because of the numeric representation rather than
the algorithm. Covers fixed-width overflow and wraparound, silent truncation on cast or integer
division, float equality and near-equality comparison, precision loss from magnitude or ordering,
division and modulo by zero, modulo and floor division of negatives, and array dtype wraparound that the
same expression on a Python integer would avoid.

**Detection.** Annotate every arithmetic expression with the width of each operand, resolved from the
annotation, the array constructor, the packed struct field, or the declared C++ or C# type. Python
integers are arbitrary precision, so a finding needs at least one fixed-width operand or a value
crossing into a fixed-width sink. Compute the worst-case magnitude of every fixed-width expression from
the operand domains and compare it against the type maximum, giving attention to accumulators inside
loops, shift expressions, running counters, and fixed-width microsecond timestamps.

**Evidence.** Give the operand widths with the lines that declare them, the concrete input magnitude
that crosses the limit, the wrapped, truncated, or imprecise value stated as an actual number, and the
consumer that acts on the wrong value, cited. For a float-equality finding, give the concrete pair of
values that compare unequal.

**Severity.** CRITICAL when the wrong value is a checksum, a length field, a packet header, a hardware
duty cycle, or a persisted sample. HIGH for a silently truncated measurement.

---

## STATE_LIFECYCLE_DEFECT

**Definition.** An object is used outside the window in which it is valid, or is left partially
constructed or partially torn down. Covers use before initialization, use after close, stop, join, or
release, double release, an exception unwinding past half of a multi-step mutation, a missing cleanup on
the error path, and a context manager whose exit never runs.

**Detection.** Build the lifecycle table from the Pass 1 state ledger, classify every public method as
CREATE, USE, or DESTROY, then test every ordered pair the public API permits, as Pass 5 describes. Any
attribute read whose defining line fails to dominate the read is a candidate.

**Evidence.** Give the concrete call sequence as a numbered list of API calls, or the specific raising
callee that unwinds mid-mutation together with the line it raises from. Name the resulting state
exactly, such as a specific missing-attribute error, a log file left open, or a thread left running.

**Severity.** CRITICAL when the partial state leaves a process, a thread, or a hardware actuator
running. HIGH for use-after-close and double release. MEDIUM for a re-initialization that silently
discards state.

---

## RESOURCE_LEAK

**Definition.** An acquired resource is left unreleased on some reachable path. Covers file handles,
sockets, serial ports, camera and device handles, subprocesses, threads, multiprocessing children,
shared-memory blocks, queues, locks, semaphores, memory-mapped arrays, and disposable instances.

**Detection.** Run the acquisition half of Pass 5 in `detection-passes.md`. Verify the release runs on
every path out, giving particular attention to a terminating error call placed between an acquisition
and its release, which is the highest-yield leak site in this framework.

**Evidence.** Give the acquisition line, write the reachable path that skips the release as an ordered
list of line numbers naming the exception, the early return, or the terminating call that takes it, and
state the accumulation consequence concretely, such as one descriptor per failed frame against the
process descriptor limit.

**Severity.** CRITICAL for an unjoined process or thread that keeps the interpreter alive or holds a
device. HIGH for a leak inside a loop or on a retry path. MEDIUM for a single-shot leak.

---

## DURABILITY_DEFECT

**Definition.** A persisted artifact that a crash, a kill, or a second writer can leave partial,
stale, or internally inconsistent, where a later read accepts it as complete. Covers a write straight
to its destination path rather than through a temporary file and a rename, a rename across
filesystems, a completion marker written before the data it vouches for, a checksum computed over the
in-memory buffer rather than the bytes that reached the disk, an unguarded second writer to one path,
and a resume path that treats a partial artifact as finished.

This category owns the state left ON DISK. `RESOURCE_LEAK` owns the handle left open, and
`STATE_LIFECYCLE_DEFECT` owns the object left half-built in memory. A defect that leaves both a leaked
handle and a truncated file is filed here when the persisted bytes are the damaging half.

**Detection.** Run the durability half of Pass 5 in `detection-passes.md`, establishing atomicity,
ordering against the marker, checksum subject, and second writer for every persisted artifact.

**Evidence.** Cite the write with its `<path>:<line>` and quote it. Name the concrete interruption
point as a line number, or the concurrent interleaving as a line-numbered trace. State the resulting
on-disk content exactly, such as a marker naming a session whose descriptor holds zero bytes, or a
checksum that matches a buffer the file never received. Cite the later reader that accepts the
artifact, with its `<path>:<line>`.

**Severity.** CRITICAL when the surviving artifact is acquisition data, a session marker, or a
checksum that a later stage trusts, because the corruption is silent and the audit trail agrees with
it. HIGH when the partial artifact fails a later read loudly. MEDIUM when the only loss is a cache or
a regenerable intermediate.

---

## CONCURRENCY_DEFECT

**Definition.** A defect requiring two execution contexts to manifest. Covers a data race, a non-atomic
read-modify-write, unsynchronized shared mutable state across threads or processes, inconsistent lock
ordering, a callback mutating state the main path reads, and interrupt or signal handler unsafety.

**Detection.** Run Pass 6 of `detection-passes.md`. Enumerate the contexts that actually exist first,
and proceed only when at least two reach the same state.

**Evidence.** Name both contexts with the lines that create them, write the exact interleaving as an
ordered trace with line numbers, state the corrupted result as a value, such as two increments producing
one net increment, and give the observable downstream consequence.

**Severity.** CRITICAL when the race corrupts persisted acquisition data or actuates hardware. HIGH for
a deadlock or a lost update. MEDIUM for a race whose only effect is duplicated or reordered output.

---

## ERROR_HANDLING_DEFECT

**Definition.** The error path itself is wrong. Covers an exception silently swallowed, an over-broad
handler catching more than intended, an error return indistinguishable from a valid success value, an
error message naming the wrong parameter, value, unit, or limit, a documented or computed precondition
never enforced, and a raise happening too late to prevent the damaging side effect.

**Detection.** Classify every handler. For a broad handler, list the exception types actually raisable
inside the protected block and name the ones the handler cannot handle. For a handler body that only
passes, logs, or continues, trace what the caller then does with the un-updated state. For a handler
returning a default that is also a legal success value, find the caller that cannot tell them apart.
Check handler width, because a protected block wider than the statement that raises mislabels an
unrelated failure.

**Evidence.** Quote the handler or guard with its line, name the concrete failure that is swallowed or
mislabeled by exception type and by the operation that raises it, quote the caller line that cannot
distinguish an error return from a success value, and give the resulting wrong behavior. For a message
finding, quote the message and the guard condition side by side.

**Severity.** CRITICAL when the swallowed error lets corrupted or missing data be recorded as valid.
HIGH for a broad handler around a wide block, and for a documented precondition left unenforced.

---

## SHARED_MUTABLE_STATE_DEFAULT

**Definition.** State unintentionally shared across calls, instances, or importers. Covers a mutable
default argument, a class-body mutable used as if it were per-instance, a dataclass field with a
mutable default instead of a default factory, a module-level cache or registry mutated at runtime, and a
memoized function handing every caller the same mutable object.

**Detection.** Run the single-threaded half of Pass 6 in `detection-passes.md`. A mutable default in a
signature is a candidate, and it becomes a defect the moment the body mutates the parameter or stores it
on the instance. A missing default factory on its own is a style finding, so the demonstrated
cross-instance aliasing is the finding here.

**Evidence.** Give the shared object's defining line, the mutating line, and a two-call or two-instance
trace showing the second call or instance observing the first's mutation, with the concrete wrong value.

**Severity.** HIGH when the shared object accumulates across sessions or instances and reaches persisted
data. MEDIUM when it affects only a cached lookup recomputed elsewhere.

---

## ALIASING_MUTATION

**Definition.** A callable mutates an object it does not own, or hands out a live reference to internal
state the caller can then mutate. Covers an in-place array operation on a caller's array, an in-place
sort where a copying sort was meant, a returned internal container without a copy, a stored reference to
a caller-supplied mutable, a shallow copy where a deep copy is required, and an array view treated as an
independent array.

**Detection.** For every mutable parameter, search the body for a mutation of it. Where a mutation
exists, check whether the contract declares it, because a declared mutation IS the contract and an
undeclared one is the defect. For every return of internal mutable state, ask whether the class relies
on that state staying unchanged afterwards.

**Evidence.** Name the caller-owned object, cite the mutating line, give a two-statement trace at a real
call site showing the caller's object before and after, and cite the caller line that subsequently reads
the mutated object and gets the wrong answer.

**Severity.** HIGH when the caller's object is raw acquisition data, a configuration object reused across
iterations, or a buffer shared with another process. MEDIUM when the aliased object is local.

---

## CONTROL_FLOW_DEFECT

**Definition.** The shape of the control flow is wrong. Covers an unreachable branch, a condition that is
always true or always false, a missing final arm that lets a value pass through unchanged, a loop that
cannot terminate or that terminates one iteration early or late, an early exit that skips required work,
a break bound to the wrong loop, duplicated or shadowed conditions in a chain, and a cleanup block that
swallows a propagating exception.

**Detection.** For every conditional chain, check for a later condition subsumed by an earlier one, for a
duplicated condition, and for a chain over an enum whose arms number fewer than its members with no final
arm. Evaluate every condition against the declared domain of its operands, giving attention to a check
against a value the type cannot hold, an identity comparison on a computed value, an operator precedence
mistake, and truthiness applied to an array.

**Evidence.** Quote the condition or loop with its line, give the concrete operand values that make the
branch unreachable or always taken, or the concrete input that never terminates, and give the observable
result, naming the work that is skipped, the value that passes through unmodified, or the frozen loop
variable.

**Severity.** CRITICAL for a non-terminating loop on an acquisition or shutdown path. HIGH for an early
return that skips a release or a state reset. MEDIUM for an unreachable arm.

---

## API_MISUSE

**Definition.** A call that violates its callee's own contract. Covers a wrong argument order, a
positional argument bound to the wrong parameter, a misread return value, an ignored return value
carrying a result or an error status, a flag passed with inverted meaning, a required keyword omitted so
a wrong default applies, and a documented usage constraint ignored.

**Detection.** Run Pass 7 of `detection-passes.md`. The style skills mandate keyword arguments precisely
because positional binding silently follows a signature change, so every positional call is a candidate.
It becomes a finding once you show the arguments are transposed or the applied default is wrong for that
site.

**Evidence.** Cite the call site, quote the callee's signature or documented constraint verbatim with its
`<path>:<line>`, which for an installed library is its path inside the environment, give the concrete
argument values that bind wrongly or the return value that is discarded or misread, and give the
resulting wrong behavior at the call site.

**Severity.** CRITICAL for a transposed argument reaching hardware or a persisted schema. HIGH for a
discarded error status and for a violated thread-safety or ordering constraint.

---

## ORDERING_PROTOCOL_DEFECT

**Definition.** Operations performed out of the sequence a protocol, a state machine, or a lifecycle
requires. Covers a state machine permitting an illegal transition, a handshake step skipped or reordered,
a packet assembled with fields in the wrong order or without its framing or checksum, a staged command
advancing without completing its stage, an initialization performed after its first use, and a shutdown
releasing in the wrong order.

**Detection.** Run the sequence half of Pass 7 in `detection-passes.md`. Recover the required order from
an in-repo authority and write it down BEFORE reading the implementation, then build the actual order and
diff the two. Check three specific holes: a transition with no guard, so any state can enter it, a state
with no exit, so the recovery path cannot leave it, and a silently returning default arm.

**Evidence.** Quote the required order with its source line, give the actual order as an ordered list of
implementation lines, give the specific illegal sequence a caller or a peer can produce, and give the
concrete consequence, such as a packet the receiver rejects or an actuator driven before its calibration
loads.

**Severity.** CRITICAL for a protocol, checksum, or framing defect that silently corrupts a wire message
or a persisted record. HIGH for an illegal state transition reachable from the public API.

---

## UNIT_SCALE_CONSTANT_DEFECT

**Definition.** A quantity carried in the wrong unit or scale, or a constant inconsistent with the same
constant elsewhere. Covers milliseconds passed where microseconds are expected, a scale factor applied
twice or omitted, a percentage treated as a fraction, a sample index treated as a time, a rate treated as
an interval, a hard-coded literal duplicating a named constant with a different value, and a value
converted at one boundary but not at its sibling.

**Detection.** Run the unit-ledger half of Pass 7 in `detection-passes.md`. Record the unit each variable
carries from its name suffix, its documentation, and the constant it derives from, writing down every
inherited unit explicitly. Verify each conversion's direction by substituting one concrete value, and
search for a quantity converted at more than one point on the same path.

**Evidence.** Quote both sides of the mismatch with their lines and their declared units, give one worked
substitution with a concrete number showing the wrong magnitude, and cite the consumer that acts on it.
For a constant-consistency finding, quote both literals with their lines and give the reason they must
agree.

**Severity.** CRITICAL when the wrong scale reaches hardware actuation or persisted timestamps. HIGH for
any thousand-fold error on the acquisition path. MEDIUM for a display or log unit error.

---

## UNTESTED_PATH_DEFECT

**Definition.** A defect from any other category living specifically on a line, branch, or module the
project's coverage machinery leaves unexercised. The category exists so those regions get named
explicitly rather than blending into the rest of the report.

**Detection.** Run Pass 9 of `detection-passes.md`, working the tiers in order.

**Evidence.** The underlying category's full evidence, meaning trigger, path, and wrong result, PLUS the
coverage citation that ranked it: the tier, the artifact it came from with a path, and for a T3 finding
the specific branch outcome named.

**Severity.** Inherit the underlying category's severity, then raise it one step for a T0 or T1 region,
because nothing else in the pipeline will catch a defect there. Never lower a severity on coverage
grounds.

---

## CPP_LOW_LEVEL_DEFECT

**Definition.** C++ and embedded defects with no Python analogue. Covers undefined behavior, meaning
signed overflow, out-of-bounds indexing, strict-aliasing violations, uninitialized reads, and misaligned
access, plus integer promotion surprises, object-lifetime and dangling-reference errors, interrupt and
`volatile` misuse, packed-struct and endianness assumptions, and static initialization order.

**Detection.** Run the C++ half of Pass 8 in `detection-passes.md`. Determine the archetype first,
because the embedded prohibitions bind embedded firmware alone. Walk every expression mixing widths and
every comparison between a signed and an unsigned operand, since the signed side converts.

**Evidence.** Give the declarations with their widths and lines, the concrete operand values that trigger
the promotion, overflow, truncation, or undefined behavior, with the actual resulting number, the target
board or archetype whenever the outcome is platform-dependent, named explicitly, and the observable
consequence at a boundary.

**Severity.** CRITICAL for undefined behavior, an interrupt race on an actuation timer, and a
packed-struct layout mismatch corrupting every packet. HIGH for promotion and rollover defects.

---

## CSHARP_UNITY_DEFECT

**Definition.** C# and Unity defects with no Python analogue. Covers a null reference on a reachable path,
Unity's overloaded null where a destroyed object compares equal to null while its reference stays live,
lifecycle-order assumptions across the component callbacks, an event subscription without its matching
unsubscription, a coroutine outliving its component, and access to a destroyed object.

**Detection.** Run the C# half of Pass 8 in `detection-passes.md`, building the lifecycle table and
checking event symmetry.

**Evidence.** Cite the lifecycle method or subscription with its line and show the absence of its
counterpart by citing the paired method, give the concrete runtime sequence that triggers it, such as a
scene reload or a disable and enable cycle stated with a count, and name the observable failure exactly.

**Severity.** CRITICAL when a destroyed-object access or a missed unsubscription silences an
experiment-critical channel or double-actuates a reward. HIGH for a null dereference on a reachable path.

---

## TEST_ORACLE_GAP

**Definition.** A behavior the test suite EXECUTES without PINNING. The line is covered, so the coverage
gate is satisfied, yet no assertion would fail if the behavior changed. This is a finding about a hole in
the safety net, tied to a specific plausible wrong behavior that would ship undetected.

**Detection.** Run Pass 10 of `detection-passes.md`. This is read-only reasoning over the assertion text,
and editing source or tests to try a hypothesis is prohibited.

**Evidence.** Give the target statement with its line, state the exact behavior mutation as a one-line
edit, enumerate the tests that touch the symbol with each cited by `<path>:<line>` and the reason each
still passes under that mutation, and give the concrete wrong value that would then reach a consumer. A
finding saying only that more tests are needed is unreportable.

**Severity.** MEDIUM by default, since this is a latent exposure rather than an active defect. HIGH when
the unpinned behavior is a serialization width, a unit conversion, a checksum, or a safety gate.
