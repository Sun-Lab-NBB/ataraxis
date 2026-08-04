# Anti-patterns and examples

Common anti-patterns to avoid and input/output transformation examples for Python code.

---

## Input/output examples

Transform code to match project style:

| Input (What you wrote)                                        | Output (Correct style)                                         |
|---------------------------------------------------------------|----------------------------------------------------------------|
| `def calc(x):`                                                | `def calculate_value(x: float) -> float:`                      |
| `pos = get_pos()`                                             | `position = get_position()`                                    |
| `np.zeros((4,), np.float32)`                                  | `np.zeros((4,), dtype=np.float32)`                             |
| `# set x to 5`                                                | Remove comment (self-explanatory code)                         |
| `data: NDArray`                                               | `data: NDArray[np.float32]`                                    |
| `"""A class that processes data."""`                          | `"""Processes experimental data."""`                           |
| `"""Whether to enable filtering."""`                          | `"""Determines whether to enable filtering."""`                |
| `raise ValueError("Bad input")`, with ataraxis-base-utilities | `console.error(message="...", error=ValueError)`               |
| `print("Starting...")`                                        | `console.echo(...)` (use `raw=True` for pre-formatted output)  |
| `time.sleep(0.005)`                                           | `timer.delay(delay=5000)` (microseconds)                       |
| `elapsed = time.time() - start`                               | `elapsed = timer.elapsed` (use PrecisionTimer)                 |
| `f"{elapsed:.2f}s"` for display                               | `timer.format_elapsed()` (human-readable)                      |
| Manual `while` + `time.sleep` polling                         | `for cycle in timer.poll(interval=...):`                       |
| `time.time() - start < timeout`                               | `Timeout(duration=...).expired`                                |
| `datetime.now().strftime("%Y-%m-%d")`                         | `get_timestamp(output_format=TimestampFormats.STRING)`         |
| `datetime.strptime(s, fmt)`                                   | `parse_timestamp(date_string=s, format_string=fmt)`            |
| `duration_s = duration_us / 1_000_000`                        | `convert_time(time=duration_us, from_units=..., to_units=...)` |
| `interval = 1_000_000 / hz`                                   | `rate_to_interval(rate=hz)`                                    |
| `timedelta(seconds=us / 1e6)`                                 | `to_timedelta(time=us, from_units=TimeUnits.MICROSECOND)`      |
| `yaml.dump(config.__dict__, file)`                            | `config.to_yaml(file_path=path)` (subclass YamlConfig)         |
| `if flag == True:`                                            | `if flag:` (use truthiness)                                    |
| `if data != None:`                                            | `if data is not None:` (use identity)                          |
| `'single quotes'`                                             | `"double quotes"` (enforced by ruff)                           |
| `"value: %d" % count`                                         | `f"value: {count}"` (f-strings only)                           |
| `raise ValueError(msg)`, with ataraxis-base-utilities         | `console.error(message=msg, error=ValueError)`                 |
| `max(1, os.cpu_count() - 4)`                                  | `resolve_worker_count()`                                       |
| `np.array([v], dtype=dt).view(uint8)`                         | `convert_scalar_to_bytes(value=v, dtype=dt)`                   |
| `np.frombuffer(data, dtype=dt)[0]`                            | `convert_bytes_to_scalar(data=data, dtype=dt)`                 |

---

## Documentation anti-patterns

| Anti-Pattern                         | Problem                     | Solution                                     |
|--------------------------------------|-----------------------------|----------------------------------------------|
| `"""A class that processes data."""` | Noun phrase, not imperative | `"""Processes experimental data."""`         |
| Bullet lists in docstrings           | Breaks prose flow           | Use complete sentences instead               |
| `# Set x to 5` before `x = 5`        | States the obvious          | Remove or explain *why*                      |
| Missing dtype in `NDArray`           | Type checking fails         | Always specify `NDArray[np.float32]`         |
| `Whether to...` for bool params      | Incomplete phrasing         | Use `Determines whether to...`               |
| `# ======` section separators        | Visual clutter              | Use blank lines to separate sections         |
| `:class:` in non-MCP docstrings      | Humans parse prose better   | Use prose, with specifiers in MCP tools only |

---

## Documentation quality anti-patterns

| Anti-Pattern                                | Problem                        | Solution                                            |
|---------------------------------------------|--------------------------------|-----------------------------------------------------|
| Sentences over 40 words in prose            | Hard for humans to parse       | Split at clause boundaries into shorter sentences   |
| Docstring length driven by function length  | Tracks size, not difficulty    | Match length to conceptual difficulty               |
| Docstring explains where it is called       | Usage context goes stale       | Describe what the function does                     |
| Docstring restates the type signature       | Redundant information          | Describe behavior, not types                        |
| Docstring contradicts function behavior     | Defect masquerading as docs    | Update docstring to match implementation            |
| Stale issue numbers in comments             | Misleading after issue closure | Remove or update with current reference             |
| Typos and grammar errors in comments        | Signals unreviewed prose       | Proofread comments and docstrings before submission |
| Comments narrate what code obviously does   | Adds noise, no signal          | Remove or explain why the code does what it does    |
| Extended description on a self-evident body | Padding the reader skips       | Keep the summary line alone unless a fact is earned |
| `# Now also handles the empty case`         | Records the edit, not the code | State current behavior, leave history to the commit |
| Docstring grown on every edit               | Ratchets toward unreadable     | Rewrite in place, delete what the change made moot  |

---

## Naming anti-patterns

| Anti-Pattern        | Problem            | Solution                     |
|---------------------|--------------------|------------------------------|
| `pos`, `idx`, `val` | Abbreviations      | `position`, `index`, `value` |
| `curIdx`            | Missing underscore | `_current_index`             |
| `process()`         | Too generic        | `process_frame_data()`       |
| `data1`, `data2`    | Non-descriptive    | `input_data`, `output_data`  |

---

## Code anti-patterns

| Anti-Pattern                                          | Problem                  | Solution                                       |
|-------------------------------------------------------|--------------------------|------------------------------------------------|
| `np.zeros((4,), np.float32)`                          | Positional dtype arg     | `np.zeros((4,), dtype=np.float32)`             |
| `raise ValueError(...)`, with ataraxis-base-utilities | Wrong error handling     | `console.error(message=..., error=ValueError)` |
| `from typing import Optional`                         | Old-style optional       | Use `Type \| None`                             |
| `@numba.njit` without `cache=True`                    | Recompiles every run     | `@numba.njit(cache=True)`                      |
| Inconsistent f-string prefixes                        | Confusing multi-line     | Use `f` prefix on all lines                    |
| `'single quotes'`                                     | Violates ruff formatting | Use `"double quotes"`                          |
| `"val: %d" % x` or `.format()`                        | Old-style formatting     | Use f-strings exclusively                      |
| `if flag is True:` / `== True`                        | Redundant comparison     | Use truthiness: `if flag:`                     |
| `if len(items) == 0:`                                 | Verbose emptiness check  | Use truthiness: `if not items:`                |
| Deep nesting for validation                           | Hard to read             | Use guard clauses with early returns           |
| Missing `__all__` in `__init__.py`                    | Unclear public API       | Add alphabetically sorted `__all__`            |

---

## Ataraxis library anti-patterns

| Anti-Pattern                          | Problem                           | Solution                                                         |
|---------------------------------------|-----------------------------------|------------------------------------------------------------------|
| `print("message")` for plain text     | No logging, inconsistent          | `console.echo(message="...")` (use `raw=True` for pre-formatted) |
| `time.sleep(0.001)`                   | Low precision, blocks GIL         | `PrecisionTimer.delay(delay=1000)`                               |
| `time.time()` for intervals           | Insufficient precision            | `PrecisionTimer.elapsed`                                         |
| `f"{elapsed:.2f}s"` for display       | Inconsistent, manual formatting   | `PrecisionTimer.format_elapsed()`                                |
| Manual elapsed snapshots in a list    | Verbose, error-prone              | `PrecisionTimer.lap()` / `.laps`                                 |
| `while True: time.sleep()` polling    | Low precision, blocks GIL         | `PrecisionTimer.poll(interval=...)`                              |
| `time.time() - start < timeout` check | Low precision, verbose            | `Timeout(duration=...).expired`                                  |
| `datetime.now().strftime(...)`        | Inconsistent format               | `get_timestamp()`                                                |
| `datetime.strptime()` + epoch math    | Verbose, timezone-unsafe          | `parse_timestamp()`                                              |
| `elapsed_us / 1_000_000`              | Magic number conversion           | `convert_time(time=..., from_units=..., to_units=...)`           |
| `1_000_000 / hz` for intervals        | Magic number, fragile             | `rate_to_interval(rate=hz)`                                      |
| `timedelta(seconds=us / 1e6)`         | Magic number, error-prone         | `to_timedelta(time=us, from_units=...)`                          |
| Manual YAML dump/load                 | No type safety                    | Subclass `YamlConfig`                                            |
| `multiprocessing.Array`               | Limited dtype support             | `SharedMemoryArray`                                              |
| Direct file writes in loops           | Blocks acquisition                | `DataLogger` with `LogPackage`                                   |
| Manual `isinstance()` for list check  | Verbose, error-prone              | `ensure_list()`                                                  |
| Manual slice batching                 | Verbose, doesn't preserve dtype   | `chunk_iterable()`                                               |
| `os.cpu_count() - N` for workers      | No None guard, fragile            | `resolve_worker_count()`                                         |
| `os.cpu_count() // N` for job slots   | No None guard, fragile            | `resolve_parallel_job_capacity()`                                |
| `np.array([v]).view(np.uint8)` manual | Duplicated, no cache, error-prone | `convert_scalar_to_bytes()` / `convert_bytes_to_scalar()`        |
| `np.frombuffer(data, dtype=...)` raw  | No validation, no copy safety     | `convert_bytes_to_array()` / `convert_array_to_bytes()`          |

---

## Comment anti-patterns

| Anti-Pattern                                 | Problem                     | Solution                                 |
|----------------------------------------------|-----------------------------|------------------------------------------|
| `# This function sends...`                   | Not third-person imperative | `# Sends...` (third-person imperative)   |
| `# ========================`                 | Visual clutter              | Remove the separator and use blank lines |
| `# ---- Section ----`                        | Visual clutter              | Remove the separator and use blank lines |
| End-of-line comment on complex logic         | Hard to scan                | Place comment above the code block       |
| `x = 5  # Set x to 5`                        | States the obvious          | Remove or explain *why*                  |
| Adding docstrings to code not written by you | Unnecessary churn           | Only document your changes               |
| Type annotations as comments (`# type: int`) | Redundant                   | Use actual type hints in the signature   |

---

## Cross-language consistency violations

These anti-patterns drift toward C++ or C# conventions that do not apply in Python:

| Wrong (C++/C# drift)                       | Correct (Python convention)                   | Rule                                |
|--------------------------------------------|-----------------------------------------------|-------------------------------------|
| `def SendData(self):`                      | `def send_data(self):`                        | snake_case methods, not PascalCase  |
| `kTimeout = 100`                           | `_TIMEOUT: int = 100`                         | _UPPER_SNAKE constants, not kPrefix |
| `static constexpr` style `TIMEOUT = 100`   | `_TIMEOUT: int = 100` with inline docstring   | Module-level constant with type     |
| `enum class` style `class Status(Enum):`   | `class Status(StrEnum):` or `(IntEnum):`      | Use StrEnum/IntEnum, not bare Enum  |
| `/// Doxygen @brief comment`               | `"""Google-style docstring."""`               | Docstrings, not Doxygen             |
| `// NOLINT` style bare `# type: ignore`    | `# type: ignore[code]` with the error code    | Specific suppression codes          |
| `_camelCase` for private members           | `_snake_case` for private members             | snake_case, not camelCase           |
| `self.publicField` (C# camelCase)          | `self._private_field` or public via property  | Private with `_` prefix             |
