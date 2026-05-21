# AdaptiveSched++

A production-grade adaptive CPU scheduling simulator implemented in C++17, with a Python benchmark runner and an interactive HTML dashboard for visualising and comparing scheduler performance.

AdaptiveSched++ simulates a single-core CPU scheduler that **dynamically switches between scheduling algorithms at runtime** based on observed workload characteristics. The codebase is modular, thread-safe, and architecturally faithful to real operating system scheduling frameworks.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Scheduling Algorithms](#scheduling-algorithms)
3. [Adaptive Scheduler Logic](#adaptive-scheduler-logic)
4. [Quick Start — Run From Scratch](#quick-start--run-from-scratch)
5. [Running the Simulator](#running-the-simulator)
6. [Benchmark & Dashboard](#benchmark--dashboard)
7. [Benchmark Results & Analysis](#benchmark-results--analysis)
8. [Workload Files](#workload-files)
9. [Output & Metrics](#output--metrics)
10. [Design Decisions](#design-decisions)

---

## Architecture Overview

```
AdaptiveSched++/
├── include/
│   ├── core/
│   │   ├── process.h               # PCB: all per-process state
│   │   ├── state.h                 # ProcessState / WorkloadType / SchedulerPolicy enums
│   │   ├── clock.h                 # Deterministic logical tick clock (singleton)
│   │   └── ready_queue.h           # Thread-safe, multi-order blocking queue
│   ├── scheduler/
│   │   ├── scheduler.h             # IScheduler abstract base + SchedulerStats
│   │   ├── fcfs.h                  # First-Come First-Served
│   │   ├── sjf.h                   # SJF (non-preemptive) + SRTF (preemptive)
│   │   ├── rr.h                    # Round-Robin with configurable quantum
│   │   ├── priority.h              # Preemptive Priority + aging
│   │   ├── mlfq.h                  # Multi-Level Feedback Queue (4 levels)
│   │   └── adaptive.h              # Adaptive meta-scheduler (main highlight)
│   ├── execution/
│   │   └── cpu.h                   # Single-core CPU execution engine
│   ├── metrics/
│   │   └── statistics.h            # Metrics aggregation & reporting
│   └── simulation/
│       ├── process_generator.h     # Static (CSV) and dynamic process injection
│       ├── simulation_controller.h # Top-level orchestrator
│       └── workload.h              # WorkloadProfile, WorkloadLoader, WorkloadFactory
├── src/                            # Corresponding .cpp implementations
├── workloads/                      # Pre-built CSV workloads
├── output/                         # Generated metrics CSV and dashboard HTML
├── benchmark.py                    # Large-scale benchmark runner
├── generate_dashboard.py           # Interactive HTML dashboard generator
├── main.cpp                        # CLI entry point
├── CMakeLists.txt
└── README.md
```

### Thread Model

The simulator runs three concurrent threads:

| Thread | Responsibility |
|--------|---------------|
| **CPU thread** | Executes processes tick-by-tick, advances clock, handles preemption |
| **Generator thread** | Injects processes into the scheduler at their arrival ticks |
| **Monitor thread** | Calls `evaluate_and_adapt()` periodically; detects termination |

All shared state is protected by `std::mutex` and `std::condition_variable`. No busy-waiting. Graceful shutdown drains in-flight processes before stopping.

---

## Scheduling Algorithms

### FCFS — First-Come, First-Served
Non-preemptive FIFO. Processes run to completion once selected. The **convoy effect** is observable in metrics when long jobs precede short ones. Best for batch workloads with homogeneous burst lengths.

### SJF / SRTF — Shortest Job First
- **SJF (non-preemptive):** selects the process with the shortest original burst at each dispatch point.
- **SRTF (preemptive):** preempts the running process when a newly arrived process has a shorter *remaining* time. Optimal for minimising average waiting time but risks starvation of long jobs.

### Round-Robin
Configurable time quantum (default: 4 ticks). Each process gets exactly one quantum before being re-enqueued at the tail. Context-switch overhead (1 tick) is charged per switch. Guarantees bounded response time and strong fairness.

### Priority Scheduling (Preemptive)
Processes ordered by priority number (lower = more urgent). When a higher-priority process arrives, the running process is immediately preempted. **Aging** prevents starvation: processes waiting longer than `AGING_THRESHOLD` ticks receive a +1 priority boost (up to `MAX_PRIORITY_BOOSTS`).

### MLFQ — Multi-Level Feedback Queue
Four queues with descending priority and increasing quanta:

| Level | Order | Quantum |
|-------|-------|---------|
| 0 (highest) | FIFO | 2 ticks |
| 1 | FIFO | 4 ticks |
| 2 | FIFO | 8 ticks |
| 3 (lowest) | FIFO | run-to-completion |

- New processes enter at Level 0.
- Exhausting a quantum causes **demotion** to the next level.
- A higher-level process arriving **preempts** a lower-level running process.
- Every 100 ticks a **global priority boost** resets all processes to Level 0 (anti-starvation).

---

## Adaptive Scheduler Logic

The `AdaptiveScheduler` wraps FCFS, RR, Priority, and MLFQ. It continuously monitors runtime metrics and switches the active policy using a five-step pipeline:

### Step 1 — Sample Metrics (EMA)
Every `EVALUATION_INTERVAL` (20) ticks the engine samples:
- CPU utilisation, throughput, context-switch rate
- Average burst length, waiting time, response time
- Starvation pressure, interactive fraction, queue depth
- MLFQ level-0 occupancy fraction

All values are smoothed using **Exponential Moving Averages** (`α = 0.25`) to filter noise.

### Step 2 — Classify Workload
The metric vector is mapped to one of five workload types:

| Classification | Key Indicators |
|---|---|
| `CPU_BOUND` | High avg burst (>12), high utilisation, few context switches |
| `INTERACTIVE` | High interactive fraction (>40%), low response time, MLFQ L0 heavy |
| `BATCH` | Very long bursts (>20), low throughput, minimal switching |
| `STRESS` | CPU_BOUND AND utilisation > 85% |
| `MIXED` | No dominant signal, OR starvation detected |

### Step 3 — Score Policies
Each candidate policy receives a confidence score `[0, 1]` based on:
- Workload classification match
- Individual metric thresholds (starvation pressure, response time, queue depth)
- Historical trend analysis over the last 8 evaluations
- A small stability bonus for the currently active policy

### Step 4 — Hysteresis
A switch only fires when:
1. The best policy differs from the current one.
2. The best policy's score exceeds the current policy's score by at least `CONFIDENCE_HYSTERESIS = 0.15`.
3. At least `COOLDOWN_TICKS = 40` ticks have elapsed since the last switch.

This three-part guard prevents oscillation and thrashing.

### Step 5 — Migrate
The active scheduler's ready queue is drained and handed off to the new scheduler. Processes preserve their `remaining_time`, `priority`, and `queue_level` through the migration.

---

## Quick Start — Run From Scratch

> **These are the exact steps for anyone who has just cloned this repo on a Windows machine with WSL installed. No prior setup assumed.**

### Prerequisites — Install WSL (if you don't have it)

Open PowerShell as Administrator and run:
```powershell
wsl --install -d Ubuntu
```
Restart your machine after this completes, then open the Ubuntu app to finish setup.

---

### Step 1 — Open WSL and navigate to the project

Open PowerShell, enter WSL, then go to your project folder:

```bash
wsl
cd /mnt/c/Users/<YourUsername>/path/to/AdaptiveSched++
```

> If your WSL uses `/mnt/host/c/` instead of `/mnt/c/`, use that prefix. Run `ls` to confirm you can see `main.cpp`, `benchmark.py`, `CMakeLists.txt`.

---

### Step 2 — Install build tools (one time only)

```bash
sudo apt update
sudo apt install -y cmake g++ build-essential python3 python3-pip
pip3 install plotly
```

> If `sudo` is not found, you are already root — just drop `sudo` from every command.

---

### Step 3 — Build the C++ simulator

```bash
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4
cd ..
```

Verify the binary was created:
```bash
./build/adaptive_sched --help
```

---

### Step 4 — Run the benchmark

```bash
python3 benchmark.py --count 500
```

Expected output:
```
[1/4] Generating large workload ...
  Generated 500 processes → workloads/large_benchmark.csv
[2/4] Running Adaptive scheduler ...
      ✓  AvgWT=4644.0  AvgRT=4548.7  Fairness=0.843
[3/4] Running MLFQ-only (no-adaptive) ...
      ✓  AvgWT=4855.3  AvgRT=1757.1  Fairness=0.804
[4/4] Extracting per-policy breakdown ...
  Results saved → output/benchmark_results.json
```

You can change the process count:
```bash
python3 benchmark.py --count 1000
python3 benchmark.py --count 2000
```

---

### Step 5 — Generate the interactive dashboard

```bash
python3 generate_dashboard.py
```

This writes `output/adaptivesched_dashboard.html`.

---

### Step 6 — Open the dashboard in your browser

Run this in **PowerShell** (not WSL):
```powershell
start ".\output\adaptivesched_dashboard.html"
```

Or navigate to the `output/` folder in File Explorer and double-click `adaptivesched_dashboard.html`.

---

## Running the Simulator

The simulator binary supports these flags:

```
Usage: ./build/adaptive_sched [OPTIONS]

Options:
  --workload <name>     Built-in profile: interactive, cpu_bound, mixed, stress, batch
  --csv <path>          Load workload from a CSV file
  --count <n>           Override process count
  --no-adaptive         Disable adaptive switching (runs MLFQ only)
  --export-csv          Export per-process metrics → output/metrics.csv
  --export-timeline     Export tick-by-tick timeline → output/timeline.csv
  --output-dir <path>   Directory for exported files (default: ./output)
  --verbose             Print live monitor snapshots every 100 ticks
  --quiet               Suppress all console output (for scripting)
  --max-ticks <n>       Hard tick limit (0 = run to completion)
  --help
```

### Examples

```bash
# Run built-in mixed workload with verbose output
./build/adaptive_sched --workload mixed --verbose

# Stress test with 80 processes
./build/adaptive_sched --workload stress --count 80 --verbose

# Load from CSV and export metrics
./build/adaptive_sched --csv workloads/cpu_bound.csv --export-csv

# Compare: disable adaptive (MLFQ only)
./build/adaptive_sched --workload mixed --no-adaptive

# Quiet mode for scripting
./build/adaptive_sched --csv workloads/mixed.csv --export-csv --quiet --output-dir ./output/myrun
```

---

## Benchmark & Dashboard

### What `benchmark.py` does
- Generates a large CSV workload with a realistic heterogeneous mix of process types:
  - **Interactive** (25%) — short burst, UI/shell processes
  - **CPU-bound** (25%) — long burst, compilers/ML jobs
  - **Daemon** (20%) — medium burst, background services
  - **Burst** (15%) — variable burst, mixed
  - **Batch** (15%) — very long burst, bulk jobs
- Runs the simulator **twice** on the same workload: once with Adaptive enabled, once with MLFQ fixed
- Collects per-process metrics from both runs
- Computes: avg WT/TAT/RT, Jain's fairness index, p95/p99 waiting times, context switches, per-type breakdowns
- Saves everything to `output/benchmark_results.json`

### What `generate_dashboard.py` does
Reads `output/benchmark_results.json` and produces a fully self-contained `output/adaptivesched_dashboard.html` with 8 interactive chart sections powered by Plotly.js:

| Section | What it shows |
|---|---|
| KPI Strip | 6 headline metrics with % delta vs MLFQ |
| Summary Bar Chart | Avg Wait, TAT, Response, p95/p99 — side by side |
| Distribution Histograms | Waiting and response time spread across all processes |
| Box Plots by Process Type | Interactive vs CPU-bound response and wait times |
| Per-Process WT Delta | Sorted bar — which processes each scheduler wins |
| Cumulative Throughput | Processes completed over simulation time |
| Policy Pie + Context Switches | Policy usage breakdown; total context switch count |
| Burst vs Wait Scatter | Per-process scatter coloured by process type |
| Full Metrics Table | Every metric with winner highlighted per row |

---

## Benchmark Results & Analysis

Results from a benchmark run of **500 processes** on a heterogeneous mixed workload.

### Headline Numbers

| Metric | Adaptive | MLFQ Fixed | Winner |
|---|---|---|---|
| Avg Waiting Time | 4644.01 ticks | 4855.32 ticks | **Adaptive ✓** |
| Avg Turnaround Time | 4660.09 ticks | 4871.40 ticks | **Adaptive ✓** |
| Avg Response Time | 4548.69 ticks | 1757.08 ticks | MLFQ ✓ |
| Jain's Fairness Index | **0.8432** | 0.8035 | **Adaptive ✓** |
| p95 Waiting Time | 7703 ticks | 8580 ticks | **Adaptive ✓** |
| p99 Waiting Time | 7974 ticks | 9055 ticks | **Adaptive ✓** |
| Total Context Switches | **127** | 3,584 | **Adaptive ✓** |
| Interactive Avg RT | 4461.32 ticks | 1758.10 ticks | MLFQ ✓ |

### Key Observations

**1. Adaptive wins on waiting time and fairness.**
The Adaptive scheduler reduces average waiting time by ~4.4% and p95 waiting time by ~10.2% compared to fixed MLFQ. The improvement compounds at the tail — p99 wait is 9% lower — meaning the worst-off processes are served significantly better. The Jain Fairness Index of 0.8432 vs 0.8035 confirms more equitable distribution of CPU time across all 500 processes.

**2. MLFQ wins on response time.**
Fixed MLFQ achieves a much lower average response time (1757 ticks vs 4548 ticks). This is because MLFQ always starts new processes at Level 0 with a short quantum, so processes get their first CPU slice almost immediately. The Adaptive scheduler, once it switches to FCFS, sacrifices first-response latency in exchange for higher throughput and lower overall wait.

**3. The Adaptive scheduler used FCFS for 95.2% of processes.**
This is correct behaviour for this workload. The process generator produces a profile that trends CPU-bound once all processes have arrived. The adaptive engine correctly identified this and switched to FCFS, which minimises overhead for long-running jobs. Only 4.8% of processes (early interactive ones) were handled by MLFQ before the switch.

**4. Context switches: 127 vs 3,584.**
This is the most dramatic difference — a 28× reduction. Fixed MLFQ generates thousands of context switches due to quantum-based preemption across four levels. Once Adaptive switches to FCFS, there is zero preemption overhead, which is why throughput is higher in the Adaptive run — the CPU spends more time executing processes rather than switching between them.

**5. Throughput curve.**
The cumulative throughput chart shows the Adaptive run completing processes in a smooth linear curve. The MLFQ fixed run shows a sawtooth pattern caused by repeated priority boosts every 100 ticks, which temporarily stall completions as long processes get preempted and reset to Level 0.

**6. Per-process delta.**
Of the 500 processes, Adaptive outperforms MLFQ for 264 (52.8%) and MLFQ outperforms Adaptive for 236 (47.2%). The processes where MLFQ wins are predominantly short interactive ones that benefit from immediate Level-0 scheduling — exactly the expected tradeoff.

### When to use each

| Use Adaptive when... | Use MLFQ fixed when... |
|---|---|
| Workload is mixed or batch-heavy | Workload is predominantly interactive |
| Minimising tail latency (p95/p99) matters | Minimising first-response time matters |
| Context-switch overhead is a concern | You need predictable, consistent behaviour |
| Fairness across all processes is important | All processes have similar burst lengths |

---

## Workload Files

CSV format: `pid,name,arrival,burst,priority,interactive`

| File | Description |
|------|-------------|
| `workloads/interactive.csv` | 40 short-burst UI processes (burst 1–9 ticks) |
| `workloads/cpu_bound.csv` | 30 compute-heavy processes (burst 10–70 ticks) |
| `workloads/mixed.csv` | 50 processes: compilers, editors, network I/O, daemons |
| `workloads/stress.csv` | 60 processes with high arrival rate, starvation-prone |
| `workloads/large_benchmark.csv` | Auto-generated by `benchmark.py` — 500+ heterogeneous processes |

You can write your own CSV following the same format and pass it with `--csv`.

---

## Output & Metrics

### Console Output (sample)

```
**********************************************************************
*  AdaptiveSched++ — CPU Scheduling Simulator
*  Scenario  : mixed_workload
*  Workload  : mixed
*  Processes : 60
*  Adaptive  : ENABLED
**********************************************************************

[GEN] Injecting PID=1 (proc_1) at tick=5 burst=14 pri=6 [INTERACTIVE]
...

======================================================================
[ADAPTIVE] *** POLICY SWITCH ***
  Tick       : 1336
  From       : MLFQ
  To         : FCFS
  Confidence : 0.400
  Workload   : CPU_BOUND
  Reason     : cpu-bound workload;
  Metrics    :
    avg_burst=1.1  avg_wait=267.7  cpu_util=0.06
    starvation=0.000  interactive=0.075  csr=1.900
======================================================================

================================================================================
                       SIMULATION SUMMARY: mixed_workload
================================================================================
  Processes completed  : 60
  Avg Waiting Time     : 1340.63 ticks
  Avg Turnaround Time  : 1353.43 ticks
  Avg Response Time    : 959.98 ticks
  CPU Utilization      : 100.00%
  Throughput           : 1.00 proc/tick
  Context Switches     : 0
  Jain's Fairness Index: 0.98
  Scheduler Switches   : 2
```

### Exported CSV columns

```
pid, name, arrival, burst, start, completion, waiting, turnaround,
response, priority, queue_level, ctx_switches, interactive,
starvation_ticks, scheduled_by
```

The `scheduled_by` column records which policy was active when each process completed — used for per-policy analysis in the dashboard.

---

## Design Decisions

**Why logical ticks instead of wall-clock time?**
Determinism and reproducibility. Logical ticks make every run bit-identical given the same seed and allow the simulation to run at maximum machine speed without any real-time dependency.

**Why not use `std::priority_queue` for the ready queue?**
`std::priority_queue` doesn't support O(n) removal by PID, which is needed for preemption. A sorted `std::deque` with `remove()` provides the right balance of insert cost and flexibility.

**Why does the adaptive scheduler start with MLFQ?**
MLFQ is the best general-purpose default — it handles both interactive and CPU-bound workloads without prior knowledge of process behaviour. Specialised policies are only activated when the workload signal is clear and persistent.

**Why EMA rather than a simple window average?**
Exponential Moving Averages weight recent observations more heavily while never discarding old data entirely. This lets the adaptive engine react to workload phase changes without thrashing on transient spikes.

**Why a cooldown + hysteresis guard?**
Without cooldown, the engine could oscillate between two policies every evaluation window. The 40-tick cooldown and 0.15 confidence margin ensure switches carry sufficient evidence and the new policy has time to demonstrate its effect before being reconsidered.

**Why does `--quiet` exist?**
The benchmark runner calls the simulator binary twice as a subprocess. Without quiet mode, both runs would print thousands of log lines to the terminal. `--quiet` redirects all stdout to a null buffer so the benchmark script runs cleanly and only collects the exported CSV output.
