# v0.5 — "Complete Python"

> Target: Q1 2026
> Rust: edition 2024, MSRV 1.85
> Python: 3.11 – 3.14 (requires-python ≥ 3.11)

---

## Goal

Ship `pip install nanobook` with the **full** Rust API exposed to Python,
automated CI on all platforms, and IDE support via type stubs.

---

## WP1 — Complete Python Bindings

Expose every public Rust method to Python. Currently ~60% covered.

### New Python classes

| Class | Wraps | Fields / methods |
|-------|-------|------------------|
| `Order` | `order::Order` | `id`, `side`, `price`, `remaining_quantity`, `filled_quantity`, `status`, `time_in_force` |
| `Position` | `portfolio::Position` | `symbol`, `quantity`, `avg_entry_price`, `total_cost`, `realized_pnl`, `unrealized_pnl(price)` |
| `Event` | `event::Event` | `kind` (str), `__repr__`, round-trip serialization |

### New methods on existing classes

| Class | Method | Returns |
|-------|--------|---------|
| `Exchange` | `get_order(id)` | `Order \| None` |
| `Exchange` | `get_stop_order(id)` | dict \| `None` |
| `Exchange` | `full_book()` | `BookSnapshot` (L3) |
| `Exchange` | `events()` | `list[Event]` |
| `Exchange` | `replay(events)` | `Exchange` (class method) |
| `BookSnapshot` | `imbalance()` | `float \| None` |
| `BookSnapshot` | `weighted_mid()` | `float \| None` |
| `BookSnapshot` | `mid_price()` | `int \| None` |
| `BookSnapshot` | `spread()` | `int \| None` |
| `Portfolio` | `position(symbol)` | `Position \| None` |
| `Portfolio` | `positions()` | `dict[str, Position]` |
| `Portfolio` | `rebalance_lob(weights, exchanges)` | `None` |
| `Portfolio` | `save_json(path)` / `load_json(path)` | persistence |
| `Portfolio` | `snapshot(prices)` | dict with equity, cash, unrealized |
| `MultiExchange` | `best_prices()` | `list[tuple[str, int|None, int|None]]` |

### MultiExchange clone problem

**Critical design issue:** `PyMultiExchange.get_or_create()` currently returns a
*copy* of the inner Exchange. Mutations don't flow back. Options:

1. **Context manager** — `with multi.exchange("AAPL") as ex:` borrows mutably,
   writes back on `__exit__`. Safe, Pythonic, but verbose.
2. **Proxy object** — `PyExchangeRef` holds `Py<PyMultiExchange>` + symbol key,
   delegates all calls to the inner exchange. Transparent, but complex.
3. **Method forwarding** — Add `multi.submit_limit(symbol, ...)` etc. directly
   on `PyMultiExchange`. Flat API, no borrowing issues.

Decision: **option 3** (method forwarding) as primary API, with option 1 as
convenience. Matches how `MultiExchange` is used in Rust (call methods on it,
not extract inner exchanges).

### Files

- Update: `python/src/{exchange,portfolio,multi,types,results}.rs`
- New: `python/src/event.rs`, `python/src/order.rs`, `python/src/position.rs`
- Update: `python/src/lib.rs` (register new classes)

### Tests

- Target: **80+ Python tests** (up from 39)
- Every new method gets at least one test
- Cover: event round-trip, position lifecycle, book analytics, L3 snapshot,
  persistence, MultiExchange forwarding

### Acceptance

- `dir(nanobook)` shows all types and functions
- No Rust pub method without a Python wrapper (audit script)
- MultiExchange mutations work correctly from Python

---

## WP2 — Python Strategy Callback

Allow Python users to define custom strategies without Rust.

```python
def momentum(bar_index, prices, portfolio):
    """prices: dict[str, int], returns list of (symbol, weight) tuples."""
    return [("AAPL", 0.6), ("GOOG", 0.4)]

results = nanobook.run_backtest(
    strategy=momentum,
    price_series=prices,          # list[dict[str, int]]
    initial_cash=1_000_000_00,
    cost_model=nanobook.CostModel(10, 5, 100),
    periods_per_year=12.0,
)
print(f"Sharpe: {results.metrics.sharpe:.2f}")
```

### Implementation

- New struct `PyStrategy` wrapping `Py<PyAny>` (the callable)
- Implements Rust `Strategy` trait via GIL acquire → call → extract weights
- GIL held only during callback; released for portfolio math + LOB matching
- Note: PyO3 `allow_threads` API may change in 0.28+ — pin PyO3 version

### Files

- New: `python/src/strategy.rs`
- Update: `python/src/lib.rs`

### Tests

- Momentum strategy matches Rust EqualWeight on uniform data
- Strategy that raises exception → clean Python error with traceback
- Strategy returning invalid weights → ValueError

### Acceptance

- `run_backtest` works with arbitrary Python callables
- Python exceptions propagate with traceback (not Rust panics)

---

## WP3 — Platform & Release

Consolidates toolchain modernization, packaging metadata, CI, wheels, and
type stubs into one release-engineering track.

### 3a. Toolchain Modernization

**Rust:**

| Change | From | To | Why |
|--------|------|----|-----|
| Edition | 2021 | 2024 | `unsafe_op_in_unsafe_fn` default, improved captures |
| MSRV | 1.70 | 1.85 | Edition 2024 floor; cargo resolver benefits |

**Python:**

| Change | From | To | Why |
|--------|------|----|-----|
| `requires-python` | ≥ 3.9 | ≥ 3.11 | 3.9 EOL, 3.10 EOL Oct 2026, 3.11 safe through Oct 2027 |
| Free-threading | — | tested on 3.14t | Experimental — document status, don't gate release on it |

**Files:** `Cargo.toml`, `python/pyproject.toml`, `python/Cargo.toml`

### 3b. Packaging Metadata

Complete `pyproject.toml` for a professional PyPI listing:

```toml
[project]
name = "nanobook"
version = "0.5.0"
requires-python = ">=3.11"
description = "Deterministic limit order book, portfolio simulator, and matching engine"
license = "MIT"
authors = [{ name = "Ricardo Frantz" }]
readme = "README.md"
keywords = ["orderbook", "trading", "matching-engine", "backtesting", "finance"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Financial and Insurance Industry",
    "Intended Audience :: Science/Research",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: Implementation :: CPython",
    "Programming Language :: Rust",
    "Topic :: Office/Business :: Financial :: Investment",
]

[project.urls]
Homepage = "https://github.com/ricardofrantz/nanobook"
Repository = "https://github.com/ricardofrantz/nanobook"
Documentation = "https://docs.rs/nanobook"
Issues = "https://github.com/ricardofrantz/nanobook/issues"
```

### 3c. CI: Python Tests + Windows

Add to `ci.yml`:

```yaml
python-test:
  name: Python (${{ matrix.os }}, ${{ matrix.python }})
  runs-on: ${{ matrix.os }}
  strategy:
    matrix:
      os: [ubuntu-latest, macos-latest, windows-latest]
      python: ["3.11", "3.12", "3.13", "3.14"]
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ matrix.python }}
    - uses: dtolnay/rust-toolchain@stable
    - run: pip install maturin pytest
    - run: cd python && maturin develop --release
    - run: pytest python/tests/
```

Also add `windows-latest` to Rust test matrix.

### 3d. PyPI Wheels

New workflow: `.github/workflows/wheels.yml`

| Platform | Python | Architecture |
|----------|--------|-------------|
| manylinux_2_28 | 3.11, 3.12, 3.13, 3.14 | x86_64, aarch64 |
| macOS | 3.11, 3.12, 3.13, 3.14 | universal2 |
| Windows | 3.11, 3.12, 3.13, 3.14 | x86_64 |

Uses `PyO3/maturin-action@v1`, PyPI trusted publishing.
Triggers on `v*` tags + manual dispatch.

### 3e. Type Stubs (.pyi)

Hand-written `nanobook.pyi` covering all public types from WP1 + WP2.

- `pyright --verifytypes nanobook` score ≥ 95%
- CI step: compare stub signatures vs actual module (catch drift)

**Files:** `python/nanobook.pyi` (new), include in wheel via pyproject.toml

### 3f. Benchmark Baseline Capture

Capture Criterion baselines for v0.5 release so v0.6 performance work
has a clean comparison point.

- Run `cargo bench` with `--save-baseline v0.5` on release
- Store baseline artifact in CI
- Document baseline procedure in RELEASING.md

**Files:** `.github/workflows/ci.yml` (update bench job),
`RELEASING.md` (update)

### Acceptance (all of WP3)

- `cargo +stable build` clean on edition 2024
- `maturin develop` works on Python 3.11, 3.12, 3.13, 3.14
- Free-threaded 3.14t build documented (experimental status)
- Python tests run in CI on 3 OSes × 4 Python versions
- `pip install nanobook` works on Linux, macOS, Windows
- PyPI page shows description, links, classifiers
- Type stubs provide IDE autocompletion
- Benchmark baseline saved for v0.6 comparison

---

## Execution Order

```
WP3a (toolchain) ──→ WP1 (bindings) ──→ WP3c (CI) ──→ WP3d (wheels)
WP3b (metadata)  ─┘                  ↘            ↗
                                      WP2 (strategy)
                                                ↓
                                           WP3e (stubs)
                                                ↓
                                           WP3f (baseline)
```

1. **WP3a + WP3b** first (quick, unblock everything)
2. **WP1** is the bulk of the work
3. **WP3c** immediately after WP1 (catch regressions)
4. **WP2** in parallel with WP3c
5. **WP3d** after CI is green
6. **WP3e** after API is final
7. **WP3f** at release time

---

## Metrics

| Metric | v0.4 | v0.5 target |
|--------|------|-------------|
| Rust edition | 2021 | 2024 |
| Rust MSRV | 1.70 | 1.85 |
| Min Python | 3.9 | 3.11 |
| Python API coverage | ~60% | 100% |
| Python tests | 39 | 80+ |
| Python in CI | no | yes (3 OS × 4 Python) |
| Windows CI | no | yes |
| PyPI published | no | yes |
| Type stubs | no | yes |
| Benchmark baseline | no | yes |
