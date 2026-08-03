# False-positive guards

Every candidate finding of `/audit-correctness` passes through these guards in order before it enters
the report. The trigger requirement runs first and removes the most candidates. Record the count of
discarded candidates for the coverage ledger.

---

## Guard 1: No trigger, no finding

Every finding names a concrete input, state, or interleaving AND the concrete wrong result it produces.
When the trigger resists being written as an executable expression, a numbered call sequence, or a
line-numbered interleaving, the candidate is deleted.

"This is fragile", "this is not robust", "this could break if someone passed X" where nothing in the
repository can pass X, and "consider adding a check" are all unreportable. This guard removes more
candidates than the rest combined, so apply it first and without mercy.

---

## Guard 2: Private helpers whose callers enforce their preconditions

Before reporting a missing check inside a module-private callable, enumerate EVERY call site. This stays
tractable here because the framework forbids reaching across module boundaries into private names, and a
symbol used outside its defining module must be public. Tests are the sole exception.

When every caller establishes the precondition on every path, the missing check is a design preference
rather than a defect. When a call site exists that fails to enforce it, report THAT path, filed at that
caller, citing its line.

---

## Guard 3: Already pinned by an existing test

Search the suite for an assertion that already fails on your trigger before reporting. A passing
assertion covering it means your reading of the code is wrong, so re-derive before writing anything.

One exception survives. When a test asserts the buggy behavior itself, the finding stays reportable as a
CONTRACT_VIOLATION naming that test as codifying the wrong side, and its suggested fix must state that
correcting the code requires updating that test.

---

## Guard 4: Type-system-impossible inputs on internal paths

Each language polices its own internal paths. Python runs a strict type checker as a lint gate, C++
rejects a mismatched type at compile time, and C# does the same. An internal callable therefore cannot
receive a type its declaration forbids, and questions of the form "what if a string arrives where an
integer is declared" stay out of the report for purely internal paths.

The reportable exception is any boundary the type system leaves unpoliced. In Python that is YAML and
JSON deserialization, CLI arguments, wire bytes, environment variables, `**kwargs`, `Any`, every
`cast(...)`, and every line carrying a type-ignore comment. In C++ it is every C-style cast,
`reinterpret_cast`, `void*` round-trip, and byte buffer parsed into a struct. In C# it is every `as`
cast, unboxing cast, `dynamic` value, null-forgiving `!` operator, and deserialized payload.

---

## Guard 5: Defensive-programming wishes

A missing None check, bounds check, or exception handler is a finding only when a reachable path actually
supplies the bad value. Absent that path the suggestion is a design preference, and this audit makes no
design recommendations. The same applies to "this should validate its input" on a callable whose input is
already validated upstream.

---

## Guard 6: The documentation is the wrong side

Run the ownership ladder before every contract-versus-behavior finding.

When the implementation is self-consistent, every caller already matches the implementation, and no
observable failure exists for an input the documentation declares supported, the documentation is merely
stale. That is a rung 4 result, it belongs to `/audit-facts`, and it must not appear in this report. Only
genuinely undecidable rung 5 cases are reported here, at MEDIUM confidence, marked AMBIGUOUS with both
candidate fixes stated.

---

## Guard 7: Style and convention findings belong to /audit-style

A bare array annotation, a positional `dtype` argument, a missing frozen or slots setting, a positional
call where a keyword was required, a hand-rolled conversion where a library helper exists, a deeply
nested validation block, a function-local import, a bare suppression comment, an IDE suppression comment,
and a hand-edited stub file are every one of them style findings.

Each becomes a finding HERE only once you name the input that produces a wrong VALUE through it. Report
the wrong value as the finding, cite the style rule as supporting context, and never report the style
violation on its own.

---

## Guard 8: Performance findings belong to /audit-performance

Slow, allocating, quadratic, an unnecessary copy, a sleep in a hot loop, a per-frame component lookup,
and query operators in a per-frame method are all cost and speed.

The single exception is a timing property that forms part of a CONTRACT, meaning a documented deadline, a
watchdog interval, a real-time acquisition rate, or a timeout the code itself computes. A missed deadline
there is a correctness finding, and its evidence must state the deadline, cite the line declaring it, and
give the derived or measured overrun.

---

## Guard 9: Console enable and disable calls are correct

`console.enable()` and `console.disable()` calls are correct at every library tier, including component
and dependency libraries. Mention such a call to the user for awareness at most, and never report it as a
defect or propose removing it.

---

## Guard 10: The sanctioned error-reporting patterns are correct

The framework's terminating error call is typed `NoReturn` and always raises, so code after it is
unreachable by design. In a function with a non-`None` return annotation, ruff RET503 REQUIRES a trailing
return carrying a coverage-exclusion pragma and a reason comment, which is compliance rather than dead
code.

Equally, a project without the shared base utilities dependency correctly uses a plain raise. Neither
pattern is an error-handling defect.

---

## Guard 11: Sanctioned coverage exclusions are correct

A module in the coverage omit list, a pragma on an unreachable guard or a platform-specific branch, and
the standard exclude corpus are all deliberate. The exclusion itself is NEVER the finding, so report only
a concrete defect found inside the excluded region, carrying its own trigger.

One narrow exception is reportable. A module in the omit list that matches none of the qualifying kinds,
which are command modules, the MCP server module with its tool modules, and process entry points, is real
logic made invisible to the coverage gate, and that is worth reporting.

---

## Guard 12: Speculative concurrency

A race requires two execution contexts that ACTUALLY exist in this project and that both reach the same
state. A single-threaded project, or a second context that is hypothetical future use, yields no finding.

Check what the synchronization primitive already in use guarantees before reporting, because a shared
array wrapper, a value wrapper, a queue, or a lock may already provide the atomicity you are about to
claim is missing.

---

## Guard 13: Overflow requires a fixed-width domain

Report overflow, wraparound, or truncation only where a fixed-width domain is involved: an array dtype,
a struct format, a packed wire field, a C++ fixed-width type, or a C# numeric type.

A Python integer is arbitrary precision, so a Python accumulator that grows without bound is a memory
topic and belongs to `/audit-performance` if anywhere. C++ and C# integers are fixed width throughout,
so their arithmetic is always inside this guard's domain and never exempt from it.

---

## Guard 14: Intentional exact float comparison

Equality against a value that was ASSIGNED rather than computed, such as a sentinel, a literal held in a
constant, or a value round-tripped through the identical path, is exact and correct.

The finding requires arithmetic to intervene between the assignment and the comparison, and the evidence
must name the two values that differ in their last bits.

---

## Guard 15: Declared mutation is the contract

When the docstring, the parameter tag, an output keyword, or the callable's own name states that it
mutates its argument or returns a live view, the mutation IS the contract.

Report it only when a specific caller relies on the opposite, and then file the finding at that caller
with its line cited.

---

## Guard 16: Generated and vendored code is out of scope

Stub files and the typing marker are generated by the stubs environment and never hand-authored, so a
stub-versus-source divergence is regenerated rather than fixed and is no correctness finding.

Audit nothing inside a virtual environment, site-packages, a tox working directory, a build directory, or
a vendored third-party tree. Read them as authority for a callee's contract, and report findings only
against this repository's own source.

---

## Guard 17: Archetype-inappropriate C++ findings

The embedded prohibitions, meaning no exceptions, no dynamic allocation, no RTTI, and no STL heap
containers, bind embedded firmware files alone. A nanobind Python extension may use every one of them,
and nanobind translates exceptions into Python exceptions by design.

Determine the archetype from the build files before reporting any of these. Equally, never report a
reduced Python or docs-only project for lacking coverage machinery it is defined to omit.

---

## Guard 18: One root cause, one finding

A single defect satisfying several category definitions is reported ONCE, under the most specific
category, with the others listed as tags. Re-reporting the same line from four different passes inflates
the count without adding information.

When the identical defect repeats across several sites, collapse it into one finding carrying the site
list and a count, in the same style the sibling audits use for repeated violations.

---

## Guard 19: An unexercised path is a place to look

A branch the coverage data shows unexercised is a place to LOOK rather than a finding. It becomes a
finding only when the ordinary category procedures produce a concrete trigger and a concrete result
inside it.

Never report a tier, a percentage, or a missing-line list as though it were a defect, and never raise a
severity merely because a region is uncovered. Only T0 and T1 regions raise severity, and only for a
defect already established.

---

## Guard 20: The absence of a test is not a defect

Missing coverage is a project-management observation, and the coverage gate already enforces it
mechanically.

TEST_ORACLE_GAP is reportable only with a named mutation, the enumerated tests that would still pass
under it, and the concrete wrong value that would then ship.

---

## Guard 21: Markers and commented-out code are not defects

TODO markers, FIXME markers, and commented-out code may be style findings. They enter this report only
when the surrounding LIVE code holds a demonstrable defect carrying its own trigger.

---

## Guard 22: A callee that leaves a documented precondition undefended

When the docstring states that an array must be non-empty and no caller passes an empty array, there is
nothing to report.

When a caller does pass one, file the finding at that caller. File it at the callee only when the
callee's contract also promises to raise on that input and fails to.
