---
name: svelte-d3-charting
description: Best practices for building data visualizations with Svelte 5 and D3. Load this skill when creating, editing, or reviewing chart/plot components that use D3 scales, axes, or data transforms inside Svelte templates.
---

# Charting with Svelte + D3

Use **Svelte for DOM rendering** and **D3 for math/scales** — never D3's imperative DOM manipulation.

> For general Svelte patterns (reactivity, events, styling), see the **svelte-core-bestpractices** skill.

## Core Principle: Svelte Renders, D3 Computes

Use D3 for scales, extents, tick generation, color schemes, and data transforms. Use Svelte's template syntax (`{#each}`, `{#if}`, attribute bindings) to render SVG/HTML elements directly.

```svelte
<script>
  import { scaleLinear, scaleBand, max } from 'd3';

  let { scrollyIndex } = $props();

  let width = $state(800);
  const height = 400;
  const margin = { top: 20, right: 20, bottom: 80, left: 60 };

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
</script>

<div class="chart-container" bind:clientWidth={width}>
  <svg viewBox={`0 0 ${width} ${height}`}>
    <g transform={`translate(${margin.left},${margin.top})`}>
      {#each data as d (d.label)}
        <rect
          x={xScale(d.label)}
          y={yScale(d.value)}
          width={xScale.bandwidth()}
          height={innerHeight - yScale(d.value)}
          fill="#a6a6a6"
        />
      {/each}
    </g>
  </svg>
</div>
```

**Never do this:**
```js
// Don't use D3 to manipulate the DOM
d3.select(svgRef).selectAll('rect').data(data).join('rect')...
```

## Responsive Sizing

Bind `clientWidth` on the chart container to get a reactive width. Derive inner dimensions from it. Use `viewBox` on the SVG so it scales naturally.

```svelte
<script>
  let width = $state(800);
  const height = 400;
  const margin = { top: 20, right: 20, bottom: 60, left: 60 };

  let innerWidth = $derived(width - margin.left - margin.right);
  let innerHeight = $derived(height - margin.top - margin.bottom);
</script>

<div class="chart-container" bind:clientWidth={width}>
  <svg viewBox={`0 0 ${width} ${height}`}>
    <g transform={`translate(${margin.left},${margin.top})`}>
      <!-- chart content -->
    </g>
  </svg>
</div>

<style>
  .chart-container {
    width: 100%;
    height: auto;
  }

  svg {
    width: 100%;
    height: auto;
  }
</style>
```

Because `width` is `$state` and scales are `$derived`, resizing the window automatically recomputes the chart — no manual resize listeners needed.

For mobile-responsive margins or tick counts, use `$derived`:

```js
let isMobile = $derived(width < 768);

let margin = $derived(isMobile
  ? { top: 40, right: 20, bottom: 50, left: 50 }
  : { top: 60, right: 40, bottom: 70, left: 70 }
);

let xTicks = $derived(isMobile ? [0, 0.5, 1.0] : [0, 0.2, 0.4, 0.6, 0.8, 1.0]);
```

## Reactive Scales

Scales depend on data and dimensions — both can change. Make every scale `$derived` so it recomputes automatically:

```js
let xScale = $derived(
  scaleBand()
    .domain(currentData.map(d => d.label))
    .range([0, innerWidth])
    .padding(0.1)
);

let yScale = $derived(
  scaleLinear()
    .domain([0, max(currentData, d => d.value)])
    .range([innerHeight, 0])
    .nice()
);
```

When data changes with the scroll step, scales recompute and the template re-renders — no imperative update logic needed.

## Scroll-Driven Data

Use `$derived` or `$derived.by` to select/transform data based on `scrollyIndex`:

```js
let currentData = $derived.by(() => {
  if (scrollyIndex >= 1) {
    return data.sort((a, b) => b.value - a.value);
  }
  return shuffle(data);
});

let chartType = $derived(scrollyIndex >= 2 ? 'lollipop' : 'bar');
```

## Animations and Transitions

### CSS Transitions (preferred for simple property changes)

Use inline `style` with CSS `transition` for smooth interpolation between states. This is the simplest approach and works well for position, size, opacity, and color changes:

```svelte
{#each currentData as d (d.label)}
  <rect
    x={xScale(d.label)}
    y={yScale(d.value)}
    width={xScale.bandwidth()}
    height={innerHeight - yScale(d.value)}
    fill="#a6a6a6"
    opacity={chartType === 'bar' ? 1 : 0}
    style="transition: x 0.8s ease-in-out, y 0.8s ease-in-out, height 0.8s ease-in-out, opacity 0.8s ease-in-out;"
  />
{/each}
```

### CSS Keyframe Animations

For repeating or complex motion (bounce, pulse, shake), define `@keyframes` in `<style>` and toggle via a class:

```svelte
<rect
  class:bounce={stepConfig.animate === 'bounce'}
  style="animation-delay: {0.1 * i}s;"
/>

<style>
  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    30% { transform: translateY(-50px); }
    50% { transform: translateY(0); }
    70% { transform: translateY(-10px); }
  }

  .bounce {
    animation: bounce 0.6s ease infinite;
  }
</style>
```

### Svelte `Tween` (for derived numeric values)

When you need to smoothly animate a value that drives a scale (like axis domain extents), use `Tween` from `svelte/motion`. This is useful when the data range itself changes between steps:

```js
import { Tween } from 'svelte/motion';
import { cubicOut } from 'svelte/easing';

let lifeExpExtent = $derived.by(() => {
  const vals = filteredData.map(d => d.life_expectancy);
  return [Math.floor(Math.min(...vals)) - 5, Math.ceil(Math.max(...vals)) + 5];
});

const yMin = Tween.of(() => lifeExpExtent[0], { duration: 800, easing: cubicOut });
const yMax = Tween.of(() => lifeExpExtent[1], { duration: 800, easing: cubicOut });

let yScale = $derived(
  scaleLinear()
    .domain([yMin.current, yMax.current])
    .range([innerHeight, 0])
);
```

### When to Use Which

| Technique | Use for | Example |
|-----------|---------|---------|
| CSS `transition` | Property interpolation between states | Bar positions, opacity fades |
| CSS `@keyframes` | Repeating or multi-step animations | Bounce, pulse, entrance effects |
| Svelte `Tween` | Smoothly animating derived numeric values | Axis domain changes, scale transitions |

## Keyed Each Blocks

Always key `{#each}` blocks with a stable identifier so Svelte can animate individual elements rather than replacing them:

```svelte
{#each currentData as d (d.label)}
  <rect ... style="transition: x 0.8s ease-in-out;" />
{/each}
```

Without the key `(d.label)`, reordering data would destroy and recreate DOM elements, breaking CSS transitions.

## Component Decomposition

For complex charts, extract axes, legends, regression lines, and interactive overlays into child components:

```svelte
<g transform={`translate(${margin.left},${margin.top})`}>
  <XAxis {xScale} {innerWidth} {innerHeight} ticks={xTicks} label={xConfig.label} />
  <YAxis {yScale} {innerWidth} {innerHeight} ticks={yTicks} label="Life Expectancy" />
  <ScatterDots data={filteredData} {xScale} {yScale} {colorScale} bind:hoveredCountry />
</g>
```

Pass scales and dimensions as props. Keep the main component focused on data pipeline and layout.
