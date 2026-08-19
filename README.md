# Nishant Gemawat — Quantitative Systems Engineer

Low-latency trading infrastructure · C++20/23/26 · Python · OCaml · Financial Markets

---

## System Architecture

14 repositories forming a coherent trading system — from raw market data through matching, execution, smart routing, and quantitative research.

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
│  mpsc-queue  ←  Lock-free MPSC · 53.1M msg/sec                         │
│  Six-claim memory-model proof · 18 TSan litmus tests · push_batch API   │
│                                                                          │
│  io-uring-queue  ←  SPSC ring + io_uring async logger                  │
│  p50 push 12ns · IOURingLogger p50 548ns · 375/375 tests                │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          MATCHING ENGINE                                 │
│                                                                          │
│  options-engine  ←  p50 submit 28ns                                     │
│  SoA LOB · AVX2 SIMD find_level 2.0× · pool allocator · PROFILING.md   │
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
| [mpsc-queue](https://github.com/nisgemML/mpsc-queue) | Lock-free MPSC, six-claim formal proof | 53.1M msg/sec, 18 TSan litmus tests |
| [options-engine](https://github.com/nisgemML/options-engine) | AVX2 SIMD matching engine | p50=28ns, 2.0× SIMD speedup, PROFILING.md |
| [fix-parser](https://github.com/nisgemML/fix-parser) | Zero-copy FIX 4.2/4.4 | 112ns full / 60ns fast, 34 tests |
| [udp-multicast-receiver](https://github.com/nisgemML/udp-multicast-receiver) | MoldUDP64/ITCH 5.0 feed handler | SO_TIMESTAMPING, recvmmsg, 3 tests |
| [hash-map](https://github.com/nisgemML/hash-map) | Robin Hood + SSE4.2 SIMD probe | avg probe <1.5, 1212 tests |
| [io-uring-queue](https://github.com/nisgemML/io-uring-queue) | SPSC ring + io_uring logger | p50 push=12ns, 375 tests |
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
x86) is unnecessary and costs ~15 cycles per push. The formal proof in
[mpsc-queue](https://github.com/nisgemML/mpsc-queue) documents all six claims
with an explicit x86-TSO instruction table proving the weaker ordering suffices.

**Why SoA beats pointer-based order books by ~25ns per match**
A pointer-based LOB chases pointers across cache lines during the matching sweep —
3 cache misses per match. SoA keeps `prices[]` as a hot contiguous array; at 128
levels the entire array fits in L1 cache. Measured difference: ~25ns per match,
confirmed in options-engine benchmarks with committed PROFILING.md.

**Why Robin Hood with backward-shift deletion**
Tombstone deletion accumulates probe length under heavy cancel churn — degrades
to O(n) probe over time. Backward-shift maintains ≤1.5 average probe length
indefinitely. Fibonacci hashing gives uniform distribution on sequential order
IDs that modulo hashing clusters into the first N buckets.

**Why io_uring SQPOLL instead of a writer thread for logging**
A dedicated writer thread still crosses a queue boundary (cache miss + atomic).
io_uring SQPOLL submits writes via a plain memory store (~5ns) — the kernel
polls the SQ ring from a pinned kernel thread. No syscall, no context switch,
no interference with the matching thread critical path.

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

---

## Production Readiness Scorecard

| Component | Status | Remaining gap |
|-----------|--------|---------------|
| MPSC queue | ✅ Complete | Integration into engine hot path |
| Options matching engine | ✅ Complete | Multi-symbol sharding |
| FIX parser | ✅ Complete | Full session-layer state machine |
| Market data feed handler | ✅ Complete | Hardware timestamp correlation |
| Hash map | ✅ Complete | In-engine integration benchmarks |
| io_uring async logger | ✅ Complete | Multi-sink backends |
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
Remaining work: full system integration, long-running stability under realistic
sustained load, and kernel-bypass networking (hardware-dependent). These gaps
close in the first 3–6 months at a firm with appropriate infrastructure.

---

## Background

13 years delivering software infrastructure across financial services (Morgan Stanley,
State Street Alpha Frontier via TCS), payments (Worldpay), and enterprise systems.
M.S. Computer Science, Texas A&M University–Commerce.
Post-Graduate Certificate AI/ML, Purdue University.

---

*All benchmark numbers reproducible — build instructions in each repo's README.*
*Environment: Ubuntu 22.04/24.04, GCC 12/13/14, x86-64. Container numbers reported*
*honestly; isolated-core predictions noted where applicable.*
