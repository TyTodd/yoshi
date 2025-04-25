<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  // Removed direct import since Vite cannot import CSVs from /public; data will be loaded dynamically
  const slant = -41;
  const barWidth = 106;
  let svg;
  let selectedYear = 2021; // single selected year filter
  let mergedData = [];
  $: filteredData = mergedData.filter((d) => d.year === selectedYear); // reactive filtered data

  // Load CSV data dynamically
  onMount(async () => {
    console.log("Loading CSV data...");
    const raw = await d3.csv("/income_and_rent.csv", d3.autoType);
    console.log("Raw data loaded:", raw);
    // Multiply rent by 12 for annual rent
    raw.forEach((d) => (d.rent = d.rent * 12));
    mergedData = raw;
  });

  // Function to update the chart based on the selected year
  $: if (filteredData.length && svg) {
    console.log("Updating chart for selected year:", selectedYear);
    const data = filteredData;
    console.log("Filtered data:", data);

    // Set up chart dimensions and margins
    const margin = { top: 0, right: 30, bottom: 48, left: 100 },
      height = 200 - margin.top - margin.bottom, // reduced from 800 to 400
      width = 120; // keeping the same width for thick bars

    // Clear previous chart elements
    d3.select(svg).selectAll("*").remove();
    console.log("Cleared previous SVG contents");

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
    console.log("Appended chart group element");

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
</script>

<!-- Create a container with relative positioning -->
<div class="visualization-container">
  <img
    src="walletnoBack.png"
    alt="Wallet illustration"
    style="width: 100%; max-width: 300px; margin-top: 20px;"
  />
  <!-- Position SVG absolutely over the image -->
  <svg bind:this={svg}></svg>
</div>

<!-- Year slider can stay outside the container -->
<input type="range" min="2005" max="2021" bind:value={selectedYear} step="1" />
Selected Year: {selectedYear}

<style>
  .visualization-container {
    position: relative;
    width: fit-content;
  }

  svg {
    position: absolute;
    top: 59%;
    left: 19%;
    transform: translate(-50%, -50%);
    font-family: sans-serif;
  }

  text {
    font-size: 12px;
    pointer-events: none;
  }
</style>
