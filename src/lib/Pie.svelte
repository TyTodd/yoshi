<script>
  import { onMount, afterUpdate } from "svelte";
  import * as d3 from "d3";

  export let selectedIndex = -1;
  export let data = [];
  export let year;

  let arcGenerator = d3.arc().innerRadius(0).outerRadius(50);
  let sliceGenerator = d3
    .pie()
    .sort(null)
    .value((d) => d.value);
  let colors = d3.scaleOrdinal(d3.schemeTableau10);
  //   $: colors = d3
  //     .scaleOrdinal()
  //     .domain(data.map((_, i) => i))
  //     .range(d3.quantize(d3.interpolateBlues, data.length));

  // Define arcData and arcs outside the reactive block
  let arcData;
  let arcs;
  $: {
    // Reactively calculate arcData and arcs the same way we did before with sliceGenerator and arcGenerator
    arcData = sliceGenerator(data);
    arcs = arcData.map((d) => arcGenerator(d));
  }

  let showChart = true;
  function toggleView() {
    showChart = !showChart;
    liveText = showChart ? "Pie chart view shown." : "Table view shown.";
  }

  let liveText = "";
  function toggleWedge(index, event) {
    if (!event.key || event.key === "Enter") {
      if (selectedIndex === index) {
        selectedIndex = -1;
        liveText = `No projects selected.`;
      } else {
        selectedIndex = index;
        const d = data[index];
        liveText = `${d.label}: ${d.value} projects selected.`;
      }
    }
  }

  $: description = `A pie chart showing project counts by year. ${data.map((d) => `${d.label}: ${d.value} projects`).join(", ")}.`;
</script>

<div class="container">
  <svg
    viewBox="-50 -50 100 100"
    role="img"
    aria-labelledby="pie-title pie-desc"
  >
    <title id="pie-title">Rates</title>
    <desc id="pie-desc">{description}</desc>
    <circle class="pie-outline" r="50" />
    {#each arcs as arc, index}
      <path
        d={arc}
        fill={colors(index)}
        class:selected={selectedIndex === index}
        on:click={(e) => toggleWedge(index, e)}
        on:keyup={(e) => toggleWedge(index, e)}
        tabindex="0"
        role="button"
      />
    {/each}
  </svg>
  <ul class="legend">
    <p><strong>Current Year: {year}</strong></p>
    {#each data as d, index}
      <li
        style="--color: {colors(
          index
        )}; display: flex; align-items: center; gap: 5px"
        class:selected={selectedIndex === index}
      >
        <span class="swatch"></span>
        {d.label} <em>({d.value.toFixed(4)})</em>
      </li>
    {/each}
  </ul>
  <p aria-live="polite" class="sr-only">{liveText}</p>
</div>

<style>
  .data-table {
    margin-top: 1rem;
    margin-bottom: 1rem;
    border-collapse: collapse;
    width: 100%;
    max-width: 30em;
  }

  .data-table caption {
    font-weight: bold;
    margin-bottom: 0.5em;
    text-align: left;
  }

  .data-table th,
  .data-table td {
    border: 1px solid #ccc;
    padding: 0.5em;
    text-align: left;
  }

  .data-table th {
    background-color: #f0f0f0;
  }
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
  svg:hover path:not(:hover),
  svg:focus-within path:not(:focus-visible) {
    opacity: 50%;
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
  path:hover {
    opacity: 100% !important;
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

  ul:has(.selected) li:not(.selected) {
    color: gray;
  }

  .swatch {
    display: inline-block;
    background-color: var(--color);
    height: 10px;
    width: 10px;
    border-radius: 50%;
  }
  .legend {
    display: grid;
    gap: 10px;
    flex: 1;
    min-width: 250px;
    padding: 10px;
    border: 1px solid black;
  }
  .container {
    display: flex;
    gap: 50px;
    align-items: center;
  }
  .sr-only {
    position: absolute;
    left: -9999px;
    width: 1px;
    height: 1px;
    overflow: hidden;
  }
</style>
