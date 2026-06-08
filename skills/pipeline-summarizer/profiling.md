# Profiling — Resource Consumption

Profiling is hard. Pipelines often have dedicated skills or tooling for
empirical measurement (e.g., a `duckdb-slurm-hpc` skill with experiment
logs, or a polars/pandas benchmarking setup). The pipeline-summarizer
**never guesses** resource usage from code — it only reports empirical data.

## Where to find data

Search in this order — stop at the first hit:

1. **Profiling skills** — look for `.claude/skills/*/experiments/README.md`.
   These are authoritative: they contain production wall times, peak RSS,
   thread counts, and success rates from real runs. Also check
   `.claude/skills/*/experiments/*/SUMMARY.md` for per-experiment detail.
2. **User-provided logs** — the user may point to SLURM logs, CI artifacts,
   or a monitoring dashboard. Ask if nothing is found in step 1.
3. **User estimates from memory** — acceptable, but mark every cell with `†`
   and add: `† User-provided estimate — not empirically measured.`
4. **Nothing found** — mark cells as `*Not profiled*` and note that a
   profiling skill or manual benchmarking is needed.

## The golden rule

**Never estimate resource usage from reading the code.** Pipelines are
complex enough that code-based guesses are unreliable and actively
misleading to anyone using PIPELINE.md as a catalog entry.

## Table format

Split rows by variant when timings diverge across parameter sets:

```markdown
| Step | Variant | Wall Time | Peak RSS |
|---|---|---|---|
| Transform — Stage 1 | small | 20–40s | ~49 GB |
| Transform — Stage 1 | large | 3–5 min | ~85 GB |
```

Always include a `Source:` line citing where the data came from.
