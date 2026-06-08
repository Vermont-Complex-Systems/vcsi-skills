# DAG — Dependency Graph + Schemas

This section produces the step-by-step narrative and the two Mermaid diagrams. An orchestration file, such as a Makefile, Snakefile, or tasks defined by orchestration tools such as Dagster or Airflow,  is expected to facilitate graph traversal.

## Missing orchestration file

Warn the user of the situation and create a Makefile in root using [this Makefile example](makefile-example.md).

## Steps narrative

- One `## Step N — <Name>` heading per pipeline step, in execution order.
- For each step include: script, rule/job name, inputs, outputs, filters,
  key parameters, notes.
- For >12 steps, group under `##` stage headings with `###` sub-steps.
- When the same step runs with different parameter sets (e.g., different
  input sizes or modes), call out the differences explicitly.

## Pipeline Overview Diagram

Mermaid `graph LR` at the end of PIPELINE.md.

- Inputs on the left, final outputs / API on the right.
- One node per step, edges labeled with data format.
- Keep under 15 nodes — use stage grouping for large pipelines.

## Dataset Schema Diagram

Mermaid `erDiagram` showing final output datasets only (not scratch).

- One entity block per registered dataset.
- Columns with SQL types and short descriptions.
- Mark Hive partition keys in the description.
- Relationship lines with join keys on the label.
- Include an `entities` lookup table if the pipeline uses entity mappings.
- Extract schemas from: adapter/registration code, Parquet metadata, or
  output schema sections in pipeline scripts.
