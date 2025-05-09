<script>
  import { onMount, afterUpdate } from "svelte";
  import * as d3 from "d3";

  export let selectedIndex = -1;
  export let data = [];
  export let year;

  let arcGenerator = d3.arc().innerRadius(0).outerRadius(70);
  let sliceGenerator = d3
    .pie()
    .sort(null)
    .value((d) => d.value);
  // let colors = d3.scaleOrdinal(d3.schemeTableau10);
  let colors = d3.scaleOrdinal([
    "#d5912f", // Corporate Ownership
    "#dd8d97", // Other 1
    "#da8e71", // Other 2
  ]);

  // Define arcData and arcs outside the reactive block
  let arcData;
  let arcs;
  $: {
    // Reactively calculate arcData and arcs the same way we did before with sliceGenerator and arcGenerator
    arcData = sliceGenerator(data);
    arcs = arcData.map((d) => arcGenerator(d));
  }

  $: description = `A pie chart showing project counts by year. ${data.map((d) => `${d.label}: ${d.value} projects`).join(", ")}.`;
  function polarToCartesian(angle, radius) {
    return [
      Math.cos(angle - Math.PI / 2) * radius,
      Math.sin(angle - Math.PI / 2) * radius,
    ];
  }
  $: totalValue = data.reduce((acc, d) => acc + d.value, 0);
  $: dataWithPercentages = data.map((d) => ({
    ...d,
    percentage: ((d.value / totalValue) * 100).toFixed(1),
  }));
</script>

<div class="container">
  <svg
    viewBox="-50 -50 100 100"
    preserveAspectRatio="xMidYMid meet"
    role="img"
    aria-labelledby="pie-title pie-desc"
  >
    <title id="pie-title">Rates</title>
    <desc id="pie-desc">{description}</desc>

    <!-- Year label stays up here -->
    <text
      x="0"
      y="-40"
      text-anchor="middle"
      dominant-baseline="middle"
      font-size="15"
      font-weight="bold"
      fill="black"
    >
      Year: {year}
    </text>

    <!-- Move all pie elements down -->
    <g transform="translate(0, 70)">
      <circle class="pie-outline" r="70" />
      {#each arcData as d, index}
        {@const angle = (d.startAngle + d.endAngle) / 2}
        {@const arcSize = d.endAngle - d.startAngle}
        {@const radius = arcSize < 1.2 ? 85 : 37}
        {@const [x, y] = polarToCartesian(angle, radius)}
        <path
          d={arcGenerator(d)}
          fill={colors(index)}
          stroke="white"
          stroke-width="2"
          class:selected={selectedIndex === index}
          tabindex="0"
          role="button"
        />
        <text
          {x}
          {y}
          text-anchor="middle"
          dominant-baseline="middle"
          font-size="9"
          fill="black"
          style="pointer-events: none;"
        >
          <tspan {x} dy="-0.3em">{dataWithPercentages[index].label}</tspan>
          <tspan {x} dy="1.2em"
            >({dataWithPercentages[index].percentage}%)</tspan
          >
        </text>
      {/each}
    </g>
  </svg>
</div>

<style>
  .pie-outline {
    stroke: black;
    fill: none;
    stroke-width: 1;
  }

  svg {
    max-width: 20em;
    margin-block: 2em;

    /* Do not clip shapes outside the viewBox */
    overflow: visible;
  }
  path:focus-visible {
    opacity: 100% !important;
    stroke: white;
    stroke-width: 2px;
    stroke-dasharray: 4; /* Adjust the dash length as needed */
  }
  path {
    transition: 300ms;
    cursor: pointer;
    outline: none;
  }
  .selected {
    --color: oklch(60% 45% 0) !important;

    &:is(path) {
      fill: var(--color) !important;
    }

    &:is(li) {
      color: var(--color);
    }
  }
  .container {
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 2em;
  }
</style>
