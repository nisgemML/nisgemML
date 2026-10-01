# Nishant Gemawat — Quantitative Systems Engineer

Low-latency trading infrastructure · C++20/23/26 · Python · OCaml · Financial Markets

---

## System Architecture

15 repositories forming a coherent trading system — from raw market data through matching, execution, smart routing, and quantitative research. **tick-to-trade** is the integration layer: it actually wires the market-data and messaging layers below into one running, tested, measured pipeline, rather than leaving them as separate components that have only ever been benchmarked in isolation.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          MARKET DATA LAYER                               │
│                                                                          │
│  udp-multicast-receiver  ←  MoldUDP64 / ITCH 5.0 feed handler           │
│  SO_TIMESTAMPING · recvmmsg batch (64 datagrams/syscall) · 3/3 tests    │
│                                                                          │
│  fix-parser  ←  Zero-copy FIX 4.2/4.4 parser                           │
│  p50 112ns full parse · p50 60ns fast parse · std::span zero-copy       │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          MESSAGING LAYER                                 │
│                                                                          │
│  mpsc-queue  ←  Lock-free MPSC · beats mutex/spinlock/boost::lockfree   │
│  by 3.7-4.3x (real hardware, median of 5 runs) · six-claim formal proof │
│  · 18 TSan litmus tests · push_batch API                               │
│                                                                          │
│  io-uring-queue  ←  SPSC ring + io_uring async logger                  │
│  ring push/pop p50=215ns · io_uring cuts producer-thread p50 by ~9x    │
│  vs synchronous write() · 6,322 assertions across 2 CTest suites       │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION LAYER                                     │
│                                                                          │
│  tick-to-trade  ←  Four independently-deep repos (mpsc-queue,           │
│  io-uring-queue, udp-multicast-receiver, options-engine) composed       │
│  into one measurable pipeline, plus a real inventory-aware market       │
│  maker consuming its own fills. 12 real composition-layer bugs found    │
│  via ASan/TSan and real multi-core hardware — not a single-core dev    │
│  sandbox — documented in BUGS_FOUND.md, not hidden. MarketMaker's       │
│  inventory skew calibrated against real BTCUSDT volatility (Binance    │
│  public API), verified stable before shipping. io_uring logging is     │
│  actually wired in (-DHFT_WITH_IOURING=ON) and verified byte-identical │
│  against the default path — not vendored-but-unused.                   │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          MATCHING ENGINE                                 │
│                                                                          │
│  options-engine  ←  p50 submit 28ns (see that repo's own PROFILING.md   │
│  for a noted discrepancy against its README figure, not yet reconciled) │
│  SoA LOB · AVX2 SIMD find_level 2.0× · pool allocator                  │
│                                                                          │
│  low-latency-trading-engine  ←  full-stack C++20 + OCaml               │
│  ITCH 5.0 · Kyle λ microstructure · 6/6 test suites                    │
│                                                                          │
│  hash-map  ←  Robin Hood + SSE4.2 SIMD-probe hash maps                 │
│  avg probe < 1.5 (Robin Hood) · 16-slot SIMD groups · 1212/1212 tests  │
│                                                                          │
│  cpp26-alloc  ←  C++26 allocator · Contracts P2900R6                    │
│  std::generator · std::add_sat · std::saturate_cast · 102/102 tests    │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          EXECUTION LAYER                                 │
│                                                                          │
│  sor  ←  Smart Order Router                                             │
│  BestPrice / LowestFee / ProRata · 4-venue fee model · VWAP · 18 tests  │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          QUANTITATIVE RESEARCH                           │
│                                                                          │
│  quant-signal-research  ←  Short-horizon return-direction signal study │
│  on real BTCUSDT 1-minute data (Jul 2024-Jun 2025). AUC 0.521           │
│  [0.512, 0.529], Bonferroni-corrected, consistent across all 5          │
│  walk-forward folds — detectable, below a 40bps cost hurdle. Reported   │
│  honestly as "detectable, not tradeable" rather than oversold.          │
│                                                                          │
│  options-market-maker  ←  Heston Gil-Pelaez FFT · SSVI · vanna-volga   │
│  Sharpe 2.26 · 89 tests (Python + OCaml QCheck)                        │
│                                                                          │
│  avellaneda-stoikov  ←  Closed-form A-S market maker                   │
│  Sharpe 10.0 vs 3.6 baseline · 87 tests · multi-agent LOB simulation   │
│                                                                          │
│  lob-microstructure-calibration  ←  Kyle λ · Roll · kappa MLE · OFI   │
│  HAC-robust OLS · Bartlett-corrected autocovariance · 18 tests         │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                          FUNCTIONAL SYSTEMS                              │
│                                                                          │
│  ocaml-trading-primitives  ←  Functional LOB in OCaml                  │
│  Make(P:PRIORITY) functor · CME Rule 512.B · 11/11 QCheck tests        │
│                                                                          │
│  competitive-programming  ←  Trading-oriented algorithms                │
│  SegTree · SparseTable O(1) RMQ · DSU+rollback · CHT · 16/16 tests    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Index

### C++ Systems

| Repo | Signal | Key numbers |
|------|--------|-------------|
| [tick-to-trade](https://github.com/nisgemML/tick-to-trade) | Real integration: feed → MpscQueue → decision → IOURingLogger, plus a calibrated market maker | 12 real bugs found (ASan/TSan + real multi-core hardware), MarketMaker calibrated against real BTCUSDT volatility, io_uring logging verified wired-in and byte-identical to the default path |
| [mpsc-queue](https://github.com/nisgemML/mpsc-queue) | Lock-free MPSC, six-claim formal proof | 3.7-4.3x faster than mutex/spinlock/boost::lockfree::queue (real hardware, median of 5 runs), 18 TSan litmus tests |
| [options-engine](https://github.com/nisgemML/options-engine) | AVX2 SIMD matching engine | p50=28ns, 2.0× SIMD speedup, PROFILING.md (see note below on an unreconciled figure) |
| [fix-parser](https://github.com/nisgemML/fix-parser) | Zero-copy FIX 4.2/4.4 | 112ns full / 60ns fast, 34 tests |
| [udp-multicast-receiver](https://github.com/nisgemML/udp-multicast-receiver) | MoldUDP64/ITCH 5.0 feed handler | SO_TIMESTAMPING, recvmmsg, 3 tests |
| [hash-map](https://github.com/nisgemML/hash-map) | Robin Hood + SSE4.2 SIMD probe | avg probe <1.5, 1212 tests |
| [io-uring-queue](https://github.com/nisgemML/io-uring-queue) | SPSC ring + io_uring logger | ring push/pop p50=215ns, io_uring cuts producer-thread p50 ~9x vs write(), 6,322 assertions |
| [sor](https://github.com/nisgemML/sor) | Smart order router | BestPrice/LowestFee/ProRata, 18 tests |
| [cpp26-alloc](https://github.com/nisgemML/cpp26-alloc) | C++26 Contracts allocator | P2900R6, std::generator, 102 tests |
| [low-latency-trading-engine](https://github.com/nisgemML/low-latency-trading-engine) | Full-stack C++20 + OCaml | Kyle λ sim, ITCH 5.0, 6 test suites |

### Quantitative Research

| Repo | Signal | Key numbers |
|------|--------|-------------|
| [quant-signal-research](https://github.com/nisgemML/quant-signal-research) | Short-horizon return-direction signal study, real BTCUSDT 1-minute data | AUC 0.521 [0.512, 0.529], Bonferroni-corrected, fold-consistent (worst-fold AUC 0.515) — detectable, below a 40bps cost hurdle |
| [options-market-maker](https://github.com/nisgemML/options-market-maker) | Heston FFT, SSVI, vanna-volga | Sharpe 2.26, 89 tests |
| [avellaneda-stoikov](https://github.com/nisgemML/avellaneda-stoikov) | A-S stochastic control | Sharpe 10.0 vs 3.6 baseline, 87 tests |
| [lob-microstructure-calibration](https://github.com/nisgemML/lob-microstructure-calibration) | Kyle λ, Roll, kappa MLE, OFI | HAC-robust, real AAPL LOBSTER data |

### Functional Systems

| Repo | Signal | Key numbers |
|------|--------|-------------|
| [ocaml-trading-primitives](https://github.com/nisgemML/ocaml-trading-primitives) | Make(P:PRIORITY) functor, CME Rule 512.B | 11/11 QCheck property tests |
| [competitive-programming](https://github.com/nisgemML/competitive-programming) | SegTree, SparseTable, DSU+rollback, CHT | 16/16 tests, trading use cases |

---

## Key Design Decisions

**Why acq_rel is sufficient for the MPSC queue — no MFENCE needed**
The producer's release store on `next` and the consumer's acquire load establish
the happens-before chain that makes the payload visible. `seq_cst` (MFENCE on
x86) is unnecessary and costs real cycles per push for zero correctness benefit.
The formal proof in [mpsc-queue](https://github.com/nisgemML/mpsc-queue) documents
all six claims with an explicit x86-TSO instruction table proving the weaker
ordering suffices. Real hardware confirms the payoff: 3.7-4.3x faster than
mutex, spinlock, or `boost::lockfree::queue` on the same workload.

**Why the ring buffer helps even when io_uring's own completion latency doesn't**
[io-uring-queue](https://github.com/nisgemML/io-uring-queue)'s own measurements
across four logging strategies found something more interesting than "io_uring is
faster": decoupling the producer (matching-engine thread) from the logger via an
SPSC ring cuts the producer's own p50 by ~9x vs blocking synchronous `write()`
(267ns → 30ns) — and that win holds identically whether the *logger side* uses a
classic blocking writer or io_uring, because from the producer's perspective both
are just "push into a ring." What io_uring adds on top is asynchronous
*completion* on the logger side, not less contention on the producer's own hot
path. **SQPOLL, measured in that same environment, was ~52x worse at p50 than
plain io_uring** (7,988,146ns vs 154,341ns) — the opposite of SQPOLL's usual
selling point — because SQPOLL's kernel polling thread needs a dedicated pinned
core to deliver on its theoretical zero-syscall benefit, which that environment
didn't have. That root cause is documented, not hand-waved, in the repo's own
`BENCHMARK_RESULTS.md`, and it's a better story for an interview than a clean
win would have been: it shows the difference between a technique's asymptotic
promise and what it actually does on the hardware you're given.

**Why tick-to-trade vendors instead of using git submodules**
[tick-to-trade](https://github.com/nisgemML/tick-to-trade) copies the exact
header files it needs from mpsc-queue, io-uring-queue, and
udp-multicast-receiver rather than depending on them via submodule. A portfolio
reviewer gets a single `cmake && cmake --build`, no submodule init step, no risk
of an out-of-date submodule pointer silently changing what a benchmark actually
ran against. The trade-off, stated plainly: each vendored file's source of truth
remains its origin repo — a bug fix in the queue belongs in mpsc-queue, not
copy-pasted forward, and vendored copies can drift from their origin if not
periodically re-synced (this actually happened — `third_party/mpsc-queue`
inside tick-to-trade went stale relative to mpsc-queue's own real-hardware
benchmarking pass, and needed an explicit re-sync once caught). That trade-off
is worth it for a portfolio piece meant to be cloned and built in five minutes
by someone who has never seen the other 14 repos.

**Why SoA beats pointer-based order books by ~25ns per match**
A pointer-based LOB chases pointers across cache lines during the matching sweep —
3 cache misses per match. SoA keeps `prices[]` as a hot contiguous array; at 128
levels the entire array fits in L1 cache. Measured difference: ~25ns per match,
confirmed in options-engine benchmarks with committed PROFILING.md — though see
the note in the repository table above: that repo's PROFILING.md and its
README/BENCHMARK_RESULTS.md currently disagree on the exact AVX2 speedup figure
(1.6x vs 2.0x) for what should be the same benchmark, a sync gap flagged for
reconciliation, not yet fixed.

**Why Robin Hood with backward-shift deletion**
Tombstone deletion accumulates probe length under heavy cancel churn — degrades
to O(n) probe over time. Backward-shift maintains ≤1.5 average probe length
indefinitely. Fibonacci hashing gives uniform distribution on sequential order
IDs that modulo hashing clusters into the first N buckets.

**Why recvmmsg over recvmsg for market data**
At 1M packets/sec, individual recvmsg() calls cost 200ms CPU/sec in syscall
overhead alone. recvmmsg() with batch=64 reduces this to 3.1ms/sec — 64×
reduction. Trade-off: up to 63 × inter-packet latency added; acceptable for
feed handler, not for the matching engine.

**Why the A-S Sharpe (10.0) is honest, not cherry-picked**
1,000 paths, fresh seed per path, PnL = cash + inventory × final_mid, Sharpe
computed across all paths — not the best. Naive uses identical engine and seeds.
The 2.8× improvement is solely from inventory skew — ablated and documented in
[avellaneda-stoikov/SIMULATION_RESULTS.md](https://github.com/nisgemML/avellaneda-stoikov).

**Why the quant-signal-research verdict is "detectable, not tradeable" — and why that's the right answer, not a weak one**
[quant-signal-research](https://github.com/nisgemML/quant-signal-research)'s
primary pre-registered configuration (`logit_all`, 5-minute horizon, 1-bar
delay) detects a real signal — AUC 0.521 [0.512, 0.529], consistent across
every one of 5 walk-forward folds, not just a lucky holdout split — at a gross
edge of 0.15bps against a 40bps round-trip cost. That's gating by a mechanical
rule applied to a pre-registered config, not picking the best-looking metric
after the fact: a weaker single-feature baseline tested alongside it correctly
shows no detectable signal at the same horizon. The univariate feature-IC
table shows negative rank correlation on recent returns and taker imbalance —
the textbook signature of short-horizon microstructure reversal, not an
arbitrary pattern. Reporting "detectable but not tradeable" honestly, with the
walk-forward evidence to back it, is a stronger research-track signal than a
suspiciously clean profitable backtest would have been.

**Why the IC lookahead correction matters**
Rolling IC weights including future returns inflated the t-stat to 3.76. After
fixing the weight shift, t-stat = 1.08 — borderline, not highly significant.
This is the correct result. Reporting it honestly is the only valid approach.

**Why Make(P:PRIORITY) functor in OCaml**
PriceTime and ProRata priority are exchange-specific rules that change without
warning (CME Rule 512.B add_order_front for reduce-only replaces). The functor
separates the priority policy from the matching logic — swap priority without
touching the matching core. This is the idiom Jane Street uses in their trading
systems.

**Real bugs caught while integrating, kept as findings**
Two separate times, the same mistake: an unbounded feed thread racing ahead of
a slower consumer thread, measured as a huge but meaningless latency number
until bounded backpressure was added. First in mpsc-queue's own tick-to-trade
benchmark, then again in tick-to-trade's own pipeline — same root cause, same
fix, caught independently both times rather than assumed-fixed-by-association.
Separately: tick-to-trade's README once advertised "io_uring async logger" as
the active logging path; grepping the actual source showed the vendored
io-uring-queue dependency was never linked into the pipeline at all — real
dead weight behind real prose. Fixed by actually wiring a `HFT_WITH_IOURING`
build option and verifying byte-identical output against the default path
across 7,236 records, not just making the docs vaguer. Both are documented in
their respective BUGS_FOUND.md / git history as findings, not silently
patched — the point of documenting a caught bug is that it happened, not that
it was eventually fixed.

---

## Production Readiness Scorecard

| Component | Status | Remaining gap |
|-----------|--------|---------------|
| MPSC queue | ✅ Complete | Integration into engine hot path |
| Options matching engine | ✅ Complete | Multi-symbol sharding; PROFILING.md/README figure reconciliation |
| FIX parser | ✅ Complete | Full session-layer state machine |
| Market data feed handler | ✅ Complete | Hardware timestamp correlation |
| Hash map | ✅ Complete | In-engine integration benchmarks |
| io_uring async logger | ✅ Complete | Multi-sink backends |
| **tick-to-trade (integration)** | 🔧 **Mature, actively hardened — 12 real bugs found & fixed, real multi-core hardware validated** | Live multicast wiring (interface already exists, not yet connected); true `recv_ns` from SO_TIMESTAMPING; market maker explicitly not a calibrated trading strategy (no backtest harness) by its own documentation |
| **quant-signal-research** | ✅ **Complete as a research artifact** | Single symbol (BTCUSDT) and single asset class; a genuinely different signal or asset would need its own study, not an extension of this one |
| Smart Order Router | ✅ Complete | Real multi-venue historical data |
| cpp26-alloc | ✅ Complete | 24h stability test under load |
| A-S market maker | ✅ Complete | Real ITCH data calibration (in progress) |
| Microstructure calibration | ✅ Complete | Live LOB data pipeline |
| Options market maker | ✅ Complete | Live vol surface feed |
| Risk layer (pre-trade) | 🔧 In progress | Hard latency budgets |
| Multi-symbol sharding | 🔧 In progress | Per-symbol CPU affinity |
| Kernel bypass (DPDK) | 📋 Designed | NIC hardware required |
| 24h+ stability tests | 📋 Planned | Dedicated bare-metal required |

**Honest gaps:** Components are individually production-quality with real benchmarks.
tick-to-trade is the most mature demonstration of wiring several of them into one
measured system — not finished (it has its own documented roadmap), but no
longer a thin first pass either: 12 real composition bugs found and fixed,
validated on real multi-core hardware, with a market maker calibrated against
real market data rather than synthetic assumptions. quant-signal-research closes
the research-track gap with a real, walk-forward-verified finding rather than an
unexecuted pipeline. Remaining work across the portfolio: live multicast
wiring, long-running stability under realistic sustained load, and kernel-bypass
networking (hardware-dependent). These gaps close in the first 3-6 months at a
firm with appropriate infrastructure.

---

## Background

13 years delivering software infrastructure across financial services (Morgan Stanley,
State Street Alpha Frontier via TCS), payments (Worldpay), and enterprise systems.
M.S. Computer Science, Texas A&M University–Commerce.
Post-Graduate Certificate AI/ML, Purdue University.

---

*All benchmark numbers reproducible — build instructions in each repo's README.*
*Environment: Ubuntu 22.04/24.04, GCC 12/13/14, x86-64, plus WSL2 on real laptop*
*hardware for mpsc-queue's and tick-to-trade's cross-thread and multi-core*
*figures, and for quant-signal-research's real BTCUSDT data pull. Container*
*numbers reported honestly; single-vCPU-sandbox and pending-real-hardware*
*numbers labeled as such where applicable — see each repo's own*
*BENCHMARK_RESULTS.md for the exact tier.*
