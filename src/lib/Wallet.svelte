<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  // Removed direct import since Vite cannot import CSVs from /public; data will be loaded dynamically
  const slant = -41;
  const barWidth = 106;
  let svg;
  export let selectedYear = 2008; // single selected year filter
  let mergedData = [];
  let incomeRentData = [];
  $: filteredData = mergedData.filter((d) => d.year === selectedYear); // reactive filtered data
  // Add new variables for the ratio chart
  let ratioSvg;
  const ratioMargin = { top: 10, right: 10, bottom: 20, left: 30 };
  const ratioWidth = 300 - ratioMargin.left - ratioMargin.right;
  const ratioHeight = 200 - ratioMargin.top - ratioMargin.bottom;

  // Load CSV data dynamically
  onMount(async () => {
    console.log("Loading CSV data...");
    // const raw = await d3.csv("/income_and_rent.csv", d3.autoType);
    const raw = await d3.csv("income_and_rent.csv", d3.autoType);
    console.log("Raw data loaded:", raw);
    // Multiply rent by 12 for annual rent
    raw.forEach((d) => (d.rent = d.rent * 12));
    mergedData = raw;
  });

  // Function to update the chart based on the selected year
  $: if (filteredData.length && svg) {
    // console.log("Updating chart for selected year:", selectedYear);
    const data = filteredData;
    // console.log("Filtered data:", data);

    // Set up chart dimensions and margins
    const margin = { top: 0, right: 30, bottom: 48, left: 100 },
      height = 200 - margin.top - margin.bottom, // reduced from 800 to 400
      width = 120; // keeping the same width for thick bars

    // Clear previous chart elements
    d3.select(svg).selectAll("*").remove();
    // console.log("Cleared previous SVG contents");

    // Define y scale for stacked bar values (income + rent)
    const y = d3
      .scaleLinear()
      .domain([0, d3.max(data, (d) => d.income + d.rent)])
      .range([height, 0]); // reversed range for proper orientation

    // Append group to hold chart content
    const g = d3
      .select(svg)
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);
    // console.log("Appended chart group element");

    // Draw rent portion at the bottom
    g.selectAll(".rent")
      .data(data)
      .enter()
      .append("path")
      .attr("class", "rent")
      .attr("d", (d) => {
        const xPos = 0;
        const yTop = y(d.rent);
        const yBottom = height;
        const slant = -41;
        const barRight = xPos + barWidth;
        const radius = 6;

        const barHeight = yBottom - yTop;
        const r = Math.min(radius, Math.abs(barHeight / 2));

        const topLeftY = yTop;
        const topRightY = yTop + slant;
        const bottomLeftY = yBottom;
        const bottomRightY = yBottom + slant + 1;

        return `
      M${xPos + r},${bottomLeftY}
      Q${xPos},${bottomLeftY} ${xPos},${bottomLeftY - r}
      L${xPos},${topLeftY}
      L${barRight},${topRightY}
      L${barRight},${bottomRightY - r}
      Q${barRight},${bottomRightY} ${barRight - r},${bottomRightY + 2}
      Z
    `;
      })
      .attr("fill", "#ff7f0e");

    // Draw income portion on top
    g.selectAll(".income")
      .data(data)
      .enter()
      .append("path")
      .attr("class", "income")
      .attr("d", (d) => {
        const xPos = 0;
        const yBottom = y(d.rent);
        const yTop = y(d.rent + d.income);
        const slant = -41;
        const barHeight = yBottom - yTop;

        const radius = 6;
        const r = Math.min(radius, Math.abs(barHeight / 2));
        const rightR = -2;
        const barRight = xPos + barWidth;

        // Start both top corners at the same height
        const leftTopY = yTop - slant - 13;
        const rightTopY = yTop + 3;

        return `
    M${xPos},${yBottom}
    L${xPos},${leftTopY + r}
    Q${xPos},${leftTopY} ${xPos + r},${leftTopY}
    L${barRight - r},${rightTopY}
    Q${barRight},${rightTopY} ${barRight},${rightTopY + rightR}
    L${barRight},${yBottom + slant}
    Z
  `;
      })
      .attr("fill", "#69b3a2");

    // Update labels with fixed x position
    g.selectAll(".income-label")
      .data(data)
      .enter()
      .append("text")
      .attr("class", "income-label")
      .attr("x", barWidth / 2 - 15) // fixed position
      .attr("y", (d) => y(d.rent + d.income / 2) + 20)
      .attr("text-anchor", "middle")
      .attr("transform", `rotate(-15)`)
      .text((d) => `$${Math.round(d.income).toLocaleString()}`)
      .attr("fill", "white");

    // Update rent label with fixed x position
    g.selectAll(".rent-label")
      .data(data)
      .enter()
      .append("text")
      .attr("class", "rent-label")
      .attr("x", barWidth / 2 - 30) // fixed position
      .attr("y", (d) => y(d.rent / 2) - 5)
      .attr("text-anchor", "middle")
      .attr("transform", `rotate(-19)`)
      .text((d) => `$${Math.round(d.rent).toLocaleString()}`)
      .attr("fill", "white");

    // Update ratio label with fixed x position
    // g.selectAll(".ratio-label")
    //   .data(data)
    //   .enter()
    //   .append("text")
    //   .attr("class", "ratio-label")
    //   .attr("x", barWidth + 20) // fixed position
    //   .attr("y", (d) => y(d.rent + d.income / 2) + slant / 2)
    //   .attr("text-anchor", "start")
    //   .text((d) => `${((d.rent / d.income) * 100).toFixed(1)}%`)
    //   .attr("fill", "black");
  }

  // Create reactive statement for ratio chart
  $: if (mergedData.length && ratioSvg) {
    // Clear previous chart
    d3.select(ratioSvg).selectAll("*").remove();

    // Calculate ratios for all years
    const ratioData = mergedData.map((d) => ({
      year: d.year,
      ratio: (d.rent / d.income) * 100,
    }));

    // Create scales
    const x = d3
      .scaleLinear()
      .domain([
        d3.min(ratioData, (d) => d.year),
        d3.max(ratioData, (d) => d.year),
      ])
      .range([0, ratioWidth]);

    const y = d3
      .scaleLinear()
      .domain([0, d3.max(ratioData, (d) => d.ratio)])
      .range([ratioHeight, 0]);

    // Create chart group
    const g = d3
      .select(ratioSvg)
      .attr("width", ratioWidth + ratioMargin.left + ratioMargin.right)
      .attr("height", ratioHeight + ratioMargin.top + ratioMargin.bottom)
      .append("g")
      .attr("transform", `translate(${ratioMargin.left},${ratioMargin.top})`);

    // Add Y grid lines
    g.append("g")
      .call(
        d3
          .axisLeft(y)
          .tickSize(-ratioWidth) // Make the ticks extend horizontally across the chart
          .tickFormat("") // Remove labels from grid lines
          .ticks(5)
      )
      .selectAll("line")
      .attr("opacity", 0.3);

    g.selectAll("path").attr("opacity", 0.3);

    g.append("line")
      .attr("x1", 0)
      .attr("x2", ratioWidth)
      .attr("y1", y(30))
      .attr("y2", y(30))
      .attr("stroke", "red")
      .attr("stroke-width", 2)
      .attr("stroke-dasharray", "6,3");

    // Add line
    const line = d3
      .line()
      .x((d) => x(d.year))
      .y((d) => y(d.ratio));

    g.append("path")
      .datum(ratioData)
      .attr("fill", "none")
      .attr("stroke", "#69b3a2")
      .attr("stroke-width", 2)
      .attr("d", line);

    // Add highlighted dot for selected year
    g.append("circle")
      .attr("cx", x(selectedYear))
      .attr("cy", y(ratioData.find((d) => d.year === selectedYear).ratio))
      .attr("r", 6)
      .attr("fill", "#ff7f0e");

    // Add axes (simplified)
    g.append("g")
      .attr("transform", `translate(0,${ratioHeight})`)
      .call(d3.axisBottom(x).tickFormat(d3.format("d")).ticks(5));

    g.append("g").call(
      d3
        .axisLeft(y)
        .tickFormat((d) => d + "%")
        .ticks(5)
    );
    // console.log("data: ", mergedData);
    incomeRentData = (() => {
      const d = mergedData.find((d) => d.year === selectedYear);
      // console.log("d: ", d.income);
      // console.log("d: ", d.rent.toFixed(2));
      return [
        {
          label: "Income",
          value: `$${d.income}`,
          color: "#69b3a2", // same green as your income bar
        },
        {
          label: "Rent",
          value: `$${d.rent.toFixed(2)}`,
          color: "#ff7f0e", // same orange as your rent bar
        },
        {
          label: "Ratio",
          value: `${((d.rent / d.income) * 100).toFixed(1)}%`,
          color: "grey", // neutral color for ratio
        },
      ];
    })();
  }
</script>

<!-- Replace the existing container structure with this new layout -->
<div class="page-container">
  <!-- <h2 style="text-align: center;">Rent to Income Ratio</h2> -->
  <div
    class="layout-container"
    style="display: flex; justify-content: center; align-items: center;"
  >
    <div class="visualization-container">
      <img
        src="walletnoBack.png"
        alt="Wallet illustration"
        style="width: 100%; max-width: 300px; margin-top: 20px;"
      />
      <svg bind:this={svg}></svg>
    </div>

    <div class="ratio-chart">
      <svg bind:this={ratioSvg}></svg>
    </div>
  </div>
  <ul class="legend">
    <p><strong>Year: {selectedYear}</strong></p>
    {#each incomeRentData as d, index}
      <li style="--color: {d.color};">
        <span class="swatch"></span>
        {d.label}: <em>{d.value}</em>
      </li>
    {/each}
  </ul>
</div>

<!-- <input type="range" min="2005" max="2021" bind:value={selectedYear} step="1" />
Selected Year: {selectedYear} -->

<style>
  .page-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 100vw;
  }

  .layout-container {
    display: flex;
    flex-direction: row;
    gap: 5rem;
    align-items: center;
    justify-content: center;
    width: 100%;
  }

  /* .layout-container {
    display: flex;
    flex-direction: row;
    gap: 5rem;
    align-items: center;
    width: 100vw;
  } */

  .visualization-container {
    position: relative;
    width: fit-content;
  }

  .ratio-chart {
    width: 300px;
    height: 100%;
    display: flex;
  }

  .visualization-container svg {
    position: absolute;
    top: 59%;
    left: 19%;
    transform: translate(-50%, -50%);
    font-family: sans-serif;
  }

  .ratio-chart svg {
    position: static;
    transform: none;
  }

  .legend {
    display: grid;
    gap: 10px;
    flex: 1;
    min-width: 200px;
    max-width: 200px;
    padding: 10px;
    border: 1px solid black;
  }

  .legend p {
    margin: 0 0 5px 0;
    font-weight: bold;
    font-size: 14px;
  }

  .legend li {
    display: flex;
    align-items: center;
    gap: 5px;
    margin-bottom: 5px;
    font-size: 12px;
  }

  .swatch {
    width: 10px;
    height: 10px;
    background-color: var(--color);
    border-radius: 2px;
    display: inline-block;
  }
</style>
