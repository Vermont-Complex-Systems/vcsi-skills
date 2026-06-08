---
name: svelte-d3-charting
description: Best practices for building declaratively data visualizations with Svelte 5 and D3. Use this skill whenever creating, editing, or reviewing chart/plot components that combine D3 with Svelte.
---

# Charting with Svelte + D3

Use **Svelte for DOM rendering** and **D3 for math/scales** — never D3's imperative DOM manipulation.

## Companion skills and tools

This skill focuses on D3+Svelte integration patterns. For general Svelte guidance (reactivity, events, styling), it works best alongside:

- **svelte-core-bestpractices** skill — if available, use it for `$state`, `$derived`, `$effect`, event handling, and component patterns. If not installed, suggest the user install it.
- **Svelte MCP server** (`plugin:svelte:svelte`) — if available, use `list-sections` and `get-documentation` to look up current Svelte docs on demand. Prefer this over guessing at Svelte APIs.

If neither is available, mention it to the user with a reference to https://svelte.dev/docs/ai/overview.

## Core Principle: Svelte Renders, D3 Computes

Use D3 for scales, extents, tick generation, path generators, color schemes, and data transforms. Use Svelte's template syntax (`{#each}`, `{#if}`, attribute bindings) to render SVG/HTML elements directly.

```svelte
<script>
  import { scaleLinear, scaleBand, scaleOrdinal, max } from 'd3';

  const data = [
    { label: 'A', value: 30 },
    { label: 'B', value: 80 },
    { label: 'C', value: 45 },
  ];

  let width = $state(800);
  const height = 400;
  const margin = { top: 20, right: 20, bottom: 40, left: 50 };

  let innerWidth = $derived(width - margin.left - margin.right);
  let innerHeight = $derived(height - margin.top - margin.bottom);

  let xScale = $derived(
    scaleBand()
      .domain(data.map(d => d.label))
      .range([0, innerWidth])
      .padding(0.1)
  );

  let yScale = $derived(
    scaleLinear()
      .domain([0, max(data, d => d.value)])
      .range([innerHeight, 0])
      .nice()
  );

  const colorScale = scaleOrdinal()
    .domain(data.map(d => d.label))
    .range(["#1f77b4", "#ff7f0e", "#2ca02c"]);

  let yTicks = $derived(yScale.ticks(5));
</script>

<div class="chart-container" bind:clientWidth={width}>
  <svg viewBox={`0 0 ${width} ${height}`}>
    <g transform={`translate(${margin.left},${margin.top})`}>
      {#each yTicks as tick}
        <line x1={0} x2={innerWidth} y1={yScale(tick)} y2={yScale(tick)} stroke="#e0e0e0" />
      {/each}

      {#each data as d (d.label)}
        <rect
          x={xScale(d.label)}
          y={yScale(d.value)}
          width={xScale.bandwidth()}
          height={innerHeight - yScale(d.value)}
          fill={colorScale(d.label)}
        />
      {/each}

      {#each yTicks as tick}
        <text x={-10} y={yScale(tick)} text-anchor="end" alignment-baseline="middle" font-size="10">
          {tick}
        </text>
      {/each}

      {#each data as d (d.label)}
        <text
          x={xScale(d.label) + xScale.bandwidth() / 2}
          y={innerHeight + 16}
          text-anchor="middle"
          font-size="10"
        >
          {d.label}
        </text>
      {/each}

      <line x1={0} x2={innerWidth} y1={innerHeight} y2={innerHeight} stroke="black" />
      <line x1={0} x2={0} y1={0} y2={innerHeight} stroke="black" />
    </g>
  </svg>
</div>
```

This single example demonstrates the full pattern: reactive dimensions, derived scales, hand-rendered axes, and declarative SVG — all working together.

**Never do this:**
```js
// Don't use D3 to manipulate the DOM
d3.select(svgRef).selectAll('rect').data(data).join('rect')...
```

## Reactive Scales

The core example shows the basic pattern — every D3 scale as `$derived`. Two additional notes:

- When computing extents from filtered or dynamic data requires multiple steps, use `$derived.by()`:

```js
let xExtent = $derived.by(() => {
  const data = filteredData.length > 0 ? filteredData : allData;
  const [min, max] = extent(data, d => d.x);
  const padding = (max - min) * 0.05;
  return [min - padding, max + padding];
});
```

- Never use D3's axis generators (`d3.axisBottom`, etc.) — they imperatively mutate the DOM. Render axes by hand from `scale.ticks()` as shown in the core example. For time axes, use `timeFormat` from `d3-time-format` to format tick labels.

## Path Generators: Lines and Areas

Use D3's path generators (`line`, `area`) for the math, then render the resulting path string in a Svelte `<path>` element. Make generators `$derived` so they update with scale changes. Always use `?? ""` when calling generators, since they return `null` for empty data:

```svelte
<script>
  import { line, area, curveMonotoneX } from 'd3-shape';

  let lineGenerator = $derived(
    line()
      .x(d => xScale(d.label) ?? 0)
      .y(d => yScale(d.value))
      .curve(curveMonotoneX)
  );

  let areaGenerator = $derived(
    area()
      .x(d => xScale(d.label) ?? 0)
      .y0(innerHeight)
      .y1(d => yScale(d.value))
      .curve(curveMonotoneX)
  );

  let linePath = $derived(lineGenerator(data) ?? "");
  let areaPath = $derived(areaGenerator(data) ?? "");
</script>

<!-- Area fill underneath, line on top -->
<path d={areaPath} fill={color} opacity={0.2} />
<path d={linePath} fill="none" stroke={color} stroke-width={2} />

<!-- Data point circles with native tooltips -->
{#each data as d (d.label)}
  <circle cx={xScale(d.label) ?? 0} cy={yScale(d.value)} r={3} fill={color}>
    <title>{d.label}: {d.value}</title>
  </circle>
{/each}
```

The generator and the path it produces are separate derived values — this keeps things composable and debuggable.

## Reference files

- `references/animations.md` — CSS transitions, keyframes, Svelte `Tween` for chart animations.
- `references/tooltips.md` — SVG `<title>`, rich HTML tooltips with viewport-aware positioning, hover state patterns.
