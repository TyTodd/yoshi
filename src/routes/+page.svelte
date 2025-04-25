<script>
  import { onMount } from "svelte";
  import Pie from "$lib/Pie.svelte";
  import House from "$lib/House.svelte";
  import * as d3 from "d3";
  import { csv } from "d3";
  import Scrolly from "svelte-scrolly";
  import rentData from "$lib/boston_rent_estimates.json";
  import Wallet from "$lib/Wallet.svelte";
  // PIE CHART
  let pieData = [];
  let geoData = []; // LINE GRAPH
  let pieProgress = 0;
  let selectedYear = 0;
  let selectedIndex = -1;
  let walletProgress = 0;
  const colorScale = d3.scaleOrdinal(d3.schemeTableau10);

  const walletYearScale = d3
    .scaleLinear()
    .domain([0, 1])
    .range([2005, 2020])
    .clamp(true);

  // RENT CHART
  const rent_data = rentData.map((d) => ({
    year: +d.Year,
    rent: +d.Estimated_Rent_USD,
    change: +d.YoYc,
    index: +d.CPI_Index,
  }));
  let rentProgress = 0;
  let rentYearIndex = -1;
  let visibleRentData = [];

  onMount(async () => {
    // const raw_data = await csv("/merged_dataset_full.csv", d3.autoType);
    const raw_data = await csv("merged_dataset_full.csv", d3.autoType);
    const year_groups = d3.group(raw_data, (d) => d.Year);

    pieData = Array.from(year_groups.entries())
      .map(([year, entries]) => {
        const corp = d3.mean(entries, (d) => d.corp_own_rate);
        const own = d3.mean(entries, (d) => d.own_occ_rate);
        const land = 1 - corp - own;

        return {
          year,
          data: [
            { label: "Corporate", value: corp },
            { label: "Owner-Occupied", value: own },
            { label: "Landlord", value: land },
          ],
        };
      })
      .sort((a, b) => +a.year - +b.year);

    geoData = Array.from(
      d3.group(raw_data, (d) => `${d.Year}-${d.Neighborhood}`).entries()
    )
      .map(([key, entries]) => {
        const year = entries[0].Year;
        const hood = entries[0].Neighborhood?.trim();
        const corp_own_rate = d3.mean(entries, (d) => d.corp_own_rate);
        const median_house_price = d3.mean(
          entries,
          (d) => d.Median_House_Price
        );

        return {
          year,
          hood,
          corp_own_rate,
          median_house_price,
        };
      })
      .sort((a, b) => +a.year - +b.year);
    // console.log("geoData");
    // console.log(geoData);
  });

  $: if (pieData.length) {
    // console.log(pieData[0].year);
    const step = 100 / pieData.length; // % each year occupies
    selectedYear = Math.max(
      0,
      Math.min(pieData.length - 1, Math.floor(pieProgress / step))
    );
    // console.log("step: ", step);
    // console.log("length: ", pieData.length - 1);
    // console.log("other: ", Math.floor(pieProgress / step));
  }
  $: if (rent_data.length) {
    const step = 100 / rent_data.length;
    // const rentYearIndex = Math.min(
    //   rent_data.length,
    //   Math.floor(rentProgress / step)
    // );
    rentYearIndex = Math.max(
      0,
      Math.min(rent_data.length - 1, Math.floor(rentProgress / step))
    );
    // console.log("rent data: ");
    // console.log(rentData);

    visibleRentData = rent_data.slice(0, rentYearIndex + 1);
    // console.log("visible data at ", rentYearIndex);
    // console.log(visibleRentData);
  }

  /* ----------------------- 3-D MAP (parent / iframe driver) ----------------------- */
  const MIN = 2004;
  const MAX = 2024;
  const SLOW = 0.01;

  let mapProgress = 0; // 0 → 1 from <Scrolly>
  let mapYear = MIN; // integer we send to iframe
  let frame; // <iframe> reference

  // convert scroll progress → year
  // $: mapYear = Math.round(MIN + mapProgress * (MAX - MIN));
  $: {
    // apply the “gear-ratio” then clamp to 1
    const p = Math.min(mapProgress * SLOW, 1);
    mapYear = Math.round(MIN + p * (MAX - MIN));
  }

  // push every new year to the iframe
  $: if (frame?.contentWindow) {
    frame.contentWindow.postMessage({ year: mapYear }, "*");
  }

  // send initial year once the iframe finishes loading
  function handleLoad(e) {
    const win = e.target.contentWindow;
    if (win) win.postMessage({ year: mapYear }, "*");
  }

  /* ----------------------- 3-D MAP (parent / iframe driver) ----------------------- */
  const MIN_evict = 2020;
  const MAX_evict = 2024;

  let evictionProgress = 0; // 0 → 1 from <Scrolly>
  let evictionMapYear = MIN_evict; // integer we send to iframe
  let evictionFrame; // <iframe> reference

  $: {
    // apply the “gear-ratio” then clamp to 1
    const p = Math.min(evictionProgress * SLOW, 1);
    evictionMapYear = Math.round(MIN_evict + p * (MAX_evict - MIN_evict));
  }

  // push every new year to the iframe
  $: if (evictionFrame?.contentWindow) {
    evictionFrame.contentWindow.postMessage({ year: evictionMapYear }, "*");
  }
</script>

<svelte:head>
  <title>Home</title>
</svelte:head>

<Scrolly bind:progress={walletProgress}>
  <svelte:fragment slot="viz">
    <Wallet selectedYear={walletYearScale(walletProgress)} />
  </svelte:fragment>
</Scrolly>

<body
  style="width: 100%; max-width: 140ch; margin: 0 auto; padding: 1em; min-height: 150vh"
>
  <h1>Corporate ownership, owner occupancy & landlord occupancy</h1>
  <Scrolly bind:progress={pieProgress}>
    <!-- <div style="min-height: 200vh;">
      {#each pieData as p}
        <p>
          {p.year}: {(p.data[0].value * 100).toFixed(2)} % of was corporate-owned,
          {(p.data[1].value * 100).toFixed(2)} % owner-occupied and {(
            p.data[2].value * 100
          ).toFixed(2)} % owned by landlords.
        </p>
      {/each}
    </div> -->
    <div style="height: 300vh" />
    <!-- Add this before your text -->
    <svelte:fragment slot="viz">
      {#key selectedYear}
        <Pie
          data={pieData[selectedYear]?.data ?? []}
          year={pieData[selectedYear]?.year}
          bind:selectedIndex
        />
      {/key}
    </svelte:fragment>
  </Scrolly>

  <h1>Corporate ownership & Housing Prices</h1>
  <Scrolly bind:progress={mapProgress} --scrolly-layout="overlap">
    <!-- viz slot -->
    <svelte:fragment slot="viz">
      <iframe
        bind:this={frame}
        on:load={handleLoad}
        src="3dmap.html"
        title="3-D map"
        class="w-full h-full border-0"
        style="width:100%; height: 85%; border-width:0px"
      />
      <!-- <Line data={geoData} /> -->
    </svelte:fragment>
    <div style="height: 300vh" />

    <!-- story slot (gives us scroll space) -->
    <!-- {#each geoData as g}
      <p>
        In {g.year}, <strong>{g.neighborhood}</strong> had {Math.round(
          g.corp * 100
        )}% corporate ownership and a median house price of ${g.house_price.toLocaleString()}.
      </p>
    {/each} -->
  </Scrolly>

  <h1>Boston Rent change over time</h1>
  <!-- <House data={[rent_data[15]]} /> -->
  <Scrolly bind:progress={rentProgress} --scrolly-layout="overlap">
    <svelte:fragment slot="viz">
      <House data={visibleRentData} />
    </svelte:fragment>
    <div style="height: 300vh" />
    <!-- <div style="height: 300vh;">
      {#each rent_data as d}
        <p>{d.year}: Rent ${d.rent}</p>
      {/each}
    </div> -->
  </Scrolly>

  <h1>Eviction</h1>
  <Scrolly bind:progress={evictionProgress} --scrolly-layout="overlap">
    <!-- viz slot -->
    <svelte:fragment slot="viz">
      <iframe
        bind:this={evictionFrame}
        on:load={handleLoad}
        src="evictions.html"
        title="3-D map"
        class="w-full h-full border-0"
        style="width:100%; height: 85%; border-width:0px"
      />
    </svelte:fragment>
    <div style="height: 300vh" />
  </Scrolly>
</body>

<style>
</style>
