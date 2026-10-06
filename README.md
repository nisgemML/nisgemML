### Nishant Gemawat

Senior software engineer with 14+ years of experience, the recent ones in financial services. I'm currently on quantitative investment platform infrastructure at State Street Alpha Frontier (via TCS). Before that I spent four years at Morgan Stanley. I work in C++, C# and Python on Linux and Windows.

The six pinned repositories form one trading pipeline, from bytes on the wire to a research question:

```
 UDP multicast ──► ITCH decode ──► order book ──► matching ──► decision
 (gap recovery)    (C++ / FPGA)    (deep books)   (lock-free)   (+ is there a signal at all?)
```

| Repository | What it is | What it has been checked against |
|---|---|---|
| [**tick-to-trade**](https://github.com/nisgemML/tick-to-trade) | End-to-end C++ pipeline: MoldUDP64 feed → ITCH 5.0 → matching engine → strategy | **A full NASDAQ main-venue day** (2020-01-30, 423M messages): for AAPL, MSFT and TSLA, every price level of the engine's book equals an independent reference at all 12 checkpoints (52,745 levels, 0 mismatched) |
| [**lob-engine**](https://github.com/nisgemML/lob-engine) | Limit order book and matching engine: struct-of-arrays levels, AVX2 search, no allocation on the matching path | Fill-by-fill equality with a reference model on 5 × 1M random events under ASan/UBSan/TSan. Found and fixed an order-index collapse: cancel went from **3.5 ms to ~55 ns**, flat to 60,000 live orders |
| [**fpga-itch-parser**](https://github.com/nisgemML/fpga-itch-parser) | Synthesizable Verilog ITCH 5.0 decoder | **600,006 checks on real NASDAQ messages** against a third-party parser, 0 failures. 8/8 injected RTL bugs caught. Places and routes on a Lattice ECP5 at 137–141 MHz after two timing fixes (open-source flow, no board) |
| [**mpsc-queue**](https://github.com/nisgemML/mpsc-queue) | Vyukov MPSC queue with a written memory-model proof | CI on x86-64 **and AArch64**. The seq_cst cost is measured, not assumed (8.8 → 20.6 ns per push). States plainly that the queue is *not* lock-free or linearizable |
| [**udp-multicast-receiver**](https://github.com/nisgemML/udp-multicast-receiver) | MoldUDP64 receiver: gap detection, retransmit, A/B line arbitration | ITCH layout pinned by test vectors typed from the spec. Detects the kernel silently clamping its receive buffer |
| [**quant-signal-research**](https://github.com/nisgemML/quant-signal-research) | Pre-registered study of short-horizon BTCUSDT direction on 12 months of 1-minute data | Purged walk-forward validation, lookahead tests, Bonferroni-adjusted intervals. Result: **a detectable signal (AUC 0.521) far below the cost of trading it** (0.15 bps edge vs a 20 bps fee) |

#### How I work

Every repository documents the bugs I found in my own work, how each was caught, and the test that now pins it. A few that shaped how I test:

- **A test can agree with itself.** An ITCH parser passed every test because its fixtures shared its wrong byte layout. Golden vectors typed by hand from the NASDAQ spec now catch it, and they fail 11 checks against the old parser.
- **Realistic data can miss the worst case.** A hash bug that made cancels over 60,000× slower survived every correctness test, including a replay of a real exchange day, because none of them built a deep book with consecutive order IDs. The fix is pinned by a structural test (longest probe cluster), not a timing threshold.
- **Statistics need the same scrutiny as code.** Bootstrap intervals at a Bonferroni level, built from 300 resamples, were measurably too narrow. Fixing them reclassified one borderline result from "detectable" to "not". I published that change rather than keep the stronger-looking claim.
- **Say what isn't proven.** Benchmarks are labelled with the machine they ran on (laptop under WSL2, or a one-vCPU container), and each repo's `LIMITATIONS.md` lists what is *not* claimed: no FPGA board run, no co-location latency, no backtested P&L.

#### Background

M.S. in Computer Science, Texas A&M University–Commerce · Post-Graduate Certificate in AI/ML, Purdue University. Earlier work spans payments (Worldpay), where I built a C++ serial-comms subsystem called from .NET that removed ~300 ms garbage-collection latency spikes from card-terminal processing, plus enterprise systems at Dell, eBay and FedEx.

#### Contact

[LinkedIn](https://www.linkedin.com/in/nishant-gemawat-206969118) · ngemaw@outlook.com
