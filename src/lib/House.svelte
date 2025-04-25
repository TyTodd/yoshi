<script>
  import * as d3 from "d3";
  import { fly } from "svelte/transition";
  import { cubicOut } from "svelte/easing";
  import { flip } from "svelte/animate";

  export let data = [];

  const width = 1200;
  const height = 400;
  const margin = { top: 20, right: 20, bottom: 50, left: 60 };
  const innerWidth = width - margin.left - margin.right;
  const innerHeight = height - margin.top - margin.bottom;

  let tooltip = null;
  let hoveredHouse = null;

  $: xScale = d3
    .scaleBand()
    .domain(data.map((d) => d.year))
    .range([0, innerWidth])
    .padding(0)
    .paddingOuter(data.length < 6 ? 0.8 : 0);

  $: yScale = d3
    .scaleLinear()
    .domain([0, d3.max(data, (d) => d.rent) * 1.1])
    .range([innerHeight, 0])
    .nice();

  const houseImages = [
    "images/building1.jpg",
    "images/building2.jpg",
    "images/building3.jpg",
    "images/building4.jpg",
    "images/building4p2.jpg",
    "images/building5.jpg",
    "images/building6.jpg",
    "images/building7.jpg",
    "images/building8.jpg",
  ];

  const selectHouseImage = (rent) => {
    const ranges = [2200, 2400, 2600, 2800, 3000, 3200, 3400, 3600];
    for (let i = 0; i < ranges.length; i++) {
      if (rent < ranges[i]) return houseImages[i];
    }
    return houseImages[8];
  };

  let xAxis, yAxis, gridlines;

  $: if (xAxis && yAxis && gridlines) {
    d3.select(xAxis).call(d3.axisBottom(xScale));
    d3.select(yAxis).call(d3.axisLeft(yScale).tickFormat(d3.format("$.0f")));
    d3.select(gridlines).call(
      d3.axisLeft(yScale).tickSize(-innerWidth).tickFormat("")
    );
  }

  const handleMouseOver = (event, d) => {
    const containerRect =
      event.currentTarget.ownerSVGElement.getBoundingClientRect();
    hoveredHouse = {
      year: d.year,
      rent: d.rent,
      index: d.index,
      YoYc: d.change,
      x: event.clientX - containerRect.left,
      y: event.clientY - containerRect.top,
    };
  };

  const handleMouseOut = () => {
    hoveredHouse = null;
  };
</script>

<h1>need title</h1>
<div class="chart-container">
  <svg {width} {height}>
    <g transform={`translate(${margin.left},${margin.top})`}>
      <g class="gridlines" bind:this={gridlines} />

      <g transform={`translate(0, ${innerHeight})`} bind:this={xAxis} />
      <g bind:this={yAxis} />

      {#each data as d (d.year)}
        <g
          class="house-bar"
          transform={`translate(${xScale(d.year)}, ${yScale(d.rent)})`}
          on:mouseover={(e) => handleMouseOver(e, d)}
          on:mouseout={handleMouseOut}
        >
          <image
            href={selectHouseImage(d.rent)}
            width={Math.min(Math.max(xScale.bandwidth(), 60), 180)}
            height={innerHeight - yScale(d.rent)}
            preserveAspectRatio="none"
            style="cursor: pointer;"
            in:fly={{ x: 200, duration: 600, easing: cubicOut }}
            out:fly={{ x: 200, duration: 600, easing: cubicOut }}
          />
        </g>
        <!-- <g animate:flip>
          <image
            href={selectHouseImage(d.rent)}
            x={xScale(d.year)}
            y={yScale(d.rent)}
            width={Math.min(Math.max(xScale.bandwidth(), 60), 180)}
            height={innerHeight - yScale(d.rent)}
            preserveAspectRatio="none"
            in:fly={{ x: 200, duration: 600, easing: cubicOut }}
            out:fly={{ x: 200, duration: 600, easing: cubicOut }}
            on:mouseover={(e) => handleMouseOver(e, d)}
            on:mouseout={handleMouseOut}
            style="cursor: pointer;"
          />
        </g> -->
      {/each}
    </g>
  </svg>

  {#if hoveredHouse}
    <div class="centered-tooltip-wrapper">
      <div class="tooltip-panel">
        <div class="tooltip-content">
          <!-- <strong>{hoveredHouse.year}</strong> -->
          <div>Year: {hoveredHouse.year}</div>
          <div>Rent: ${hoveredHouse.rent.toFixed(0)}</div>
          <div>Index: {(hoveredHouse.index / 100).toFixed(4)}</div>
          <div>
            YoY: {hoveredHouse.YoYc.toFixed(2)}%
          </div>
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  .chart-container {
    position: relative;
    width: 100%;
    user-select: none;
  }

  .gridlines {
    stroke: #ccc;
    stroke-opacity: 0.3;
  }

  svg {
    overflow: visible;
  }
  .centered-tooltip-wrapper {
    position: absolute;
    top: 100%;
    left: 50%;
    transform: translate(-50%, -50%);
    pointer-events: none;
    z-index: 10;
  }

  .tooltip-panel {
    padding: 16px 28px;
    border-radius: 12px;
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
    font-size: 14px;
  }

  .tooltip-content {
    display: flex;
    gap: 24px;
    flex-wrap: wrap;
    justify-content: center;
  }
  .house-bar {
    transition: transform 0.5s ease;
  }
</style>
