# Week_10-Advanced-Python-Library-with-Comprehensive-Testing
A sophisticated Python library that demonstrates advanced Python concepts including decorators, generators, context managers, and metaclasses. Implement comprehensive testing with high coverage, documentation, and type hints. This project shows professional Python development practices.
# corekit

A small, dependency-free Python library of advanced building blocks: **decorators**, **generators**, **context managers** and **metaclasses**, all fully type-hinted and tested to 97% branch coverage.

[![CI](https://github.com/example/corekit/actions/workflows/ci.yml/badge.svg)](https://github.com/example/corekit/actions/workflows/ci.yml)
![Coverage](https://img.shields.io/badge/coverage-97.6%25-brightgreen)
![Python](https://img.shields.io/badge/python-3.10%20%7C%203.11%20%7C%203.12-blue)
![Typed](https://img.shields.io/badge/typing-strict-blue)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Project overview

`corekit` exists to demonstrate professional Python practice on a codebase small enough to read in one sitting. Every feature is something a real project reaches for: retrying a flaky network call, memoizing an expensive function, streaming a file that doesn't fit in memory, writing a file without risking corruption, and validating data at the boundary.

Design goals:

- **Zero runtime dependencies.** The standard library is enough.
- **Typed end to end.** Strict `mypy`, and a `py.typed` marker so your project gets the hints too.
- **Testable by construction.** Clocks and sleep functions are injected rather than imported, so nothing in the test suite ever sleeps or depends on wall-clock time.
- **Documentation that cannot rot.** Every docstring example is executed by the test run via `--doctest-modules`.

## Installation

```bash
# Clone and install in editable mode
git clone https://github.com/example/corekit.git
cd corekit
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install -e .              # runtime only
pip install -e ".[dev]"       # plus pytest, mypy, ruff, black, hypothesis
pre-commit install            # optional: run the linters on every commit
```

Requires Python 3.10 or newer (the code uses `X | Y` unions and `ParamSpec`).

## Quick start

```python
from corekit import (
    DataPipeline, Field, SafeFileHandler, ValidatedModel,
    cache, get_plugin, positive, retry, sqlite_connection, timer,
)

# 1. Decorators — retry a flaky call, time it, cache the result
@retry(max_attempts=3, delay=1.0, backoff=2.0)
@timer
@cache(ttl=3600)
def fetch(url: str) -> dict[str, str]:
    return {"url": url, "data": "sample"}

# 2. Generators — stream a file larger than RAM
pipeline = (
    DataPipeline()
    .add_processor(get_plugin("strip")())
    .add_filter(lambda record: record["status"] == "active")
)
for record in pipeline.process_large_file("people.csv"):
    print(record)

# 3. Context managers — a crash mid-write leaves the original file intact
with SafeFileHandler("report.txt", "w") as handle:
    handle.write("safe and managed file operation")

with sqlite_connection("app.db") as connection:      # commits, or rolls back
    connection.execute("INSERT INTO users VALUES ('ada')")

# 4. Metaclasses — declarative, validated models
class User(ValidatedModel):
    name: str
    age: int = Field(default=0, validator=positive())
    tags: list[str] = Field(default_factory=list)

User(name="Ada Lovelace", age=36).to_dict()
# {'name': 'Ada Lovelace', 'age': 36, 'tags': []}
```

## The four concepts

### Decorators

| Name | What it does |
|---|---|
| `retry` | Retries on failure with exponential backoff. Only the exception types you name trigger a retry; everything else propagates immediately. On exhaustion it raises `RetryError` with the real cause attached as `__cause__`. The `sleep` function is injectable, so tests never wait. |
| `cache` / `Cache` | Memoizes with a TTL, a bounded size with oldest-first eviction, thread safety via `RLock`, and hit/miss/eviction stats on `func.cache.stats`. Unhashable arguments fall back to `repr` instead of raising. |
| `timer` | Logs elapsed time. Works as `@timer` or `@timer(on_result=...)`, and measures correctly even when the call raises. |
| `validate_call` | Enforces the function's own annotations at runtime, including `*args` and `**kwargs` and the return value. |
| `deprecated` | Emits a `DeprecationWarning` with the right `stacklevel`. |

All wrappers use `functools.wraps`, so `__name__`, `__doc__`, annotations and `__wrapped__` survive decoration and `inspect.signature` still works.

### Generators

`DataPipeline` streams CSV, JSON Lines and plain text one record at a time. Processors are plain callables; returning `None` drops the record. In lenient mode (`strict=False`) a failing processor skips just that record and increments `pipeline.skipped`.

Helpers: `batched`, `sliding_window`, `take`, `fibonacci`, `count_by` — all lazy, all safe on infinite inputs.

### Context managers

| Name | Guarantee |
|---|---|
| `SafeFileHandler` | Writes go to a temp file in the destination directory and are renamed over the target only on clean exit. A crash leaves the original byte-for-byte intact and leaves no temp files behind. |
| `sqlite_connection` | Commits on success, rolls back on exception, always closes. |
| `temporary_config` | Scoped overrides, restoring previous values — including keys that didn't exist — even if the block raises. |
| `timed` | Yields a `Timing` object whose `.elapsed` is filled in on exit. |

### Metaclasses

`PluginMeta` registers every concrete `Plugin` subclass at class-creation time, so there is no `register()` call to forget, and duplicate names raise immediately. `ModelMeta` collects annotated attributes into `__fields__` and resolves string annotations with `get_type_hints`, which is what makes `ValidatedModel` work under `from __future__ import annotations`.

Four plugins ship built in: `strip`, `uppercase`, `timestamp`, `drop-empty`.

## Command line interface

```bash
corekit process data.csv --plugin strip --plugin uppercase --limit 5
corekit stats orders.jsonl --field status
corekit plugins --json
corekit --version
```

Exit codes: `0` success, `1` handled library error, `2` bad usage.

```
$ corekit plugins
drop-empty  Remove records whose values are all empty.
strip       Trim whitespace from all string values.
timestamp   Add an ingested_at UTC timestamp.
uppercase   Uppercase all string values.
```

## Testing

```bash
pytest                                   # full suite + doctests + coverage gate
pytest tests/test_decorators.py -v       # one module
pytest benchmarks/ --no-cov -s           # benchmarks (excluded by default)
mypy src                                 # strict type checking
ruff check . && black --check . && isort --check .
```

Actual output of `pytest`:

```
Name                                   Stmts   Miss Branch BrPart  Cover
------------------------------------------------------------------------
src/corekit/__init__.py                   14      0      0      0 100.0%
src/corekit/cli.py                        66      0     10      0 100.0%
src/corekit/core/context_managers.py      90      0     18      1  99.1%
src/corekit/core/decorators.py           112      0     20      0 100.0%
src/corekit/core/generators.py            93      0     30      0 100.0%
src/corekit/core/metaclasses.py           40      1      4      1  95.5%
src/corekit/exceptions.py                 15      0      0      0 100.0%
src/corekit/plugins.py                    28      0      2      0 100.0%
src/corekit/types.py                      15      0      0      0 100.0%
src/corekit/utils/serializers.py          57      1     28      1  97.6%
src/corekit/utils/validators.py          159      8     70      8  93.0%
------------------------------------------------------------------------
TOTAL                                    693     10    182     11  97.6%
Required test coverage of 90% reached. Total coverage: 97.60%
220 passed
```

The suite covers five layers:

- **Unit tests** for every public function, including error paths and edge cases.
- **Integration tests** for full workflows — CSV to pipeline to model validation to SQLite.
- **Property-based tests** with Hypothesis, asserting invariants over generated inputs (batching never loses or reorders an item; JSON round trips are lossless; a cached function runs exactly once per distinct argument).
- **Doctests**, so all 28 documentation examples are verified on every run.
- **Benchmarks** with `tracemalloc` assertions on memory, not just timing.

Mocking is used where it earns its place: `MagicMock` to assert a failing dependency is called exactly `max_attempts` times, and `patch` to make TTL expiry deterministic without sleeping.

### Testing techniques on display

```python
# Injected sleep — asserts the backoff schedule without waiting 3.5 seconds
def test_applies_exponential_backoff(self, sleeps: list[float]) -> None:
    @retry(max_attempts=4, delay=0.5, backoff=2, sleep=sleeps.append)
    def always_fails() -> None:
        raise ValueError("nope")

    with pytest.raises(RetryError):
        always_fails()
    assert sleeps == [0.5, 1.0, 2.0]


# Property-based — must hold for every list and every batch size
@given(items=st.lists(st.integers()), size=st.integers(min_value=1, max_value=50))
def test_batching_preserves_every_item_in_order(self, items, size) -> None:
    batches = list(batched(items, size))
    assert [item for batch in batches for item in batch] == items
```

## Performance

Measured by `examples/performance_comparison.py` on 100,000 CSV rows:

| Comparison | Result |
|---|---|
| `cache` vs. recomputing | **671x faster** on a hit |
| `cache` vs. `functools.lru_cache` | TTL bookkeeping costs ~2.9 µs per hit |
| Streaming vs. loading a file | 0.05 MB vs. 32.01 MB peak — **685x less memory** |
| `batched` vs. slicing a list | 0.08 MB vs. 38.15 MB peak — **485x less memory**, and faster |

Streaming and eager reading take about the same time; the win is entirely in memory, which is the point — the streaming version handles a file larger than RAM, the eager one cannot.

## Project structure

```
week10-advanced-library/
├── src/corekit/
│   ├── __init__.py              # curated public API
│   ├── cli.py                   # argparse CLI: process / plugins / stats
│   ├── exceptions.py            # CorekitError and its subclasses
│   ├── plugins.py               # four built-in plugins
│   ├── py.typed                 # PEP 561 marker
│   ├── types.py                 # shared aliases, TypeVars, protocols
│   ├── core/
│   │   ├── decorators.py        # retry, cache, timer, validate_call
│   │   ├── generators.py        # DataPipeline and lazy helpers
│   │   ├── context_managers.py  # SafeFileHandler, sqlite_connection, ...
│   │   └── metaclasses.py       # PluginMeta, ModelMeta, SingletonMeta
│   └── utils/
│       ├── validators.py        # check_type, constraints, ValidatedModel
│       └── serializers.py       # to_builtin, dumps, loads, to_model
├── tests/                       # 220 tests: unit, integration, property-based
├── benchmarks/test_benchmarks.py
├── examples/                    # basic, advanced (ETL), performance
├── docs/                        # Sphinx sources
├── .github/workflows/ci.yml
├── pyproject.toml               # packaging + pytest/mypy/ruff/black config
├── CHANGELOG.md
└── LICENSE
```

## Architecture notes

**Dependency direction is strictly one way.** `types` and `exceptions` depend on nothing; `core.metaclasses` depends on those; `utils.validators` depends on `core.metaclasses`; `core.decorators` imports `check_type` lazily inside `validate_call`. There are no import cycles.

**Everything time-related is injected.** `retry` takes a `sleep` callable and `Cache` takes a `clock`. This is why 220 tests run in about five seconds with no sleeps and no flakiness.

**`Cache` is a class, not a closure.** Holding state in an object makes `.stats`, `.clear()` and `len()` natural, and the instance is attached to the wrapper as `func.cache`.

**Atomic writes use rename, not truncate.** `os.replace` is atomic on POSIX and Windows, which is what makes the crash-safety guarantee real rather than aspirational.

## Documentation

```bash
cd docs && make html
open build/html/index.html
```

Sphinx is configured with `autodoc`, `napoleon` (Google-style docstrings) and `doctest`, so `make doctest` re-verifies every example in the API reference.

## What I learned

- **Decorators** — why `functools.wraps` matters, how `ParamSpec` preserves a signature through a decorator so type checkers still see the real parameters, and how decorator order changes behaviour (`@retry` outside `@cache` retries a cold call; the reverse would cache the failure).
- **Generators** — that laziness is a memory strategy, not a style preference: 685x less memory on the same workload.
- **Context managers** — that `__exit__` returning `False` is what stops a manager from silently swallowing exceptions, and that "safe file write" means rename-on-success, not try/finally.
- **Metaclasses** — that `__new__` runs at class-creation time, which is exactly the hook needed for a registry with no boilerplate, and that `ABCMeta` must be the base when mixing with `ABC`.
- **Type hints** — that `from __future__ import annotations` makes annotations strings, so any runtime validation has to call `get_type_hints`. This was a real bug found by the test suite, not a hypothetical.
- **Testing** — that dependency injection is what makes code testable, and that property-based tests catch the empty-list and unicode cases you'd never write by hand.

## License

MIT — see [LICENSE](LICENSE).
