# Tooltips and Hover Interactions

Two approaches, from simple to rich. Start with `<title>` and upgrade to HTML tooltips when you need custom styling or multi-line content.

## Simple: SVG `<title>` Elements

For basic hover info, nest a `<title>` inside any SVG element. The browser renders it as a native tooltip on hover — no JavaScript, no positioning logic:

```svelte
{#each data as d (d.label)}
  <circle cx={xScale(d.label)} cy={yScale(d.value)} r={3} fill={color}>
    <title>{d.label}: {d.value}</title>
  </circle>
{/each}
```

Limitations: no styling, single line, appears with a delay. Use this when the tooltip is informational and doesn't need to look polished.

## Rich: HTML Tooltip with Viewport-Aware Positioning

For styled, multi-line tooltips, render an HTML `<div>` with `position: fixed` and track mouse coordinates. Use `{@const}` to flip the tooltip when it would overflow the viewport:

```svelte
<script>
  import { innerWidth as windowWidth, innerHeight as windowHeight } from 'svelte/reactivity/window';

  let hoveredItem = $state(null);
  let tooltipX = $state(0);
  let tooltipY = $state(0);
</script>

<!-- On the data element -->
{#each data as d (d.id)}
  <circle
    cx={xScale(d.x)}
    cy={yScale(d.y)}
    r={5}
    fill={colorScale(d.category)}
    onmouseenter={(e) => {
      hoveredItem = d;
      tooltipX = e.clientX;
      tooltipY = e.clientY;
    }}
    onmousemove={(e) => {
      tooltipX = e.clientX;
      tooltipY = e.clientY;
    }}
    onmouseleave={() => hoveredItem = null}
    style="cursor: pointer;"
  />
{/each}

<!-- Tooltip (outside the SVG) -->
{#if hoveredItem}
  {@const flipX = tooltipX > (windowWidth.current ?? 1000) - 280}
  {@const flipY = tooltipY > (windowHeight.current ?? 800) - 140}
  <div
    class="chart-tooltip"
    style="left: {flipX ? tooltipX - 12 : tooltipX + 12}px;
           top: {flipY ? tooltipY - 12 : tooltipY + 12}px;
           transform: translate({flipX ? '-100%' : '0'}, {flipY ? '-100%' : '0'});"
  >
    <h4>{hoveredItem.label}</h4>
    Value: {hoveredItem.value}
  </div>
{/if}

<style>
  .chart-tooltip {
    position: fixed;
    pointer-events: none;
    background: white;
    border: 1px solid #333;
    padding: 8px 12px;
    font-size: 13px;
    box-shadow: 2px 2px 6px rgba(0,0,0,0.15);
    z-index: 1000;
    white-space: nowrap;
  }
</style>
```

The key details:
- **`position: fixed`** with `e.clientX/clientY` — positions relative to the viewport, so it works regardless of scroll position or SVG transforms.
- **`pointer-events: none`** on the tooltip — prevents it from intercepting mouse events and causing flicker.
- **Viewport flipping** — `{@const flipX/flipY}` checks whether the tooltip would overflow the right/bottom edge and flips it to the other side. The 280/140 thresholds should match your tooltip's max width/height.
- **`onmousemove`** — updates position as the user moves within the element, so the tooltip follows the cursor.

## Highlight-on-Hover Pattern

Use opacity and stroke changes to visually connect the tooltip to its data point. Dim everything except the hovered element:

```svelte
{#each data as d (d.id)}
  <circle
    cx={xScale(d.x)}
    cy={yScale(d.y)}
    r={hoveredItem?.id === d.id ? 7 : 5}
    fill={colorScale(d.category)}
    stroke={hoveredItem?.id === d.id ? '#333' : 'white'}
    stroke-width={hoveredItem?.id === d.id ? 2 : 1}
    opacity={hoveredItem && hoveredItem.id !== d.id ? 0.2 : 0.8}
    onmouseenter={() => hoveredItem = d}
    onmouseleave={() => hoveredItem = null}
    style="transition: opacity 0.15s, r 0.15s;"
  />
{/each}
```

For line charts, apply the same opacity pattern to paths:

```svelte
<path
  d={series.path}
  stroke={series.color}
  stroke-width={activeItem === series.id ? 4 : 2}
  stroke-opacity={activeItem && activeItem !== series.id ? 0.2 : 1}
  style="pointer-events: none; transition: stroke-width 0.15s, stroke-opacity 0.15s;"
/>
```

## Shared Hover State Across Components

When hover needs to flow between parent and child (e.g., hovering a legend item highlights chart marks), use `$bindable()`:

```svelte
<!-- Parent -->
<ChartMarks data={filteredData} {xScale} {yScale} bind:hoveredItem />
<Legend {categories} {colorScale} bind:hoveredItem />
<ChartTooltip data={hoveredItem} />

<!-- Child (ChartMarks.svelte) -->
<script>
  let { data, xScale, yScale, hoveredItem = $bindable() } = $props();
</script>
```

This lets hover state in one component automatically update the other — the legend can highlight its entry when you hover a dot, and vice versa.
