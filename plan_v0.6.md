# v0.6 — "Performance & Depth"

> Target: Q2–Q3 2026
> Prerequisite: v0.5 shipped (complete Python bindings, wheels on PyPI)
> Rust: edition 2024, MSRV 1.85+
> Python: ≥ 3.11, tested through 3.14

---

## Goal

Optimize the hot paths (data transfer, cancel), add real market data ingestion,
and harden benchmarking infrastructure. v0.5 made Python complete; v0.6 makes
it fast and production-grade.

---

## WP1 — Benchmark Infrastructure

**Do this first** — v0.5 captured baselines; now expand coverage and add
regression detection so WP2 and WP3 improvements are measured.

### New benchmarks

| Benchmark | What it measures |
|-----------|-----------------|
| `stop_trigger` | Stop order trigger + cascade (depth 1, 10, 50) |
| `trailing_update` | Trailing stop watermark update (Fixed, Percentage, ATR) |
| `multi_symbol` | MultiExchange with 10, 100, 1000 symbols |
| `modify` | Cancel-replace throughput |
| `event_apply` | Single event apply (not batch replay) |
| `portfolio_lob_rebalance` | `rebalance_lob` vs `rebalance_simple` |
| `persistence_roundtrip` | save_events + load_events (100, 1K, 10K events) |

### CI regression detection

- PR job: `cargo bench -- --baseline v0.5`
- Post comment with regression summary (use `criterion-compare-action`)
- Warning threshold: > 5% regression on any benchmark
- Note: shared runners can be noisy — use relative comparisons within the
  same run (A/B), not absolute timings

### Files

- Update: `benches/throughput.rs` (new groups)
- New: `benches/stops.rs`
- Update: `.github/workflows/ci.yml` (benchmark comparison step)

### Acceptance

- All operations benchmarked (20+ benchmarks, up from 12)
- CI warns on > 5% regression
- Benchmark results published as PR comment

---

## WP2 — O(1) Cancel

### Problem

Cancel is O(N) in orders at that price level — scans `VecDeque` to find the
order. With deep levels (1000+ orders at one price), cancel takes ~60 μs vs
~660 ns at shallow levels. Current benchmarks show 1.5M cancel/sec.

### Solution: Tombstone Approach

1. `HashMap<OrderId, (Price, Side, usize)>` stores position index
2. Cancel marks order as tombstone (zero quantity) — O(1)
3. Matching skips tombstones during iteration
4. Periodic compaction removes tombstones when ratio exceeds threshold

### Why tombstones over intrusive linked list

| Approach | Cancel | Match | Memory | Complexity |
|----------|--------|-------|--------|------------|
| Current (VecDeque scan) | O(N) | O(1) | Compact | Simple |
| Tombstone | O(1) | O(1) amortized | +8 bytes/order | Medium |
| Intrusive linked list | O(1) | O(1) | +16 bytes/order | High (unsafe) |

Tombstones are simpler, cache-friendly (still array), and avoid unsafe code.

### Files

- Update: `src/level.rs` (tombstone cancel, skip in matching)
- Update: `src/price_levels.rs` (compaction trigger)
- Update: `src/book.rs` (compaction on remove_level)
- Update: `src/exchange.rs` (cancel_internal uses tombstone path)

### Tests

- Existing cancel tests pass unchanged
- New: cancel 1000 orders at same price level, verify O(1) timing
- Proptest: random insert/cancel/match sequences produce same trades
- Benchmark: cancel at depth 10, 100, 1000 — all ~same latency

### Acceptance

- Cancel benchmark: < 100 ns at any depth (currently 660 ns at depth 10)
- No behavior change — determinism preserved
- Compaction tested: book stays clean after heavy cancel churn
- v0.5 baseline comparison shows improvement

---

## WP3 — Arrow Zero-Copy Price Data

### Problem

`py_sweep_equal_weight` copies all price data from Python → Rust via PyO3
list extraction. For realistic sweeps (1000 params × 252 bars × 20 symbols),
this allocates ~40 MB of intermediate `Vec`s per call.

### Pre-condition

**Profile first.** Before implementing, benchmark v0.5 sweep to confirm data
transfer is actually the bottleneck (not the backtest computation itself).
If data transfer < 10% of total time, defer this WP.

### Solution

Accept Arrow-compatible arrays via PyCapsule (Arrow C Data Interface):

```python
import polars as pl

prices = pl.DataFrame({
    "AAPL": [150_00, 151_00, ...],
    "GOOG": [280_00, 282_00, ...],
})

# Zero-copy: Polars → Arrow → Rust &[i64] slices
results = nanobook.sweep_equal_weight(
    prices=prices,              # Polars DataFrame, NOT list of dicts
    param_grid=param_grid,
    initial_cash=1_000_000_00,
    cost_model=cost_model,
)
```

### Implementation approach

| Approach | Deps added | Safety | Effort |
|----------|-----------|--------|--------|
| Raw PyCapsule + unsafe | 0 | Manual validation | Medium |
| `arrow-data` crate | 1 (small) | Type-safe FFI | Low |

Decide after profiling. Prefer `arrow-data` if it stays small.

### Files

- New: `python/src/arrow.rs`
- Update: `python/src/sweep.rs` (accept DataFrame OR list — backwards compat)
- Update: `python/nanobook.pyi` (overloaded signatures)

### Tests

- Polars DataFrame input produces same results as list input
- Benchmark: measure sweep time with 1000 params before/after
- Edge cases: empty DataFrame, single column, mismatched dtypes → clear error

### Acceptance

- Zero Python→Rust copies for price data in sweep
- Backwards compatible: list input still works
- Profile confirms measurable speedup (quantify in PR)

---

## WP4 — ITCH Parser (Stretch Goal)

> **Non-blocking for v0.6 GA.** Ship if ready; otherwise push to v0.6.x/v0.7.

### Problem

No way to ingest real market data. Users must construct synthetic order books.

### Solution

Parse NASDAQ TotalView-ITCH 5.0 binary format into nanobook `Event` stream.

```rust
use nanobook::itch;

let events = itch::parse_file("01302019.NASDAQ_ITCH50")?;
let mut exchange = Exchange::new();
for event in &events {
    exchange.apply(event);
}
```

Python:
```python
events = nanobook.parse_itch("01302019.NASDAQ_ITCH50")
ex = nanobook.Exchange.replay(events)
```

### Message coverage

| Message type | Parse | Convert to Event |
|-------------|-------|-----------------|
| Add Order (A/F) | yes | SubmitLimit |
| Execute (E/C) | yes | (implicit) |
| Cancel (X) | yes | Cancel (partial) |
| Delete (D) | yes | Cancel (full) |
| Replace (U) | yes | Modify |
| System/Stock events | yes | metadata only |
| Trade (P) | yes | off-book trade record |

### Feature gate

New feature: `itch` (optional, not default). Zero impact on core crate size.

### Deps

- `memmap2` (optional) for mmap large ITCH files (~5 GB/day)
- Or just `std::io::BufReader` for streaming parse (zero deps)

### Files

- New: `src/itch.rs` (parser + Event conversion)
- Update: `src/lib.rs` (conditional module)
- Update: `Cargo.toml` (feature flag)
- New: `python/src/itch.rs` (Python wrapper)
- New: `examples/itch_replay.rs`

### Tests

- Known-good ITCH snippet (small fixture, checked into `tests/fixtures/`)
- Round-trip: parse → replay → verify trade count matches expected
- Benchmark: parse rate (target: ≥ 10M messages/sec)

### Acceptance

- Parse a full day of NASDAQ ITCH data without error
- Replayed exchange matches expected trade counts
- Python API: `parse_itch()` returns event list

---

## Execution Order

```
WP1 (benchmarks) ──→ WP2 (O(1) cancel) ──→ WP3 (Arrow) ──→ release v0.6
                                                          ↗
                                        WP4 (ITCH) ──────  (if ready)
```

1. **WP1** first — establishes expanded baselines
2. **WP2** second — measure cancel improvement against baseline
3. **WP3** third — profile-gated, may be deferred if bottleneck is elsewhere
4. **WP4** in parallel — stretch goal, ships if ready

---

## Metrics

| Metric | v0.5 | v0.6 target |
|--------|------|-------------|
| Cancel latency (depth 1000) | ~60 μs | < 100 ns |
| Sweep data transfer | list copy | zero-copy Arrow |
| Benchmarked operations | 12 | 20+ |
| Benchmark regression CI | baseline only | active detection |
| Market data formats | none | ITCH 5.0 (stretch) |

---

## Out of Scope

| Feature | Why defer |
|---------|-----------|
| Multi-asset class (options/futures) | Needs design RFC — different matching semantics |
| OUCH protocol | Output protocol, less useful for backtesting |
| Free-threaded Python default | Wait for PyO3 ecosystem to stabilize |
| Code coverage CI | Nice-to-have, not user-facing |
