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
│  SO_TIMESTAMPING · gap detection · PCAP replay · 65 tests               │
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
│  5-claim memory-model proof · x86-TSO · 18 TSan litmus tests            │
│                                                                          │
│  io-uring-queue  ←  SPSC ring + io_uring async logger                  │
│  p50 push 12ns · submission cost ~5ns (SQPOLL) · zero blocking          │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          MATCHING ENGINE                                 │
│                                                                          │
│  options-engine  ←  p50 add_order 38ns                                  │
│  SoA LOB · AVX2 SIMD 2.0× · pool allocator · libFuzzer · 159/159 tests │
│                                                                          │
│  low-latency-trading-engine  ←  full-stack C++20/OCaml                 │
│  RDTSC timing · CPU pinning · ITCH 5.0 · Avellaneda-Stoikov             │
│                                                                          │
│  hash-map  ←  Robin Hood + SSE4.2 SIMD-probe hash maps                 │
│  < 1.5 avg probe (Robin Hood) · 1 SIMD op · Fibonacci hashing           │
│                                                                          │
│  cpp26-alloc  ←  Lock-free slab allocator · Pure C++26                  │
│  Contracts P2900R6 · std::generator · std::expected · 102/102 tests     │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                       EXECUTION & ROUTING                                │
│                                                                          │
│  sor (Smart Order Router)  ←  Multi-venue US equity routing             │
│  BestPrice · LowestFee · ProRata · NYSE/NASDAQ/BATS/IEX fee model       │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    RESEARCH & SIGNAL LAYER                               │
│                                                                          │
│  avellaneda-stoikov  ←  Optimal market making                           │
│  Sharpe 10.02 vs naive 3.61 · 200 MC paths · kappa MLE                 │
│                                                                          │
│  options-market-maker  ←  Black-Scholes · Heston · vanna-volga          │
│  Delta hedger · Greeks management · backtest engine                      │
│                                                                          │
│  lob-microstructure-calibration  ←  Market microstructure               │
│  Kyle λ (HAC/Newey-West) · Roll · OFI (70/30 OOS R²) · κ MLE          │
│                                                                          │
│  alpha-research  ←  Signal research platform                            │
│  IC/ICIR · Brinson attribution · IC lookahead corrected (3.76→1.08)    │
│                                                                          │
│  ocaml-trading-primitives  ←  Functional trading primitives             │
│  14 canonical probability results · 16,709-line QCheck suite            │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## All Repositories

### C++20/23/26 — Systems & Infrastructure

| Repo | What it does | Key signal |
|------|-------------|------------|
| [mpsc-queue](https://github.com/nisgemML/mpsc-queue) | Lock-free MPSC queue | 53.1M msg/sec · 5-claim memory-model proof · x86-TSO · 18 TSan litmus tests · 20,218 tests |
| [options-engine](https://github.com/nisgemML/options-engine) | Options matching engine | p50 add_order 38ns · SoA LOB · AVX2 SIMD 2.0× · libFuzzer · 159/159 tests |
| [cpp26-alloc](https://github.com/nisgemML/cpp26-alloc) | Lock-free slab allocator | Pure C++26 · Contracts P2900R6 · std::generator · std::expected · 102/102 tests · TSan+ASan |
| [udp-multicast-receiver](https://github.com/nisgemML/udp-multicast-receiver) | Market data feed handler | MoldUDP64 · ITCH 5.0 · SO_TIMESTAMPING · PCAP replay · 65 tests |
| [fix-parser](https://github.com/nisgemML/fix-parser) | Zero-copy FIX 4.2/4.4 parser | p50 112ns full · p50 60ns fast · std::span zero-copy · no heap allocation |
| [hash-map](https://github.com/nisgemML/hash-map) | Robin Hood + SIMD-probe hash maps | < 1.5 avg probe · 1 SIMD op (SSE4.2) · Fibonacci hashing · backward-shift delete |
| [io-uring-queue](https://github.com/nisgemML/io-uring-queue) | SPSC ring + io_uring async logger | p50 push 12ns · ~5ns SQPOLL submission · zero hot-path blocking |
| [sor](https://github.com/nisgemML/sor) | Smart Order Router | BestPrice / LowestFee / ProRata · NYSE/NASDAQ/BATS/IEX · 18 tests |
| [low-latency-trading-engine](https://github.com/nisgemML/low-latency-trading-engine) | Full-stack trading engine | RDTSC · CPU pinning · ITCH 5.0 · A-S market maker |

### Python — Quantitative Research

| Repo | What it does | Key signal |
|------|-------------|------------|
| [avellaneda-stoikov](https://github.com/nisgemML/avellaneda-stoikov) | Optimal market making | Sharpe 10.02 vs naive 3.61 (2.8×) · 200 MC paths · Poisson MLE · adverse selection model |
| [alpha-research](https://github.com/nisgemML/alpha-research) | Alpha signal platform | IC/ICIR · Brinson-Hood-Beebower · purged k-fold CV · **IC lookahead corrected (t-stat 3.76→1.08)** |
| [lob-microstructure-calibration](https://github.com/nisgemML/lob-microstructure-calibration) | Microstructure estimators | Kyle λ (HAC/Newey-West) · Roll (delta SE) · OFI (OOS R²) · κ (Poisson MLE) |
| [options-market-maker](https://github.com/nisgemML/options-market-maker) | Options market maker | Black-Scholes · Heston · vanna-volga · delta hedger · Greeks management |

### OCaml — Functional Verification

| Repo | What it does | Key signal |
|------|-------------|------------|
| [ocaml-trading-primitives](https://github.com/nisgemML/ocaml-trading-primitives) | Functional trading primitives | 14 canonical probability results · 16,709-line QCheck property-based suite |

---

## Key Design Decisions

**Why acq_rel is sufficient for the MPSC queue — no MFENCE needed**
The producer's release store on `next` and the consumer's acquire load establish
the happens-before chain that makes the payload visible. `seq_cst` (MFENCE on
x86) is unnecessary and costs ~15 cycles per push. The formal proof in
[mpsc-queue](https://github.com/nisgemML/mpsc-queue) documents all 5 claims
with an explicit x86-TSO instruction table proving the weaker ordering suffices.

**Why SoA beats pointer-based order books by ~25ns per match**
A pointer-based LOB chases pointers across cache lines during the matching sweep —
3 cache misses per match. SoA keeps `prices[]` as a hot contiguous array; at 128
levels the entire array fits in L1 cache. Measured difference: ~25ns per match,
confirmed in options-engine benchmarks.

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

**Why the A-S Sharpe (10.02) is honest, not cherry-picked**
1,000 paths, fresh seed per path, PnL = cash + inventory × final_mid, Sharpe
computed across all paths — not the best. Naive uses identical engine and seeds.
The 2.8× improvement is solely from inventory skew — ablated and documented in
[avellaneda-stoikov/SIMULATION_RESULTS.md](https://github.com/nisgemML/avellaneda-stoikov).

**Why the IC lookahead correction matters**
Rolling IC weights including future returns inflated the t-stat to 3.76. After
fixing the weight shift, t-stat = 1.08 — borderline, not highly significant.
This is the correct result. Reporting it honestly is the only valid approach.

---

## Production Readiness Scorecard

| Component | Status | Remaining gap |
|-----------|--------|--------------|
| MPSC queue | ✅ Complete | Integration into engine hot path |
| Options matching engine | ✅ Complete | Multi-symbol sharding (in progress) |
| cpp26-alloc | ✅ Complete | 24h stability test under load |
| Market data feed handler | ✅ Complete | Hardware timestamp correlation |
| FIX parser | ✅ Complete | Full session-layer state machine |
| Hash map | ✅ Complete | In-engine integration benchmarks |
| io_uring async logger | ✅ Complete | Multi-sink backends |
| Smart Order Router | ✅ Complete | Real multi-venue historical data |
| A-S market maker | ✅ Complete | Real ITCH data calibration (in progress) |
| Alpha research | ✅ Complete | Live paper trading results |
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

12+ years delivering trading infrastructure at Morgan Stanley and State Street.
M.S. Computer Science (Texas A&M). Post-Graduate Certificate AI/ML (Purdue).
Self-study: Hull's *Options, Futures & Other Derivatives*; Shreve's *Stochastic
Calculus for Finance Vol II*. Active Putnam-style mathematics blog.

---

*All benchmark numbers reproducible — build instructions in each repo's README.*
*Environment: Ubuntu 22.04/24.04, GCC 13/14, x86-64 container. Container*
*numbers reported honestly; isolated-core predictions noted where applicable.*
