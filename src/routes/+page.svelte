<script>
  import { onMount } from "svelte";
  import Pie from "$lib/Pie.svelte";
  import House from "$lib/House.svelte";
  import * as d3 from "d3";
  import { csv } from "d3";
  import Scrolly from "svelte-scrolly";
  import rentData from "$lib/boston_rent_estimates.json";
  import Wallet from "$lib/Wallet.svelte";
  import { fade } from "svelte/transition";

  const images = {
    peopleWearingMasks: "/yoshi/images/people_wearing_masks.jpeg",
    evictionNotice: "/yoshi/images/eviction_notice.jpeg",
    movingGif: "/yoshi/images/moving.gif",
    housing: "/yoshi/images/housing.jpg",
    timThinking: "/yoshi/images/timThinking.jpeg",
    timAngry: "/yoshi/images/timAngry.jpeg",
    timDreams: "/yoshi/images/timDreams.png",
    timSad: "/yoshi/images/timSad.png",
    cantReachVideo: "/yoshi/images/cantReach.mp4",
    housing2: "/yoshi/images/housing2.jpg",
    boston: "/yoshi/images/aerialBoston.jpg",
    fight: "/yoshi/images/fightback.png",
    bostonArt: "/yoshi/images/bostonArt.jpeg",
    bostonSkyLine: "/yoshi/images/bostonSkyline.jpg",
  };
  // const images = {
  //   peopleWearingMasks: "/images/people_wearing_masks.jpeg",
  //   evictionNotice: "/images/eviction_notice.jpeg",
  //   movingGif: "/images/moving.gif",
  //   housing: "/images/housing.jpg",
  //   timThinking: "/images/timThinking.jpeg",
  //   timAngry: "/images/timAngry.jpeg",
  //   timDreams: "/images/timDreams.png",
  //   timSad: "/images/timSad.png",
  //   timHappy: "/images/timHappy.jpeg",
  //   cantReachVideo: "/images/cantReach.mp4",
  //   housing2: "/images/housing2.jpg",
  //   boston: "/images/aerialBoston.jpg",
  //   fight: "/images/fightback.png",
  //   bostonArt: "/images/bostonArt.jpeg",
  //   bostonSkyLine: "/images/bostonSkyline.jpg",
  // };

  let videoTextProgress = 0;
  // PIE CHART
  let pieData = [];
  let geoData = []; // LINE GRAPH
  let pieProgress = 0;
  let selectedYear = 0;
  let selectedIndex = -1;
  $: walletProgress = 0;
  const colorScale = d3.scaleOrdinal(d3.schemeTableau10);

  const walletYearScale = d3
    .scaleLinear()
    .domain([0, 100])
    .range([2005, 2021])
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
  });

  $: if (pieData.length) {
    const step = 100 / pieData.length; // % each year occupies
    selectedYear = Math.max(
      0,
      Math.min(pieData.length - 1, Math.floor(pieProgress / step))
    );
  }
  $: if (rent_data.length) {
    const step = 100 / rent_data.length;
    rentYearIndex = Math.max(
      0,
      Math.min(rent_data.length - 1, Math.floor(rentProgress / step))
    );
    visibleRentData = rent_data.slice(0, rentYearIndex + 1);
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
  $: walletYear = Math.round(walletYearScale(walletProgress));
</script>

<svelte:head>
  <title>Home</title>
</svelte:head>

<body
  style="width: 100%; max-width: 140ch; margin: 0 auto; padding: 1em; min-height: 150vh"
>
  <!-- CONTEXT -->
  <div style="height: 800vh; text-align: center; font-size: 30px;">
    <div style="height: 100vh; ">
      <h1>Priced Out:</h1>
      <h2>The Death of the American Dream in Boston</h2>
      <img
        src={images.bostonSkyLine}
        alt="People wearing masks"
        style="width: 700px; object-fit: cover; border-radius: 50%;"
      />
      <hr />
    </div>
    <div style="height: 100vh; ">
      <img
        src={images.peopleWearingMasks}
        alt="People wearing masks"
        style="width: 600px; object-fit: cover; border-radius: 50%;"
      />
      <p>
        After mid-2021 when
        <span style="color: #1D3557;"><strong>COVID-era</strong></span>
        protections and federal rent assistance expired, a wave of evictions has
        since swept through the Greater Boston region.
      </p>
    </div>

    <div
      style="height: 100vh; display: flex; align-items: center; gap: 20px; text-align: center;"
    >
      <div>
        <p>
          State eviction filings now average 3,000 a month, up more than 15
          percent from pre-pandemic levels, according to Matija Jankovic from
          the Massachusetts Housing Partnership. Housing lawyers and advocates
          say evictions have reached crisis levels, causing waves of
          displacement that are ruining lives and communities in Boston.
        </p>
        <p>
          This eviction crisis is another manifestation of Boston’s broader
          housing affordability issue.
        </p>
      </div>
      <img
        src={images.evictionNotice}
        alt="eviction notice"
        style="width: 500px; object-fit: cover; border-radius: 50%;"
      />
    </div>
    <div style="height: 100vh; font-size: 70px;">
      <p>
        <span style="color: #1D3557;"
          ><strong>What are the culprits?</strong></span
        >
      </p>
    </div>
    <div style="height: 100vh; ">
      <img
        src={images.movingGif}
        alt="Guy holding up a house thats heavy"
        width="500px"
      />
      <p>
        One is Greater Boston’s severe housing shortage, which has pushed the
        rental vacancy rate down to
        <span style="color: #1D3557;"><strong>2.5 percent</strong></span> over
        the past few years, according to the Boston Foundation’s latest Housing
        Report Card. Among the 75 largest metropolitan areas in the U.S.,
        <span style="color: #1D3557;"><strong>only two</strong></span> have lower
        rates than Boston.
      </p>
      <p>
        <span style="color: #1D3557;"
          ><strong
            >This scarcity in rental vacancies has contributed to making Boston
            one of the most expensive rental markets in America.</strong
          ></span
        >
      </p>
    </div>

    <div style="height: 100vh; ">
      <img
        src={images.housing}
        alt="Guy holding up a house thats heavy"
        style="width: 500px; object-fit: cover; border-radius: 15%;"
      />
      <p>
        However, we believe another culprit has been overlooked, and deserves
        more investigation;
      </p>
      <p>
        <span style="color: #1D3557;"
          ><strong
            >The increasing presence of corporations in Boston’s real estate
            market.</strong
          ></span
        >
      </p>
    </div>

    <!-- INTRODUCING TIM -->
    <div
      style="display: flex; align-items: stretch; gap: 20px; text-align: center; height: 150vh"
    >
      <div style="flex: 1; position: relative;">
        <img
          src={images.timThinking}
          alt="MIT Tim the beaver thinking while looking at a laptop"
          style="width: 500px; object-fit: cover; border-radius: 25%; position: sticky; top: 20vh;"
        />
      </div>

      <div
        style="flex: 1; display: flex; flex-direction: column; justify-content: space-between; padding: 4em 2em; font-size: 30px;"
      >
        <p>
          Tim the Beaver is an individual who makes $77,771 a year, the median
          salary in Boston in 2024. Tim is looking to rent an apartment in
          Boston for around $2000 a month. Tim hopes to save up to buy a house
          in the future.
        </p>
        <p>
          We’ll time-travel with Tim from 2004-2024 to show how Tim’s
          experiences with the Boston area housing landscape would change.
        </p>
      </div>
    </div>
  </div>
  <!-- TRANSITION TO VISUAL 1-->
  <div
    style="display: flex; flex-direction: column; justify-content: center; align-items: center; font-size: 30px; padding: 2em 4em; height: 100vh; text-align: center;"
  >
    <p>
      While the pandemic revealed the fragility of Boston’s housing system, it
      also accelerated deeper, long-standing shifts that had been quietly
      reshaping the city for years.
    </p>
    <p>
      One of the most significant and arguabley most overlooked culprits behind
      this issue? The growing presence of corporate ownership in residential
      real estate.
    </p>
    <p>
      Before we can understand the pressures squeezing renters like Tim, we
      first need to do the research and find out who really owns Boston’s homes
      — and how that ownership has changed over time.
    </p>
  </div>

  <!-- VISUAL 1 -->
  <h1>Corporate ownership, owner occupancy & landlord occupancy</h1>
  <Scrolly bind:progress={pieProgress} --scrolly-layout="overlay">
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
    <div style="height: 225vh; font-size:30px;">
      <div
        style="
      position: sticky;
      top: 50%;
      transform: translateY(-50%);
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      width: 100%;
      gap: 2rem;
    "
      >
        {#if pieData[selectedYear]?.year >= 2006 && pieData[selectedYear]?.year <= 2009}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 1em; padding-right: 1em"
              in:fade
            >
              In the real-estate market, residences can be owned by the same
              owner who is occupying the residence, a landlord, or a
              corporation.
            </p>
          </div>
        {:else if pieData[selectedYear]?.year >= 2010 && pieData[selectedYear]?.year <= 2015}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 1em; padding-right: 1em"
              in:fade
            >
              During the past decade, private equity-backed firms and
              corporations have stormed into the multifamily apartment market,
              becoming major landlords in American cities, according to
              ProPublica’s analysis of National Multifamily Housing Council
              data. Private equity is now the dominant form of financial backing
              among the 35 largest owners of multifamily buildings.
            </p>
          </div>
        {:else if pieData[selectedYear]?.year >= 2016 && pieData[selectedYear]?.year <= 2020}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 1em; padding-right: 1em"
              in:fade
            >
              This influx of corporate ownership in the real-estate market,
              especially during a housing crisis, has dire impacts. These firms
              often act as corporate house flippers, seeking deals on apartment
              buildings, raising rent and slashing costs, and then selling the
              buildings at a higher price. When private equity firms buy up
              properties, their aim is to squeeze as much profit as possible.
              Rents skyrocket and become less flexible. Essential facilities,
              like heating, can become neglected. Sometimes landlords will force
              out existing tenants and replace them with new tenants who have
              the ability to pay more.
            </p>
          </div>
        {:else if pieData[selectedYear]?.year >= 2020 && pieData[selectedYear]?.year <= 2023}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 1em; padding-right: 1em"
              in:fade
            >
              In the Boston area real-estate market, from 2004 to 2024, the
              percentage of residences owned by corporations has steadily
              increased from 5.2% to 24.7%.
            </p>
          </div>
        {:else}
          <p></p>
        {/if}
      </div>
    </div>
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
  <div style="height:50vh; text-align:center">
    <p style="font-size: 30px">
      Over time, Tim would faces increasing rent prices and worsening facilities
      from unsympathetic corporate landlords who only prioritize profit.
    </p>
  </div>

  <!-- TRANSITION TO VISUAL 2-->
  <Scrolly bind:progress={videoTextProgress}>
    <!-- sticky video -->
    <svelte:fragment slot="viz">
      <div class="video-wrapper">
        <video autoplay loop muted playsinline>
          <source src={images.cantReachVideo} type="video/mp4" />
          Your browser does not support the video tag.
        </video>
      </div>
    </svelte:fragment>

    <!-- scrolling fading text -->
    <div class="scrolling-text">
      <p in:fade={{ duration: 800 }}>
        As corporate ownership grows, the character of Boston’s neighborhoods
        begins to change...
      </p>
      <p in:fade={{ duration: 1600 }}>
        For residents like Tim, these shifts aren't just statistics...
      </p>
      <p in:fade={{ duration: 2400 }}>
        <span style="color: #1D3557;"
          ><strong>Its the cost of their housing.</strong></span
        >
      </p>
      <p in:fade={{ duration: 3200 }}>
        Now let's explore how corporate ownership correlates with the rising
        cost of buying a home.
      </p>
    </div>
  </Scrolly>

  <!-- VISUAL 2 -->
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
        style="width:100%; height: 71%; border-width:0px"
      />
    </svelte:fragment>
    <div style="height: 600vh;">
      <div class="viz2">
        {#if mapYear >= 2005 && mapYear <= 2008}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 5em; padding-right: 5em"
              in:fade
            >
              The trend of increasing corporate ownership also parallels the
              increase in the average median housing prices from 2004 to 2024.
              Just as the corporate ownership rate in Boston increased from 5.2%
              to 25%, the median house price in Boston increased from $314,532
              to $[INSERT NUMBER].
            </p>
          </div>
        {:else if mapYear >= 2009 && mapYear <= 2015}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 5em; padding-right: 5em"
              in:fade
            >
              The influx of corporations in residents affects Boston’s
              neighborhoods unequally. In 2024, East Boston, South Boston, and
              Fenway are among the neighborhoods with the highest percentage of
              real estate owned by corporations
            </p>
          </div>
        {:else if mapYear >= 2016 && mapYear <= 2018}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 5em; padding-right: 5em"
              in:fade
            >
              This has undoubtedly also led to unequal impact on different
              communities in Boston, as lower-income residents are gradually
              being priced out of their homes by new corporate presences in
              their neighborhoods.
            </p>
          </div>
        {:else if mapYear >= 2019 && mapYear <= 2023}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 5em; padding-right: 5em"
              in:fade
            >
              Not only would this increase in corporate ownership of real estate
              throughout the Boston area affect Tim’s rent prices and
              facilities, but this would also affect Tim’s ability to afford
              living in specific neighborhoods, and Tim’s future goals of buying
              a house.
            </p>
          </div>
        {:else}
          <p></p>
        {/if}
      </div>
    </div>
  </Scrolly>

  <!-- TRANSITION TO VISUAL 3-->
  <div
    style="display: flex; justify-content: center; align-items: center; font-size: 30px; padding: 2em 4em; height: 100vh; text-align: center; gap: 50px"
  >
    <img
      src={images.timAngry}
      alt="Guy holding up a house thats heavy"
      style="width: 500px; object-fit: cover; border-radius: 50%"
    />
    <div>
      <p>
        Buying a home has become increasingly unattainable. For someone like
        Tim, fresh out of college and looking to build a future, the dream of
        homeownership is slipping out of reach.
      </p>
      <p>
        But it isn't just would-be homeowners who are feeling this strain. As
        corporate landlords tighten their grip, rents climbed in lockstep with
        property values — turning even modest apartments into luxuries reserved
        for the highest bidders.
      </p>
      <p>
        Next, we’ll follow the trajectory of Boston’s rental market, and see how
        it has begun squeezing everyday residents.
      </p>
    </div>
  </div>

  <!-- VISUAL 3 -->
  <h1>Boston Rent change over time</h1>
  <!-- <House data={[rent_data[15]]} /> -->
  <Scrolly bind:progress={rentProgress} --scrolly-layout="overlap">
    <svelte:fragment slot="viz">
      <House data={visibleRentData} />
    </svelte:fragment>
    <!-- <div style="height: 300vh" /> -->
    <div style="height: 300vh;">
      <div class="viz2" style="bottom:20%">
        {#if visibleRentData.length > 1 && visibleRentData.length <= 5}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 5em; padding-right: 5em"
              in:fade
            >
              For generations, homeownership was seen as the cornerstone of the
              American Dream. A symbol of financial stability and a path towards
              building generational wealth. But in Boston, that dream seems to
              be readily slipping out of reach.
            </p>
          </div>
        {:else if visibleRentData.length > 5 && visibleRentData.length <= 9}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 5em; padding-right: 5em"
              in:fade
            >
              Above is a graph of the CPI Owners Equivalent Rent for Boston over
              time, measuring how much homeowners would pay if they had to rent
              their own homes. Essentially this is another way to track the
              rising cost of simply having a roof over your head.
            </p>
          </div>
        {:else if visibleRentData.length > 9 && visibleRentData.length <= 15}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 5em; padding-right: 5em"
              in:fade
            >
              In Boston, where corporate ownership of housing has been rising
              dramatically over the last twenty years, these rent increases
              aren’t just abstract numbers. They represent the real pressure on
              real people – tenants like Tim, who are finding it harder every
              year to save, move or dream about home ownership.
            </p>
          </div>
        {:else if visibleRentData.length > 15 && visibleRentData.length <= 20}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black;"
          >
            <p
              style="font-size: 24px; padding-left: 5em; padding-right: 5em"
              in:fade
            >
              These trends set the stage for the rest of our story as rent eats
              up a larger share of income, corporate landlords tighten their
              grip on the markets and evictions rise and communities are pushed
              out.
            </p>
          </div>
        {:else}
          <p></p>
        {/if}
      </div>
    </div>
    <!-- <div style="height: 300vh;">
      {#each rent_data as d}
        <p>{d.year}: Rent ${d.rent}</p>
      {/each}
    </div> -->
  </Scrolly>

  <!-- TRANSITION TO VISUAL 4-->
  <div
    style="display: flex; justify-content: center; align-items: center; padding: 4em; height: 100vh; gap: 100px;"
  >
    <img
      src={images.timDreams}
      alt="Guy holding up a house that's heavy"
      style="width: 500px; max-width: 40vw; object-fit: cover; border-radius: 25%;"
    />
    <div style="max-width: 800px; font-size: 28px; text-align: center;">
      <p>
        Year after year, the rent burden grows heavier. What once was an
        affordable cost of living has now become a monthly scramble — a
        desperate juggling act between rent, groceries, healthcare, and savings.
      </p>
      <p>
        For Tim and countless others, it's not just about paying more: it's
        about the increasing sacrfice. Dreams of owning a home, starting a
        family, or even saving for emergencies are beginning to fade into the
        background.
      </p>
      <p>
        But we need to dive deeper to understand this sacrifice. Let's look into
        how much of Tim’s wallet was being devoured by housing alone.
      </p>
    </div>
  </div>

  <!-- VISUAL 4 -->
  <h1>Rent v. Income</h1>
  <Scrolly bind:progress={walletProgress} --scrolly-layout="overlap">
    <svelte:fragment slot="viz">
      <Wallet selectedYear={Math.round(walletYearScale(walletProgress))} />
    </svelte:fragment>
    <div style="height: 300vh; font-size:30px; width: 20vw">
      <div class="viz3">
        {#if walletYear >= 2006 && walletYear <= 2009}
          <div
            style="background-color: #F8F1E5; width: 100%; outline: 2px solid black; padding:10px"
          >
            <p in:fade>
              According to NerdWallet and other financial websites, you should
              spend no more than thirty percent of your income to pay rent.
            </p>
          </div>
        {:else if walletYear >= 2010 && walletYear <= 2013}
          <div
            style="background-color: #F8F1E5; width: 100%; outline: 2px solid black; padding:10px"
          >
            <p in:fade>
              As shown in this wallet visualization and line graph, we can see
              how the rent prices pass the 30% threshold, the critical turning
              point.
            </p>
          </div>
        {:else if walletYear >= 2014 && walletYear <= 2018}
          <div
            style="background-color: #F8F1E5; width: 100%; outline: 2px solid black; padding:10px"
          >
            <p in:fade>
              The increase from 25% to 30% in fifteen years in Boston has been a
              constant increase that shows an affordability gap and if nothing
              is done, it will continue to grow.
            </p>
          </div>
        {:else if walletYear >= 2019 && walletYear <= 2020}
          <div
            style="background-color: #F8F1E5; width: 100%; outline: 2px solid black; padding:10px"
          >
            <p in:fade>
              When reaching the critical turning point, many residents are not
              able to get basic needs covered since they need to use their money
              to pay rent. The other common case is that residents stop paying
              rent which leads to evictions.
            </p>
          </div>
        {:else}
          <p></p>
        {/if}
      </div>
    </div>
  </Scrolly>

  <!-- TRANSITION TO VISUAL 5-->
  <div
    style="display: flex; justify-content: center; align-items: center; padding: 4em; height: 100vh; gap: 100px;"
  >
    <img
      src={images.timSad}
      alt="Guy holding up a house that's heavy"
      style="width: 500px; max-width: 40vw; object-fit: cover; border-radius: 50%;"
    />
    <div style="max-width: 800px; font-size: 28px; text-align: center;">
      <p>
        As rent devoures an ever-larger share of income, something's got to
        give. For many Boston residents, the breaking point came when no amount
        of cutting back could make ends meet.
      </p>
      <p>
        <span style="color: #1D3557;"><strong>Missed Payments</strong></span>
        turn into
        <span style="color: #8B0000;"><strong>eviction notices</strong></span>.
      </p>
      <p>
        <span style="color: #1D3557;"><strong>Lifelong residents</strong></span>
        are
        <span style="color: #8B0000;"><strong>uprooted</strong></span>.
      </p>
      <p>
        <span style="color: #1D3557;"><strong>Entire communities</strong></span>
        <span style="color: #8B0000;"><strong>frayed</strong></span>.
      </p>
      <p>
        In the final part of our story, we’ll witness how the housing
        affordability crisis has transformed itself into an eviction crisis —
        reshaping the face of Boston itself.
      </p>
    </div>
  </div>

  <!-- VISUAL 5 -->
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
    <div style="height: 300vh;">
      <div class="viz2">
        {#if evictionMapYear == 2021}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black; font-size: 20px;"
          >
            <p style="padding-left: 2em; padding-right: 2em" in:fade>
              By 2019 cracks are already forming in Boston’s housing system, but
              it was the sharp turn between 2019 and 2020 that revealed how
              fragile things had become.
            </p>
            <p style="padding-left: 2em; padding-right: 2em" in:fade>
              When the pandemic hit, millions struggled to pay rent. Although
              emergency protections tried to aid people, once those protections
              expired during mid 2021, the eviction filings drastically
              increased. In Boston specifically, they only grew worse.
            </p>
          </div>
        {:else if evictionMapYear == 2022}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black; font-size: 20px;"
          >
            <p style="padding-left: 5em; padding-right: 5em" in:fade>
              Today, eviction filings average over 3,000 cases a month which is
              more than 15% higher than pre-pandemic levels. In Boston, eviction
              rates tied to non-payment of rent surged dramatically.
            </p>
            <p style="padding-left: 5em; padding-right: 5em" in:fade>
              The COVID-era protections may have ended, but their removal
              exposed deeper, ongoing wounds in Boston’s housing fabric: rising
              rents, stagnant wages, and corporate landlords tightening their
              grip.
            </p>
          </div>
        {:else if evictionMapYear == 2023}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black; font-size: 20px;"
          >
            <p style="padding-left: 5em; padding-right: 5em" in:fade>
              The pie chart highlights this human cost: the majority of
              evictions today are due to missed rent payments. And the 3D map
              visualizes how eviction rates due to rent issues have spread
              across Boston neighborhoods — hitting hardest in historically
              underrepresented, lower-income communities.
            </p>
          </div>
        {:else}
          <p></p>
        {/if}
      </div>
    </div>
  </Scrolly>

  <div style="height:100vh; font-size: 30px; text-align:center">
    <img
      src={images.housing2}
      alt="People wearing masks"
      style="width: 600px; object-fit: cover; border-radius: 25%;"
    />
    <p>These aren’t isolated cases. They are the</p>
    <p>
      <span style="color: #1D3557;"><strong>predictable</strong></span>
      and <span style="color: #1D3557;"><strong>preventable</strong></span>
    </p>
    <p>consequences of a system where</p>
    <p>
      <span style="color: #8B0000;"
        ><strong
          >housing is treated more as a commodity than a human right</strong
        ></span
      >
    </p>
  </div>

  <!-- CALL TO ACTION -->
  <div
    style="height: 100vh; display: flex; align-items: center; gap: 20px; text-align: center; font-size: 30px"
  >
    <div>
      <p>
        The eviction crisis isn’t just numbers on a map — it's a mirror
        reflecting a city at a crossroads.
      </p>
      <p>Boston faces two choices:</p>
      <p>Will we allow profit to eclipse people?</p>
      <p>
        Or will we fight to rebuild a city where stability, dignity, and
        homeownership are within reach for everyone?
      </p>
    </div>
    <img
      src={images.fight}
      alt="eviction notice"
      style="width: 500px; object-fit: cover; border-radius: 15%;"
    />
  </div>

  <div
    style="height: 150vh; align-items: center; gap: 20px; text-align: center;"
  >
    <div
      style="position: relative; height: 150vh; display: flex; flex-direction: column; align-items: center; gap: 20px; text-align: center; font-size:30px"
    >
      <!-- Main centered image -->
      <img
        src={images.bostonArt}
        alt="eviction notice"
        style="width: 800px; object-fit: cover; border-radius: 15%;"
      />

      <!-- Paragraphs -->
      <div>
        <p>It's not too late -- but action is urgent</p>
        <p>
          <span style="color: #1D3557;"
            ><strong>Advocate for stronger tenant protections.</strong></span
          >
        </p>
        <p>
          <span style="color: #1D3557;"
            ><strong>Support affordable housing initiatives.</strong></span
          >
        </p>
        <p>
          <span style="color: #1D3557;"
            ><strong>Hold corporate landlords accountable.</strong></span
          >
        </p>

        <p>
          Reimagine housing not as a financial asset, but as a foundation for
          thriving lives.
        </p>
        <p>
          A home should be a place to dream, not a battleground for survival.
        </p>
        <p>The future of Boston depends on it.</p>
      </div>

      <!-- Bottom-left image -->
      <img
        src={images.timHappy}
        alt="small supportive graphic"
        style="
      position: absolute;
      bottom: 0;
      left: 10px;
      width: 300px;
      object-fit: cover;
      border-radius: 50%;
    "
      />
    </div>
  </div>
</body>

<style>
  body {
    /* background-color: #f8f1e5; */
    color: #000000;
    font-family: "Libre Franklin", sans-serif;
  }

  h1,
  h2,
  h3 {
    color: #8b0000;
    font-family: "Merriweather", serif;
  }

  a {
    color: #1d3557;
  }

  hr {
    border-color: #d2b48c;
  }
  body::before {
    content: "";
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-image: url("/images/houseBackground.jpeg");
    background-repeat: repeat;
    background-size: 400px 400px;
    opacity: 0.05;
    z-index: -1;
    pointer-events: none;
  }
  .viz2 {
    position: fixed;
    bottom: 5%; /* a little off the bottom, tweak as you like */
    left: 0;
    width: 100%;
    display: flex;
    flex-direction: column;
    justify-content: flex-end;
    align-items: center;
    text-align: center;
    pointer-events: none; /* so you can still scroll through it */
  }
  .viz3 {
    position: fixed;
    width: 20vw;
    bottom: 40%;
    left: 10%;
    display: flex;
    flex-direction: column;
    justify-content: flex-end;
    align-items: center;
    text-align: center;
    pointer-events: none; /* so you can still scroll through it */
  }
  .video-wrapper {
    width: 500px;
    height: auto;
    position: sticky;
    top: 20vh;
    margin: 0 auto;
  }

  .video-wrapper video {
    width: 100%;
    border-radius: 20px;
  }

  .scrolling-text {
    height: 200vh; /* Make it tall enough to scroll */
    width: 60%;
    margin: 0 auto;
    padding-top: 50vh;
    display: flex;
    flex-direction: column;
    gap: 3em;
    font-size: 30px;
    text-align: center;
  }
</style>
