---
name: pipeline-summarizer
description: "Produces a structured PIPELINE.md documenting a data pipeline end-to-end: steps, schemas, parameters, diagrams. Triggers on document pipeline, summarize pipeline, write PIPELINE.md, or any Makefile/Snakemake/Dagster documentation request."
---

## Overview

This guide covers essential steps to summarize data pipelines as `PIPELINE.md` to the repo root. The agent assumes the existence of a orchestration file, such as a `Snakefile`, `Makefile`, or other orchestration tools like Dagster or Airflow, from which a Directed Acyclic Graph (DAG) can be traversed (see [./DAG.md](DAG.md)). After having traversed the DAG, each step is profiled to assess resource consumptions and time (see [./profiling.md](profiling.md)).

## Quick Start

1. **Find the entry point** — Snakefile, Makefile, or Dagster definitions.
2. **Traverse the dependency graph** — orchestration → scripts → configs.
   Only files reachable from the entry point. For how to document each step and draw the Mermaid diagrams, see [./DAG.md](DAG.md).
3. **Find profiling data** — resource consumption must come from empirical
   measurements, never estimated from code. See [./profiling.md](profiling.md)
   for where to search.
4. **Write PIPELINE.md** — see [./examples.md](examples.md) for a worked example.


## Gotchas

- **Never omit a section** — write `*Not available — <reason>*` instead.
- **Steps are `##` headings in execution order.** The heading sequence IS
  the narrative.
- When a step runs with different parameter sets, document each variant.
- Use `--shallow` for one-paragraph-per-step summaries.
