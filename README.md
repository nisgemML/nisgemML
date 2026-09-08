# Nishant Gemawat — Quantitative Systems Engineer

Low-latency trading infrastructure · C++20/23/26 · Python · OCaml · Financial Markets

---

## System Architecture

15 repositories forming a coherent trading system — from raw market data through matching, execution, smart routing, and quantitative research. **tick-to-trade** is the integration layer: it actually wires the market-data and messaging layers below into one running, tested, measured pipeline, rather than leaving them as separate components that have only ever been benchmarked in isolation.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          MARKET DATA LAYER                               │
│                                                                          │
│  udp-multicast-receiver  ←  MoldUDP64 / ITCH 5.0 feed handler            │
│  SO_TIMESTAMPING · recvmmsg batch (64 datagrams/syscall) · 3/3 tests     │
│                                                                          │
│  fix-parser  ←  Zero-copy FIX 4.2/4.4 parser                             │
│  p50 112ns full parse · p50 60ns fast parse · std::span zero-copy        │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          MESSAGING LAYER                                 │
│                                                                          │
│  mpsc-queue  ←  Lock-free MPSC · beats mutex/spinlock/boost::lockfree    │
│  by 3.7-4.3x (real hardware, median of 5 runs) · six-claim formal proof  │
│  · 18 TSan litmus tests · push_batch API                                 │
│                                                                          │
│  io-uring-queue  ←  SPSC ring + io_uring async logger                    │
│  ring push/pop p50=215ns · io_uring cuts producer-thread p50 by ~9x      │
│  vs synchronous write() · 6,322 assertions across 2 CTest suites         │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION LAYER — NEW                               │
│                                                                          │
│  tick-to-trade  ←  Real two-thread pipeline: feed → MpscQueue →          │
│  decision → IOURingLogger, using the actual source of the three          │
│  repos above (vendored, not reimplemented). Differential-tested          │
│  against an independent reference order book, deterministic replay,      │
│  a real methodology mistake caught mid-benchmark and documented.         │
│  Milestones 1-3 of 5 complete (skeleton, correctness hardening,          │
│  measurement) — see that repo's own README for what's still open.        │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          MATCHING ENGINE                                 │
│                                                                          │
│  options-engine  ←  p50 submit 28ns (see that repo's own PROFILING.md    │
│  for a noted discrepancy against its README figure, not yet reconciled)  │
│  SoA LOB · AVX2 SIMD find_level 2.0× · pool allocator                    │
│                                                                          │
│  low-latency-trading-engine  ←  full-stack C++20 + OCaml                 │
│  ITCH 5.0 · Kyle λ microstructure · 6/6 test suites                      │
│                                                                          │
│  hash-map  ←  Robin Hood + SSE4.2 SIMD-probe hash maps                   │
│  avg probe < 1.5 (Robin Hood) · 16-slot SIMD groups · 1212/1212 tests    │
│                                                                          │
│  cpp26-alloc  ←  C++26 allocator · Contracts P2900R6                     │
│  std::generator · std::add_sat · std::saturate_cast · 102/102 tests      │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          EXECUTION LAYER                                 │
│                                                                          │
│  sor  ←  Smart Order Router                                              │
│  BestPrice / LowestFee / ProRata · 4-venue fee model · VWAP · 18 tests   │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          QUANTITATIVE RESEARCH                           │
│                                                                          │
│  options-market-maker  ←  Heston Gil-Pelaez FFT · SSVI · vanna-volga     │
│  Sharpe 2.26 · 89 tests (Python + OCaml QCheck)                          │
│                                                                          │
│  avellaneda-stoikov  ←  Closed-form A-S market maker                     │
│  Sharpe 10.0 vs 3.6 baseline · 87 tests · multi-agent LOB simulation     │
│                                                                          │
│  lob-microstructure-calibration  ←  Kyle λ · Roll · kappa MLE · OFI      │
│  HAC-robust OLS · Bartlett-corrected autocovariance · 18 tests           │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                          FUNCTIONAL SYSTEMS                              │
│                                                                          │
│  ocaml-trading-primitives  ←  Functional LOB in OCaml                    │
│  Make(P:PRIORITY) functor · CME Rule 512.B · 11/11 QCheck tests          │
│                                                                          │
│  competitive-programming  ←  Trading-oriented algorithms                 │
│  SegTree · SparseTable O(1) RMQ · DSU+rollback · CHT · 16/16 tests       │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Index

### C++ Systems

| Repo | Signal | Key numbers |
|------|--------|-------------|
| [tick-to-trade](https://github.com/nisgemML/tick-to-trade) | Real integration: feed → MpscQueue → decision → IOURingLogger | 4,850 differential comparisons vs an independent reference book, deterministic replay, Milestones 1-3/5 complete |
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
periodically re-synced. That trade-off is worth it for a portfolio piece meant
to be cloned and built in five minutes by someone who has never seen the other
14 repos.

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

**A real bug caught while integrating, kept as a finding**
tick-to-trade's first working pipeline had no backpressure between the feed
thread and the decision thread — the feed thread, pure CPU-bound parsing, raced
arbitrarily far ahead of the decision thread, which does real work per event.
Result: a 3.9ms mean latency that was almost entirely queue backlog, not
processing cost. This was the identical mistake an earlier version of
mpsc-queue's own tick-to-trade benchmark shipped with. Same fix both times:
bound how far ahead the producer can get. Kept in both repos' git history and
documentation as a finding, not silently corrected — the point of documenting a
caught bug is that it happened, not that it was eventually fixed.

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
| **tick-to-trade (integration)** | 🔧 **In progress — Milestones 1-3/5** | Live multicast wiring, gap/loss injection at integration level, logger-overflow policy, LIMITATIONS.md, interview-prep polish |
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
tick-to-trade is the first attempt at wiring several of them into one measured
system rather than leaving that integration as an unverified claim — it's
mid-way through its own 5-milestone plan, not finished, and says so in its own
README rather than being presented as done. Remaining work across the portfolio:
full system integration beyond what tick-to-trade covers so far, long-running
stability under realistic sustained load, and kernel-bypass networking
(hardware-dependent). These gaps close in the first 3-6 months at a firm with
appropriate infrastructure.

---

## Background

13 years delivering software infrastructure across financial services (Morgan Stanley,
State Street via TCS), payments (Worldpay), and enterprise systems.
M.S. Computer Science, Texas A&M University–Commerce.
Post-Graduate Certificate AI/ML, Purdue University.

---

*All benchmark numbers reproducible — build instructions in each repo's README.*
*Environment: Ubuntu 22.04/24.04, GCC 12/13/14, x86-64, plus WSL2 on real laptop*
*hardware for mpsc-queue's cross-thread figures. Container numbers reported*
*honestly; single-vCPU-sandbox and pending-real-hardware numbers labeled as such*
*where applicable — see each repo's own BENCHMARK_RESULTS.md for the exact tier.*
