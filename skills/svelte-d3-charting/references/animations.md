# Animations and Transitions

Three techniques for animating chart elements, from simplest to most powerful.

## CSS Transitions (preferred for property changes)

Apply inline `style` with CSS `transition` for smooth interpolation between states. Works well for position, size, opacity, and color — any SVG presentation attribute that changes when data updates.

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

This pattern works because the keyed `{#each}` block preserves DOM elements across data changes — Svelte updates attributes on the same element, and CSS handles the interpolation.

To transition between chart types (e.g., bar to lollipop), render both mark types and crossfade with opacity:

```svelte
{#each currentData as d (d.label)}
  <!-- Bar (fades out) -->
  <rect
    x={xScale(d.label)}
    y={yScale(d.value)}
    width={xScale.bandwidth()}
    height={innerHeight - yScale(d.value)}
    opacity={chartType === 'bar' ? 1 : 0}
    style="transition: x 0.8s ease-in-out, y 0.8s ease-in-out, height 0.8s ease-in-out, opacity 0.8s ease-in-out;"
  />

  <!-- Lollipop stem (fades in) -->
  <line
    x1={xScale(d.label) + xScale.bandwidth() / 2}
    x2={xScale(d.label) + xScale.bandwidth() / 2}
    y1={innerHeight}
    y2={yScale(d.value)}
    stroke="#a6a6a6"
    stroke-width="2"
    opacity={chartType === 'lollipop' ? 1 : 0}
    style="transition: x1 0.8s ease-in-out, x2 0.8s ease-in-out, y2 0.8s ease-in-out, opacity 0.8s ease-in-out;"
  />

  <!-- Lollipop head (fades in) -->
  <circle
    cx={xScale(d.label) + xScale.bandwidth() / 2}
    cy={yScale(d.value)}
    r=5
    fill="#a6a6a6"
    opacity={chartType === 'lollipop' ? 1 : 0}
    style="transition: cx 0.8s ease-in-out, cy 0.8s ease-in-out, opacity 0.8s ease-in-out;"
  />
{/each}
```

Both mark types exist in the DOM at all times. CSS transitions handle the crossfade smoothly because the key `(d.label)` keeps each data point's elements stable.

## CSS Keyframe Animations

For repeating or multi-step motion (bounce, pulse, entrance effects), define `@keyframes` in `<style>` and toggle via a class:

```svelte
<rect
  class:bounce={animate}
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

Stagger entrance animations across data points by varying `animation-delay` with the loop index.

## Svelte `Tween` (for animated scale domains)

When the data range itself changes (filtering, zooming), use `Tween` from `svelte/motion` to smoothly animate scale domain values. This makes axes and grid lines glide instead of jumping:

```js
import { Tween } from 'svelte/motion';
import { cubicOut } from 'svelte/easing';

let xExtent = $derived.by(() => {
  const data = filteredData.length > 0 ? filteredData : allData;
  const [min, max] = extent(data, d => d.x);
  const padding = (max - min) * 0.05;
  return [min - padding, max + padding];
});

const xMin = Tween.of(() => xExtent[0], { duration: 600, easing: cubicOut });
const xMax = Tween.of(() => xExtent[1], { duration: 600, easing: cubicOut });

let xScale = $derived(
  scaleLinear()
    .domain([xMin.current, xMax.current])
    .range([margin.left, width - margin.right])
);
```

`Tween.of()` tracks its source reactively — when `xExtent` changes, the tween animates from the old value to the new one. The scale reads `.current` each frame, so everything downstream (axes, grid lines, data marks) updates smoothly.

This combines well with CSS transitions: Tween animates the scale domain (axes glide), while CSS transitions animate individual element positions within the new scale.

## When to Use Which

| Technique | Use for | Example |
|-----------|---------|---------|
| CSS `transition` | Property interpolation between states | Bar positions, opacity fades, chart type crossfade |
| CSS `@keyframes` | Repeating or multi-step animations | Bounce, pulse, staggered entrance |
| Svelte `Tween` | Smoothly animating scale domain values | Axis range changes after filtering, zoom transitions |
