---
name: tune-duckdb-load
description: Hypothesis-driven tuning of DuckDB batch workloads on SLURM. Use this skill when tuning DuckDB pipelines, investigating OOM or batch failures, profiling query performance, or sizing SLURM resource requests for DuckDB jobs.
---

# Tuning DuckDB Workloads on SLURM

The agent acts as an experimentalist: hypothesize, measure, evaluate, iterate.

## Companion skills and tools

- **duckdb-docs** skill — if available, use it to look up DuckDB configuration options, SQL syntax, and function behavior on demand. If not installed, suggest the user install it from the official DuckDB skills repo.

If duckdb-docs is not available, mention it to the user with a reference to https://duckdb.org/docs.

## Method

**1. Baseline** — Measure before optimizing. Capture: peak RSS (`sacct`), wall time per stage (`.timer on`), query plan (`EXPLAIN ANALYZE`), data profile (row counts, cardinalities, file sizes). Single job first, then pilot batch (10-20 jobs) for concurrency baseline.

**2. Scope** — Within-job (OOM, slow query) or across-job (lock contention, quota, I/O errors)? This determines whether to test with a single `sbatch` or a pilot batch.

**3. Hypothesize** — One variable at a time. State a concrete prediction ("peak RSS ~80GB"), assumptions, and what would falsify it. Record BEFORE running.

**4. Experiment** — Single job → pilot batch → full batch.

**5. Evaluate** — Compare measured vs predicted. When wrong, don't guess why — profile. Run `EXPLAIN ANALYZE` to see actual CPU time per operator, check `sacct` for real peak RSS, inspect row counts at each stage. Narrative explanations ("it's probably the JOIN") are worthless without profiling data to back them up.

**6. Iterate** — Stop when requirements are met, not when out of ideas.

## DuckDB Memory Model

### Within-job

These determine peak memory for a single DuckDB process:

- **GROUP BY**: cardinality x ~48 bytes = hash table size. High-cardinality grouping keys (e.g., 2-grams vs 1-grams) can explode memory.
- **Threads**: halving threads ~halves hash table memory (DuckDB uses partitioned hash tables). Always use `SET threads = ${SLURM_CPUS_PER_TASK}`, never bare `$(nproc)` which can return 1 in SLURM.
- **Non-spillable aggregates**: `list()`, `string_agg()`, and `PIVOT` cannot spill to disk. These will OOM on large inputs regardless of `memory_limit`.
- **HAVING**: post-aggregation — doesn't reduce peak memory of the GROUP BY.
- **Pre-materialization**: filtering a large file into a smaller intermediate parquet beats scanning the full file N times.
- **memory_limit**: set to SLURM allocation minus ~20GB headroom for OS and DuckDB overhead.
- **Spill directory**: always point `temp_directory` to scratch/tmp, not home directory. Concurrent jobs spilling to home can exhaust quotas.

### Across-job

These matter when running many SLURM jobs concurrently:

- **Shared output files**: concurrent writes need serialization (`flock`) or per-job isolation. Without it, you get lock contention or corrupt output.
- **Filesystem I/O**: concurrent reads of large parquet files from shared filesystems (GPFS, Lustre) can produce transient corruption. Usually retry-safe.
- **Spill quota**: many jobs spilling simultaneously can exhaust filesystem quotas even when individual jobs are within limits.
- **Pilot batches**: always run 10-20 jobs before full batch submission, especially when jobs share filesystem resources or output paths.

## Resource-Aware Sizing

Before choosing job sizes, profile the cluster's node pool from a login node. The distribution of available memory per node determines whether to favor fewer large jobs or many small ones:

```bash
# 1. Discover available partitions
sinfo -s -o "%P %l %F %m"

# 2. Inspect node specs in a partition (memory in MB, CPUs, state)
sinfo -p <partition> -N -o "%N %c %m %G %t"
```

After discovering the cluster specs, **save them to memory** so you can reference them in future tuning decisions without re-running `sinfo`. Include: node classes, cores/node, RAM, GPU info, and how many nodes are in each class. Note which memory thresholds shrink the eligible node pool significantly.

The key tradeoff: requesting more memory per job restricts which nodes are eligible, increasing queue wait time. If the cluster has few high-memory nodes, prefer pipelines that fan out into many smaller jobs over pipelines that require large single-job allocations. This is where DuckDB's thread-memory tradeoff matters — a job requesting 64GB with 8 threads may fit on most nodes, while the same query requesting 256GB with 32 threads can only run on a handful.
