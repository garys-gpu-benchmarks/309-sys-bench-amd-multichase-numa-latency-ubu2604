# SPEC.md: "The Human How"; Exact technical requirements, environment setup, implementation details, etc.

## Execution Description

## Parameters

| Parameter | CLI Flag | Tested Values | Default | Description |
| --- | --- | --- | --- | --- |
| num_numa_nodes | `--num-numa-nodes` | smoke=1, baseline=1, extended=1 | 1 | From Parameter list; see Execution Description With Parameters. |
| num_threads | `--num-threads` | smoke=1, baseline=4, extended=4 | 4 | From Parameter list; see Execution Description With Parameters. |
| thread_affinity | `--thread-affinity` | smoke=0-3, baseline=0-3, extended=0-3 | 0-3 | From Parameter list; see Execution Description With Parameters. |
| page_size | `--page-size` | smoke=4KB, baseline=4KB, extended=4KB | 4KB | From Parameter list; see Execution Description With Parameters. |
| working_set_size | `--working-set-size` | smoke=l1=16384,l2=262144,l3=2097152,local_dram=16777216,remote_dram=16777216, baseline=l1=32768,l2=262144,l3=8388608,local_dram=268435456,remote_dram=268435456, extended=l1=32768,l2=262144,l3=8388608,local_dram=268435456,remote_dram=268435456 | l1=32768,l2=262144,l3=8388608,local_dram=268435456,remote_dram=268435456 | From Parameter list; see Execution Description With Parameters. |
| stride | `--stride` | smoke=64, baseline=64, extended=64 | 64 | From Parameter list; see Execution Description With Parameters. |
| access_pattern | `--access-pattern` | smoke=random, baseline=random, extended=random | random | From Parameter list; see Execution Description With Parameters. |
| prefetch_enabled | `--prefetch-enabled` | smoke=false, baseline=false, extended=false | false | From Parameter list; see Execution Description With Parameters. |
| warmup_iters | `--warmup-iters` | smoke=10000, baseline=1000000, extended=1000000 | 1000000 | From Parameter list; see Execution Description With Parameters. |
| num_iterations | `--num-iterations` | smoke=200000, baseline=2900000000, extended=8800000000 | 2900000000 | From Parameter list; see Execution Description With Parameters. |

## Invocation

```bash
Compile src/multichase.c; run build/multichase under numactl via scripts/collect_multichase.py
```

## Raw Output Format

CSV with one row per cache or DRAM tier

sample_index,status,tier,latency_ns,working_set_size,stride,thread_count,error_message
0,ok,l1,1.2,32768,64,1,

## Metrics

- **#1: Pointer-chase latency for the L1-cache tier** — stored as `latency_ns_l1`.
- **#2: Pointer-chase latency for the L2-cache tier** — stored as `latency_ns_l2`.
- **#3: Pointer-chase latency for the L3-cache tier** — stored as `latency_ns_l3`.
- **#4: Local DRAM latency, ns** — stored as `latency_ns_local_dram`.
- **#5: Remote DRAM latency, ns** — stored as `latency_ns_remote_dram`.

## Framework

Compiles src/multichase.c and runs one pointer-chase per yaml working_set_size tier under numactl via scripts/collect_multichase.py. working_set_size expands from yaml. stride, warmup_iters, and num_iterations use collect_multichase.py defaults (64, 10000, 200000) because collect_workload.py does not forward yaml or run_benchmark values.

## Installation and Execution Summary

Compile src/multichase.c and run one pointer-chase per yaml working_set_size under numactl --cpunodebind=0 --membind=0 (remote_dram uses --membind=1 when two or more NUMA nodes exist; otherwise not measured), then parse latency_ns, to measure cache and DRAM chase latency. yaml stride, warmup, and iterations are not forwarded

## Platform Portability

- **AMD (primary):** ```bash
Compile src/multichase.c; run build/multichase under numactl via scripts/collect_multichase.py
```
- **NVIDIA:** Primary target is AMD ROCm. NVIDIA notes in this section are reference only and are not the execution path.

## Model Context Protocols

- **Active:** None

## Execution-Loop Validation Contract

EXECUTION CHAIN: `run_benchmark.sh` ➔ raw output ➔ `scripts/parse_results.py` ➔ `results/benchmark.db` ➔ `scripts/validate_results.py`

This benchmark uses a lightweight, SQLite-integrated execution loop for result validation. All validation is performed by `scripts/validate_results.py`.

### Validation script usage

```bash
export BENCHMARK_PYTHON=/usr/bin/python3.13  # optional; select the installed interpreter

# After a live run:
".venv/bin/python" scripts/validate_results.py --db results/benchmark.db

# CI / no-GPU path (seeds fixture and validates it):
".venv/bin/python" scripts/validate_results.py --seed-fixture --quiet

# Override DB path via environment variable:
BENCHMARK_DB=tests/fixtures/benchmark.db \
  ".venv/bin/python" scripts/validate_results.py
```

### Run artifact contract

CSV with one row per cache or DRAM tier

sample_index,status,tier,latency_ns,working_set_size,stride,thread_count,error_message
0,ok,l1,1.2,32768,64,1,

```bash
bash run_benchmark.sh --help
bash run_benchmark.sh --profile smoke --validate
bash run_benchmark.sh --profile baseline --validate
bash run_benchmark.sh --profile extended --validate
```
`run_benchmark.sh --help` prints usage and exits. The harness calls `scripts/ensure_setup.sh` when `.setup_state` is absent.

### Required integrity checks (built into `validate_results.py`)

1. Latest run exists and `runs.status = 'ok'`.
2. `run.error_message` is NULL.
3. `started_at` and `finished_at` are valid ISO-8601 UTC strings.
4. All required aggregate metrics in `runs` are non-NULL and finite.
5. All required aggregate metrics are physically sensible (positive values). Compiles src/multichase.c and runs one pointer-chase per yaml working_set_size tier under numactl via scripts/collect_multichase.py. working_set_size expands from yaml. stride, warmup_iters, and num_iterations use collect_multichase.py defaults (64, 10000, 200000) because collect_workload.py does not forward yaml or run_benchmark values.
6. At least 2 sample rows exist for the latest `run_id` (sweep coverage).
7. No sample has `status = 'error'`.
8. Compiles src/multichase.c and runs one pointer-chase per yaml working_set_size tier under numactl via scripts/collect_multichase.py. working_set_size expands from yaml. stride, warmup_iters, and num_iterations use collect_multichase.py defaults (64, 10000, 200000) because collect_workload.py does not forward yaml or run_benchmark values.

### Baseline / Threshold configuration (`config/benchmark_config.yaml`)

Expected ranges and gates live in `config/benchmark_config.yaml` under `baselines:` or `thresholds:`. To update them, edit that file — never edit validation code directly.

Threshold key suffixes encode comparison direction when `thresholds:` is present: `_min` → observed value must be ≥ threshold. `_max` → observed value must be ≤ threshold. Informational `baselines:` ranges are not pass/fail gates.
