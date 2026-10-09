# multichase Pointer-Chase Benchmark

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![CI](https://github.com/garys-gpu-benchmarks/309-sys-bench-amd-multichase-numa-latency-ubu2604/actions/workflows/ci.yml/badge.svg)](https://github.com/garys-gpu-benchmarks/309-sys-bench-amd-multichase-numa-latency-ubu2604/actions/workflows/ci.yml)

Target: Ubuntu 26.04 · AMD · see Hardware Requirements. This is a host benchmark, not a laptop `pip install` project.

## Quick Start

```bash
git clone https://github.com/garys-gpu-benchmarks/309-sys-bench-amd-multichase-numa-latency-ubu2604.git
cd 309-sys-bench-amd-multichase-numa-latency-ubu2604
sudo bash setup.sh --assume-yes
bash run_benchmark.sh --profile smoke --validate
```
Results are written to `results/benchmark.db` and `results/summary.json`.

This workload is executed on the validation host after the repository is copied there. `setup.sh` and `run_benchmark.sh` do not open an outbound SSH session.

Prerequisites: Ubuntu 26.04; AMD; Python 3.14.4; root or sudo for `setup.sh`. Framework: Bash, SQLite, Python, PyYAML, C (not C++), GCC, NUMA, numactl. This is a host benchmark, not a laptop `pip install` project.

```mermaid
flowchart LR
  setup.sh --> run_benchmark.sh --> parse_results.py --> results/benchmark.db
```

## 1. Overview

Compiles src/multichase.c and runs one pointer-chase per yaml working_set_size tier under numactl via scripts/collect_multichase.py. working_set_size expands from yaml. stride, warmup_iters, and num_iterations use collect_multichase.py defaults (64, 10000, 200000) because collect_workload.py does not forward yaml or run_benchmark values. remote_dram runs with --cpunodebind=0 --membind=1 when numactl --hardware reports two or more NUMA nodes; on a single-node host it is reported as not measured. Sweep dimensions: num_numa_nodes, num_threads, thread_affinity, page_size, working_set_size, stride, access_pattern, prefetch_enabled.

## 2. What It Validates

- Validates one latency_ns per working-set tier. Fails if fewer than two samples or any zero latency
- #1: Pointer-chase latency for the L1-cache tier (latency_ns_l1); is present and physically sensible.
- #2: Pointer-chase latency for the L2-cache tier (latency_ns_l2); is present and physically sensible.
- #3: Pointer-chase latency for the L3-cache tier (latency_ns_l3); is present and physically sensible.
- #4: Local DRAM latency, ns (latency_ns_local_dram); is present and physically sensible.
- #5: Remote DRAM latency, ns (latency_ns_remote_dram) is present and physically sensible.

## 3. Metrics Captured

- **#1: Pointer-chase latency for the L1-cache tier** — stored as `latency_ns_l1`.
- **#2: Pointer-chase latency for the L2-cache tier** — stored as `latency_ns_l2`.
- **#3: Pointer-chase latency for the L3-cache tier** — stored as `latency_ns_l3`.
- **#4: Local DRAM latency, ns** — stored as `latency_ns_local_dram`.
- **#5: Remote DRAM latency, ns** — stored as `latency_ns_remote_dram`.

## 4. Hardware Requirements

### Supported environment

- OS: Ubuntu 26.04
- GPU vendor: AMD
- Framework family: Bash, SQLite, Python, PyYAML, C (not C++), GCC, NUMA, numactl
- Python: Python 3.14.4

### Reference validation environment

The tables below describe the machine used to generate the reference results. They are not a requirement that every user buy that exact cloud instance.

### System

Ubuntu 26.04 / AMD / Bash, SQLite, Python, PyYAML, C (not C++), GCC, NUMA, numactl

### GPU

Ubuntu 26.04 / AMD / Bash, SQLite, Python, PyYAML, C (not C++), GCC, NUMA, numactl

## 5. Software Requirements

| Component | Version |
|---|---|
| OS | Ubuntu 26.04 |
| Kernel | kernel 7.0.0 |
| Python | Python 3.14.4 |
| ROCm | N/A - ROCm not used |
| rocBLAS | N/A - rocBLAS not used |

Compiles src/multichase.c and runs one pointer-chase per yaml working_set_size tier under numactl via scripts/collect_multichase.py. working_set_size expands from yaml. stride, warmup_iters, and num_iterations use collect_multichase.py defaults (64, 10000, 200000) because collect_workload.py does not forward yaml or run_benchmark values.

## 6. Installation

## 7. Running the Benchmark

```bash
Compile src/multichase.c; run build/multichase under numactl via scripts/collect_multichase.py
```

**Validating results separately:**

```bash
export BENCHMARK_PYTHON=/usr/bin/python3.13  # optional
python3 -m venv .venv
source ".venv/bin/activate"
".venv/bin/python" scripts/validate_results.py
```

## 8. Output

### `results/benchmark.db` (SQLite)

CSV with one row per cache or DRAM tier

sample_index,status,tier,latency_ns,working_set_size,stride,thread_count,error_message
0,ok,l1,1.2,32768,64,1,

```bash
Compile src/multichase.c; run build/multichase under numactl via scripts/collect_multichase.py
```

### `results/summary.json`

Consolidated metrics from the most recent run — suitable for CI artifact upload or dashboard ingestion.

### `results/raw/<timestamp>.txt`

CSV with one row per cache or DRAM tier

sample_index,status,tier,latency_ns,working_set_size,stride,thread_count,error_message
0,ok,l1,1.2,32768,64,1,

## 9. Baselines / Thresholds

Expected ranges and gates live in `config/benchmark_config.yaml` under `baselines:` or `thresholds:`. To update them, edit that file — never edit validation code directly.

## 10. Troubleshooting

**`setup.sh` missing collector**
Create cannot finish without `scripts/collect_workload.py`.

**`self_check` overlay rewritten**
Do not overwrite files listed in `results/overlay_lock.json`.

**Remote SSH drop during setup**
Reconnect and resume `bash setup.sh --assume-yes`. Do not wipe `.venv` or `.cache`.

## 11. NVIDIA H100 Coding Differences

Primary target is AMD ROCm. NVIDIA notes in this section are reference only and are not the execution path.

## Repository layout

```text
.
├── setup.sh
├── run_benchmark.sh
├── benchmark_specification.json
├── .github/workflows/      # thin CI callers (see Continuous Integration)
├── config/
├── scripts/
├── src/
├── tests/
├── docs/
├── results/
└── LICENSE
```

## Continuous Integration

| Workflow | Runs on | When | What it does |
|---|---|---|---|
| [CI](.github/workflows/ci.yml) | GitHub-hosted runner | every pull request, and every push to `main` | shellcheck, ruff, `bash -n`, `compileall`, `run_benchmark.sh --help`, specification schema, the results validator on a seeded fixture, required files, and actionlint. No GPU and no benchmark run. |
| [GPU Smoke Benchmark](.github/workflows/gpu-smoke.yml) | self-hosted runner labeled `gpu`, `amd`, `ubu2604` | only when started by hand: **Actions → GPU Smoke Benchmark → Run workflow** (choose `smoke`, `baseline` or `extended`) | Verifies the pre-provisioned GPU stack, records `results/environment.json` (driver, runtime, kernel, GPU), runs the profile with `--validate`, shows headline metrics on the run page, and uploads the results. |

Both files are short callers. The steps themselves live once, for every workload in the suite, in [`garys-gpu-benchmarks/shared-workflows`](https://github.com/garys-gpu-benchmarks/shared-workflows), pinned at `@v1`. The GPU workflow is never triggered by pull requests, so code from a fork cannot run on the GPU host.

### Running it as part of the AMD Ubuntu 26.04 bundle

This repository is one of the 32 workloads in [`bundle-amd-ubuntu-2604`](https://github.com/garys-gpu-benchmarks/bundle-amd-ubuntu-2604), which holds them as git submodules. To put the whole bundle on a GPU host and run this workload from it:

```bash
git clone --recurse-submodules https://github.com/garys-gpu-benchmarks/bundle-amd-ubuntu-2604 /opt/benchmarks
cd /opt/benchmarks/309-sys-bench-amd-multichase-numa-latency-ubu2604
bash run_benchmark.sh --profile smoke --validate
```
