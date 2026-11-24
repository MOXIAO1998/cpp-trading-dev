# C++ Low-Latency Trading Simulator

A C++20 simulator for an electronic exchange and its trading clients. It models the order lifecycle from TCP submission and price–time priority matching to private execution reports and public market data. It includes latency instrumentation, a Jupyter notebook for analyzing logs, and standalone benchmarks for comparing implementation choices.

## Architecture

| Component | Responsibility |
| --- | --- |
| Order server | Receives new orders and cancellations over TCP, sequences requests, and returns private order responses. |
| Matching engine | Maintains an order book per instrument. Better prices match first, and orders at the same price match in FIFO order. |
| Market data publisher | Publishes incremental order book updates over UDP multicast. A separate snapshot feed lets clients recover from sequence gaps. |
| Trading client | Rebuilds the order book from market data, tracks positions and P&L, and applies pre-trade risk checks. Supports market-making, liquidity-taking, and random order generation. |
| Shared runtime | Provides lock-free queues for thread handoffs, memory pools for order book objects, and asynchronous logging. |

## Build and run

Use x86-64 Linux with a C++20-capable GCC toolchain, CMake, Ninja, and Bash. The timing instrumentation uses the x86 `RDTSC` instruction.

From the repository root, build the exchange and trading client:

```bash
cmake -S . -B cmake-build-release -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build cmake-build-release --target exchange_main trading_main -j 4
```

Start the exchange in the first terminal:

```bash
./cmake-build-release/exchange_main
```

Once the order server is listening, open a second terminal in the repository root and start the clients:

```bash
bash scripts/run_clients.sh
```

The default script launches a market maker (client 1) and a random order generator (client 5). Both processes write logs to the repository root when launched as shown above. After the clients finish, stop the exchange with `Ctrl+C` in the first terminal.

## Latency analysis

The exchange and clients log two kinds of measurements:

- **`RDTSC`** records the difference in timestamp counter values across individual code regions.
- **`TTT`** records nanosecond timestamps at points such as queue writes, queue reads, and socket operations.

Open [the analysis notebook](notebooks/perf_analysis.ipynb) with its kernel working directory set to `notebooks/`. It reads exchange and client 1 logs from the repository root and plots individual samples and rolling averages. Calibrate `CPU_FREQ` against the machine that collected the logs before converting TSC ticks to nanoseconds. The notebook estimates hop latency from adjacent records and does not correlate events by message ID.

A rendered [example analysis report](perf_analysis.html) is also included.

## Benchmarks

| Benchmark | Comparison |
| --- | --- |
| `logger_benchmark` | Per-character logging vs. fixed-size string entries |
| `release_benchmark` | Baseline vs. optimized memory pool operations |
| `hash_benchmark` | Array-backed lookup vs. `std::unordered_map` in the order book |

```bash
cmake --build cmake-build-release \
  --target logger_benchmark release_benchmark hash_benchmark -j 4

./cmake-build-release/logger_benchmark
./cmake-build-release/release_benchmark
./cmake-build-release/hash_benchmark
```
