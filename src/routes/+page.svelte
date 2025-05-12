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
  import AOS from "aos";
  import "aos/dist/aos.css";

  //Images for the actual website
  // const images = {
  //   peopleWearingMasks: "/yoshi/images/people_wearing_masks.jpeg",
  //   worried: "/yoshi/images/worried.png",
  //   evictionNotice: "/yoshi/images/eviction_notice.png",
  //   eviction: "/yoshi/images/evictionNotice2.jpg",
  //   movingGif: "/yoshi/images/moving.gif",
  //   evilCorp: "/yoshi/images/evil_corps.png",
  //   frayed: "/yoshi/images/frayed.png",
  //   cantReachVideo: "/yoshi/images/cantReach.mp4",
  //   housing2: "/yoshi/images/housing2.png",
  //   fight: "/yoshi/images/fight.png",
  //   bostonArt: "/yoshi/images/bostonArt.jpeg",
  //   bostonSkyLine: "/yoshi/images/bostonSkyline.jpg",
  //   vacancyHousing: "/yoshi/images/vacancy.jpeg",
  //   rent: "/yoshi/images/rentBurden.png",
  //   bost: "/yoshi/images/bost_transparent.png",
  // };
  // const walkingImages = Array.from(
  //   { length: frameCount },
  //   (_, i) => `/yoshi/images/walking/walking${i + 1}.png`
  // );

  //Images for the local host website
  const images = {
    peopleWearingMasks: "/images/people_wearing_masks.jpeg",
    worried: "/images/worried.png",
    evictionNotice: "/images/eviction_notice.png",
    eviction: "/images/evictionNotice2.jpg",
    movingGif: "/images/moving.gif",
    evilCorp: "/images/evil_corps.png",
    frayed: "/images/frayed.png",
    cantReachVideo: "/images/cantReach.mp4",
    housing2: "/images/housing2.png",
    fight: "/images/fight.png",
    bostonArt: "/images/bostonArt.jpeg",
    bostonSkyLine: "/images/bostonSkyline.jpg",
    vacancyHousing: "/images/vacancy.jpeg",
    rent: "/images/rentBurden.png",
    bost: "/images/bost_transparent.png",
  };
  const walkingImages = Array.from(
    { length: frameCount },
    (_, i) => `/images/walking/walking${i + 1}.png`
  );

  //text
  let introProgress;
  let transitionProgress;
  let scrollProgress;

  //Navbar progress
  const updateProgress = () => {
    const scrollTop = window.scrollY;
    const scrollHeight =
      document.documentElement.scrollHeight - window.innerHeight;
    scrollProgress = (scrollTop / scrollHeight) * 100;
  };

  //Nav Bar
  let activeSection = "intro";

  const sections = [
    "intro",
    "ownership",
    "prices",
    "rent",
    "evictions",
    "actionables",
  ];

  function scrollToSection(section) {
    activeSection = section; // explicitly set active section on click
    const el = document.getElementById(section);
    el?.scrollIntoView({ behavior: "smooth", block: "start" });
  }

  // This updates activeSection based on scroll position
  function handleScroll() {
    const center = window.innerHeight / 2; // vertical midpoint
    for (let i = sections.length - 1; i >= 0; i--) {
      const el = document.getElementById(sections[i]);
      if (!el) continue;
      const { top, bottom } = el.getBoundingClientRect();
      if (top <= center && bottom >= center) {
        activeSection = sections[i];
        break;
      }
    }
  }

  //walking
  let walkingProgress;
  const frameCount = 12;

  let videoTextProgress = 0;
  // PIE CHART
  let pieData = [];
  let geoData = []; // LINE GRAPH
  let pieProgress = 0;
  let selectedYear = 0;
  let selectedIndex = -1;

  // Wallet
  $: walletProgress = 0;

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

  /* ----------------------- 3-D MAP (parent / iframe driver) ----------------------- */
  const MIN = 2004;
  const MAX = 2024;
  const SLOW = 0.01;
  let mapProgress = 0; // 0 → 1 from <Scrolly>
  let mapYear = MIN; // integer we send to iframe
  let frame; // <iframe> reference

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

  /* ----------------------- EVICTION 3-D MAP (parent / iframe driver) ----------------------- */
  const MIN_evict = 2019;
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

  onMount(async () => {
    AOS.init();
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

    // Add scroll tracking event listeners
    window.addEventListener("scroll", handleScroll);
    window.addEventListener("scroll", updateProgress);

    // Initial call
    handleScroll();
    updateProgress();

    // Combined cleanup
    return () => {
      window.removeEventListener("scroll", handleScroll);
      window.removeEventListener("scroll", updateProgress);
    };
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
</script>

<svelte:head>
  <title>Home</title>
</svelte:head>

<body
  style="width: 100%; max-width: 100%; margin: 0 auto; padding: 1em; min-height: 150vh;"
>
  <div class="navbar-progress" style="width: {scrollProgress}%;"></div>
  <nav style="font-size: 18px">
    <ul style="border-bottom: 1px solid rgba(0, 0, 0, 0.3);">
      {#each sections as section}
        <li>
          <a
            href="#{section}"
            class:active={activeSection === section}
            on:click|preventDefault={() => scrollToSection(section)}
          >
            {section.charAt(0).toUpperCase() + section.slice(1)}
          </a>
        </li>
      {/each}
    </ul>
  </nav>

  <Scrolly bind:progress={introProgress}>
    <!-- CONTEXT -->
    <div id="intro" style="height: 725vh; text-align: center; font-size: 30px;">
      <div style="height: 100vh; ">
        <h2>Priced Out:</h2>
        <h2>The Death of the American Dream in Boston</h2>
        <img
          src={images.bostonSkyLine}
          alt="Boston skyline painting"
          style="max-width: 700px; width: 700px; object-fit: cover; border-radius: 50%;"
        />
        <hr />
      </div>
      <div
        style="height: 100vh; width: 100vw; display: flex; flex-direction: column; justify-content: center; align-items: center; text-align: center;"
      >
        {#if introProgress > 13}
          <img
            data-aos="fade-left"
            data-aos-delay="100"
            src={images.peopleWearingMasks}
            alt="People wearing masks"
            style="max-width: 600px; width: 600px; object-fit: cover; border-radius: 50%; position: absolute; top: 15%"
          />
          <p
            data-aos="fade-right"
            data-aos-delay="100"
            style="width:60%; position: absolute; top: 23%"
          >
            After mid-2021 when COVID-era protections and federal rent
            assistance expired,
            <span class="highlight"><strong>a wave of evictions </strong></span
            >swept through the Greater Boston region.
          </p>
        {/if}
      </div>
      <div
        style="height: 100vh; display: flex; align-items: center; gap: 50px; text-align: center;"
      >
        <div
          data-aos="fade-up"
          data-aos-delay="100"
          style="padding-left:100px; width: 50%"
        >
          <p>
            State eviction filings now average 3,000 a month, up
            <span class="highlight"
              ><strong>more than 15 percent from pre-pandemic levels</strong
              ></span
            >, according to Matija Jankovic from the Massachusetts Housing
            Partnership. Housing lawyers and advocates say
            <span class="highlight"
              ><strong>evictions have reached crisis levels</strong></span
            >, causing
            <span class="highlight"><strong>waves of displacement</strong></span
            > that are ruining lives and communities in Boston.
          </p>
          <p>
            This eviction crisis is another manifestation of Boston’s broader
            housing affordability issue.
          </p>
        </div>
        <img
          src={images.evictionNotice}
          alt="Women given eviction notice"
          style="max-width: 500px; width: 500px; height: 450px; object-fit: cover; border-radius: 15%;"
        />
      </div>
      <div
        style="height: 80vh; width: 100vw; display: flex; flex-direction: column; justify-content: center; align-items: center; text-align: center; font-size: 70px;"
      >
        {#if introProgress > 42}
          <h1 data-aos="zoom-in-up" data-aos-delay="200">
            Who are the culprits?
          </h1>
        {/if}
      </div>
      <div
        style="height: 110vh; display: flex; flex-direction: column; justify-content: flex-start; align-items: center; text-align: center; padding-top: 40px;"
      >
        <img
          src={images.vacancyHousing}
          alt="Multiple vacant houses in a neighborhood"
          style="max-width: 500px; width:500px"
        />
        <p style="width:60%;">
          One is Greater Boston’s severe housing shortage, which has pushed the
          rental vacancy rate down to 2.5 percent over the past few years,
          according to the Boston Foundation’s latest Housing Report Card. Among
          the 75 largest metropolitan areas in the U.S., only two have lower
          rates than Boston.
        </p>
        {#if introProgress > 60}
          <p data-aos="fade-up" data-aos-delay="200" style="width:60%;">
            This scarcity in rental vacancies has contributed to making Boston <span
              class="highlight"
              ><strong
                >one of the most expensive rental markets in America</strong
              ></span
            >.
          </p>
        {/if}
      </div>

      <div
        style="height: 125vh; display: flex; flex-direction: column; justify-content: flex-start; align-items: center; text-align: center;"
      >
        <img
          data-aos="zoom-out"
          data-aos-delay="200"
          src={images.evilCorp}
          alt="Evil presence of corporations looming over city"
          style="max-height: 500px; height: 500px; object-fit: cover; border-radius: 15%;"
        />
        <p style="width: 60%; padding-top: 30px; margin-bottom:0">
          However, we believe another culprit has been overlooked, and deserves
          more investigation;
        </p>
        {#if introProgress > 60}
          <p
            data-aos="zoom-out"
            data-aos-delay="100"
            style="width: 60%; top: 1%;"
          >
            <span class="highlight"
              ><strong
                >The increasing presence of corporations in Boston’s real estate
                market.</strong
              ></span
            >
          </p>
        {/if}
      </div>

      <!-- INTRODUCING TIM -->
      <Scrolly bind:progress={walkingProgress} --scrolly-layout="overlay">
        <div style="height: 120vh;">
          <!-- Walker -->
          <div
            class="walker"
            style="transform: translateX({Math.min(walkingProgress * 0.01, 1) *
              80}vw);"
          >
            {#if walkingProgress > 0 && walkingProgress < 100}
              <div
                style="width: 400px; height: 400px; overflow: hidden; position: relative;"
                transition:fade={{ duration: 200 }}
              >
                <img
                  src={walkingImages[
                    Math.floor(
                      (Math.min(walkingProgress * 0.01, 1) * 80) % frameCount
                    )
                  ]}
                  alt="Walking animation"
                  style="width: 500px; height: auto; position: absolute; left: -80px; top: 0;"
                />
              </div>
            {/if}
          </div>
          <!-- Text overlays -->
          <div
            style="position: sticky; top: 20%; left: 10%; width: 80%; color: black; font-size: 30px"
          >
            {#if walkingProgress > 20 && walkingProgress < 30}
              <p transition:fade>This is Tim.</p>
            {/if}

            {#if walkingProgress >= 30 && walkingProgress < 50}
              <div>
                <p transition:fade>
                  <span class="highlight"
                    ><strong>Tim was recently evicted</strong></span
                  >.
                </p>
                <img
                  transition:fade
                  src={images.eviction}
                  alt="Person holding an eviction notice"
                  style="width: 400px; border-radius:50%"
                />
              </div>
            {/if}

            {#if walkingProgress >= 55 && walkingProgress < 70}
              <p transition:fade>How this happened, you ask?</p>
            {/if}

            {#if walkingProgress >= 70 && walkingProgress < 90}
              <p transition:fade>
                Well, we'll travel back in time with Tim to answer that very
                question.
              </p>
            {/if}
          </div>
        </div>
      </Scrolly>
    </div>
  </Scrolly>
  <!-- TRANSITION TO VISUAL 1-->
  <div
    id="ownership"
    style="
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  font-size: 30px;
  padding: 6em 4em 2em 4em;
  height: 125vh;
  text-align: center;
"
  >
    <p style="width:60%">
      While the pandemic revealed the fragility of Boston’s housing system, it
      also accelerated deeper, long-standing shifts that had been quietly
      reshaping the city for years.
    </p>
    <p style="width:60%">
      Before we can understand the pressures squeezing renters like Tim, we
      first need to do the research and find out
      <span class="highlight"
        ><strong>who really owns Boston’s homes</strong></span
      >
      — and how that
      <span class="highlight"
        ><strong>ownership has changed over time</strong></span
      >.
    </p>
  </div>

  <!-- VISUAL 1 -->
  <Scrolly bind:progress={pieProgress}>
    <h1
      style="
    position: sticky;
    top: 10%;
    padding: 0.5em 1em;
    margin: 0;
    font-size:30px;
    z-index: 1000;
  "
    >
      Corporate Ownership, Owner Occupancy & Landlord Occupancy Rates
    </h1>
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
              data. <span class="highlight"
                ><strong
                  >Private equity is now the dominant form of financial backing</strong
                ></span
              >
              among the 35 largest owners of multifamily buildings.
            </p>
          </div>
        {:else if pieData[selectedYear]?.year >= 2016 && pieData[selectedYear]?.year <= 2020}
          <div
            style="background-color: #F8F1E5; width: 90%; outline: 2px solid black; margin-top: 100px;"
          >
            <p
              style="font-size: 24px; padding-left: 1em; padding-right: 1em"
              in:fade
            >
              This influx of corporate ownership in the real-estate market,
              especially during a housing crisis, has dire impacts. These firms
              often act as corporate house flippers, seeking deals on apartment
              buildings, raising rent and slashing costs, and then selling the
              buildings at a higher price.
            </p>
            <p
              style="font-size: 24px; padding-left: 1em; padding-right: 1em"
              in:fade
            >
              When private equity firms buy up properties, their aim is to <span
                class="highlight"
                ><strong>squeeze as much profit as possible</strong></span
              >. Rents skyrocket and become less flexible. Essential facilities,
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
              <span class="highlight"
                ><strong>increased from 5.2% to 24.7%</strong></span
              >.
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
  <div
    style="height:50vh; display: flex; flex-direction: column; justify-content: center; align-items: center; text-align:center"
  >
    <p style="font-size: 30px; width:60%">
      Over time, Tim would faces increasing rent prices and worsening facilities
      from <span class="highlight"
        ><strong
          >unsympathetic corporate landlords who only prioritize profit</strong
        ></span
      >.
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
      <p data-aos="flip-up" data-aos-delay="400">
        As corporate ownership grows, the character of Boston’s neighborhoods
        begins to change...
      </p>
      <p data-aos="flip-up" data-aos-delay="400">
        For residents like Tim, these shifts aren't just statistics...
      </p>
      <p data-aos="flip-up" data-aos-delay="400">
        <span class="highlight"
          ><strong>Its the cost of their housing.</strong></span
        >
      </p>
      <p data-aos="flip-up" data-aos-delay="400">
        Now let's explore how corporate ownership correlates with the rising
        cost of buying a home.
      </p>
    </div>
  </Scrolly>
  <!-- Instructions on using visual 2 -->
  <div class="instructions">
    <p>Instructions for using the following visual:</p>
    <p>
      Hover over a neighborhood to see neighborhood specific stats (corporate
      ownership rate and median home price). The line charts demonstrate the
      boston wide averages, click on a neighborhood to see how it compares to
      the boston wide stats
    </p>
  </div>

  <!-- VISUAL 2 -->

  <Scrolly bind:progress={mapProgress} --scrolly-layout="overlap">
    <h1
      style="
      position: sticky;
  top: 5%;
  padding: 0.5em 1em;
  margin: 0;
  font-size:30px;
  z-index: 1000;
"
    >
      Corporate ownership & Housing Prices
    </h1>
    <!-- viz slot -->
    <svelte:fragment slot="viz">
      <div style="height: 100vh; display:flex; flex-direction: column;">
        <div
          style="position: sticky; width: 100%; height: 80%; overflow: hidden; position: relative;"
        >
          <iframe
            bind:this={frame}
            on:load={handleLoad}
            src="3dmap.html"
            title="3-D map"
            style="position: absolute; width: 95vw; height: 100vh; border: none;"
          />
        </div>
        <div class="corp-text">
          {#if mapYear >= 2009 && mapYear <= 2011}
            <div class="visual_caption">
              <p in:fade>
                The trend of increasing corporate ownership also parallels the
                increase in the average median housing prices from 2004 to 2024.
                Just as the
                <span class="highlight"
                  ><strong
                    >corporate ownership rate in Boston increased from 5.2% to
                    25%</strong
                  ></span
                >, the
                <span class="highlight"
                  ><strong
                    >median house price in Boston increased from $314,532 to
                    $659,616.71</strong
                  ></span
                >.
              </p>
            </div>
          {:else if mapYear >= 2012 && mapYear <= 2015}
            <div class="visual_caption">
              <p in:fade>
                The influx of corporations in residents affects Boston’s
                neighborhoods unequally. In 2024, East Boston, South Boston, and
                Fenway are among the neighborhoods with the highest percentage
                of real estate owned by corporations
              </p>
            </div>
          {:else if mapYear >= 2016 && mapYear <= 2018}
            <div class="visual_caption">
              <p in:fade>
                This has undoubtedly also led to unequal impact on different
                communities in Boston, as lower-income residents are gradually
                being priced out of their homes by new corporate presences in
                their neighborhoods.
              </p>
            </div>
          {:else if mapYear >= 2019 && mapYear <= 2023}
            <div class="visual_caption">
              <p class="visual_caption_text" in:fade>
                This increase affects not only rent prices and facilities, but
                also one's ability to afford living in specific neighborhoods,
                and any future goals of buying a house.
              </p>
              <p class="small-corp-text">
                <em
                  >Note: Median House Price for 2023 and 2024 was not available.</em
                >
              </p>
            </div>
          {:else}
            <p></p>
          {/if}
        </div>
      </div>
    </svelte:fragment>
    <div style="height: 300vh; width: 100%"></div>
  </Scrolly>

  <!-- TRANSITION TO VISUAL 3-->
  <div
    id="prices"
    style="display: flex; justify-content: center; align-items: center; font-size: 30px; padding: 2em 4em; height: 100vh; text-align: center; gap: 50px"
  >
    <img
      src={images.worried}
      alt="Worried person"
      style="max-width: 400px; width: 400px; object-fit: cover"
    />
    <div>
      <p>
        Buying and retaining a home has become increasingly unattainable. For
        someone like Tim and
        <span class="highlight"
          ><strong>the dream of homeownership is slipping out of reach</strong
          ></span
        >.
      </p>
      <p>
        As
        <span class="highlight"
          ><strong>corporate landlords tighten their grip</strong></span
        >, rents climbed in lockstep with property values — turning even modest
        apartments into luxuries reserved for the highest bidders.
      </p>
      <p>
        Next, we’ll follow the trajectory of Boston’s rental market, and see how
        it has begun squeezing everyday residents.
      </p>
    </div>
  </div>

  <!-- VISUAL 3 -->
  <Scrolly bind:progress={rentProgress} --scrolly-layout="overlap">
    <h1
      style="
    position: sticky;
    top: 5%;
    padding: 0.5em 1em;
    margin: 0;
    font-size:30px;
    z-index: 1000;
  "
    >
      Boston Rent change over time
    </h1>
    <svelte:fragment slot="viz">
      <div style="height: 100vh; display:flex; flex-direction: column;">
        <div
          style="transform: scale(1.1) translateY(50px); transform-origin: top center"
        >
          <House data={visibleRentData} />
        </div>
        <div class="viz2" style="bottom:2%">
          {#if visibleRentData.length > 4 && visibleRentData.length <= 5}
            <div class="caption">
              <p class="visual_caption" in:fade>
                For generations, homeownership was seen as the cornerstone of
                the American Dream. A symbol of financial stability and a path
                towards building generational wealth. But in Boston, that dream
                seems to be readily slipping out of reach.
              </p>
            </div>
          {:else if visibleRentData.length > 5 && visibleRentData.length <= 9}
            <div class="caption">
              <p class="visual_caption" in:fade>
                Above is a graph of the CPI Owners Equivalent Rent for Boston
                over time, measuring how much homeowners would pay if they had
                to rent their own homes. Essentially this is another way to
                track the rising cost of simply having a roof over your head.
              </p>
            </div>
          {:else if visibleRentData.length > 9 && visibleRentData.length <= 15}
            <div class="caption">
              <p class="visual_caption" in:fade>
                In Boston, where corporate ownership of housing has been rising
                dramatically over the last twenty years, these rent increases
                aren’t just abstract numbers. They represent the real pressure
                on real people – tenants like Tim, who are finding it harder
                every year to save, move or dream about home ownership.
              </p>
            </div>
          {:else if visibleRentData.length > 15 && visibleRentData.length <= 20}
            <div class="caption">
              <p class="visual_caption" in:fade>
                These trends set the stage for the rest of our story as rent
                eats up a larger share of income, <span class="highlight"
                  ><strong
                    >corporate landlords expand their control over the housing
                    market</strong
                  ></span
                >
                and
                <span class="highlight"><strong>evictions rise</strong></span>
                and
                <span class="highlight"
                  ><strong>communities are pushed out</strong></span
                >.
              </p>
            </div>
          {:else}
            <p></p>
          {/if}
        </div>
      </div>
    </svelte:fragment>
    <div style="height: 300vh;"></div>
  </Scrolly>
  <!-- TRANSITION TO VISUAL 4-->
  <div
    id="rent"
    style="display: flex; justify-content: center; align-items: center; padding: 4em; height: 100vh; gap: 100px;"
  >
    <img
      src={images.rent}
      alt="Guy holding up a heavy house"
      style="max-width: 400px; width: 400px; max-width: 40vw; object-fit: cover; border-radius: 25%;"
    />
    <div style="max-width: 800px; font-size: 28px; text-align: center;">
      <p>
        Year after year, <span class="highlight"
          ><strong>the rent burden grows heavier</strong></span
        >. What once was an affordable cost of living has now become a monthly
        scramble — a desperate juggling act between rent, groceries, healthcare,
        and savings.
      </p>
      <p>
        For Tim and countless others, it's not just about paying more: it's
        about the
        <span class="highlight"><strong>increasing sacrfice</strong></span>.
        Dreams of owning a home, starting a family, or even saving for
        emergencies are
        <span class="highlight"
          ><strong>beginning to fade into the background</strong></span
        >.
      </p>
      <p>
        But we need to dive deeper to understand this sacrifice. Let's look into
        how much of Tim’s wallet was being devoured by housing alone.
      </p>
    </div>
  </div>

  <!-- VISUAL 4 -->
  <Scrolly bind:progress={walletProgress} --scrolly-layout="overlap">
    <h1
      style="
    position: sticky;
    top: 5%;
    padding: 0.5em 1em;
    margin: 0;
    font-size:30px;
    z-index: 1000;
  "
    >
      Rent v. Income
    </h1>
    <svelte:fragment slot="viz">
      <div style="transform: scale(1.8); transform-origin: top center;">
        <Wallet selectedYear={Math.round(walletYearScale(walletProgress))} />
      </div>
    </svelte:fragment>
    <div style="height: 300vh; width: 20vw">
      <div class="viz2" style="bottom: 5%">
        {#if walletYear >= 2009 && walletYear <= 2011}
          <div
            style="background-color: #F8F1E5; width: 100%; outline: 2px solid black; padding:10px"
          >
            <p class="visual_caption" in:fade>
              According to NerdWallet and other financial websites, you should
              spend no more than <span class="highlight"
                ><strong>30% of your income to pay rent</strong></span
              >.
            </p>
            <p style="font-size:14px">
              <em
                >Note: This is represented by the
                <span class="red"><strong>red dotted line</strong></span> in the
                graph.</em
              >
            </p>
          </div>
        {:else if walletYear >= 2012 && walletYear <= 2014}
          <div
            style="background-color: #F8F1E5; width: 100%; outline: 2px solid black; padding:10px"
          >
            <p class="visual_caption" in:fade>
              As shown in this wallet visualization and line graph, we can see
              how the rent prices pass the 30% threshold, the critical turning
              point.
            </p>
          </div>
        {:else if walletYear >= 2015 && walletYear <= 2016}
          <div
            style="background-color: #F8F1E5; width: 100%; outline: 2px solid black; padding:10px"
          >
            <p class="visual_caption" in:fade>
              The
              <span class="highlight"
                ><strong>increase from 25% to 30%</strong></span
              > in these last 15 years in Boston has been a constant increase that
              shows an affordability gap and if nothing is done, it will continue
              to grow.
            </p>
          </div>
        {:else if walletYear >= 2017 && walletYear <= 2020}
          <div
            style="background-color: #F8F1E5; width: 100%; outline: 2px solid black; padding:10px; "
          >
            <p class="visual_caption" in:fade>
              When reaching the critical turning point, many residents are
              <span class="highlight"
                ><strong>not able to get basic needs covered</strong></span
              >
              since they need to use their money to pay rent. This inability to pay
              rent,
              <span class="highlight"><strong>leads to evictions</strong></span
              >.
            </p>
          </div>
        {:else}
          <p></p>
        {/if}
      </div>
    </div>
  </Scrolly>

  <!-- TRANSITION TO VISUAL 5-->
  <Scrolly bind:progress={transitionProgress} --scrolly-layout="overlap">
    <div
      id="evictions"
      style="display: flex; justify-content: center; align-items: center; padding: 4em; height: 100vh; gap: 100px;"
    >
      <img
        src={images.frayed}
        alt="Corporations destroying homes"
        style="max-width: 600px; width: 600px; max-width: 40vw; object-fit: cover; border-radius: 5%; position: absolute; left: 5%"
      />
      <div
        style="max-width: 800px; font-size: 24px; text-align: center; height: 80%;"
      >
        <p style="position:absolute; top: 28%">
          As rent devoures an ever-larger share of income, something's got to
          give. For many Boston residents, the breaking point came when no
          amount of cutting back could make ends meet.
        </p>
        {#if transitionProgress > 15}
          <p
            data-aos="fade-in"
            data-aos-delay="200"
            style="width:60%; position: absolute; top: 40%; left: 45%"
          >
            Missed Payments turn into
            <span class="highlight"><strong>eviction notices</strong></span>.
          </p>
        {/if}
        {#if transitionProgress > 50}
          <p
            data-aos="fade-in"
            data-aos-delay="400"
            style="width:60%; position: absolute; top: 45%; left: 45%"
          >
            Lifelong residents are
            <span class="highlight"><strong>uprooted</strong></span>.
          </p>
        {/if}
        {#if transitionProgress > 70}
          <p
            data-aos="fade-in"
            data-aos-delay="600"
            style="width:60%; position: absolute; top: 50%; left: 45%"
          >
            Entire communities
            <span class="highlight"><strong>frayed</strong></span>.
          </p>
        {/if}
        <p style="width:40%; position: absolute; top: 55%; left: 55%">
          In the final part of our story, we’ll witness how the housing
          affordability crisis has transformed itself into an eviction crisis —
          reshaping the face of Boston itself.
        </p>
      </div>
    </div>
  </Scrolly>

  <!-- VISUAL 5 -->
  <Scrolly bind:progress={evictionProgress} --scrolly-layout="overlap">
    <h1
      style="
    position: sticky;
    top: 5%;
    padding: 0.5em 1em;
    margin: 0;
    font-size:30px;
    z-index: 1000;
  "
    >
      Eviction Over Time in Boston Neighborhoods
    </h1>
    <!-- viz slot -->
    <svelte:fragment slot="viz">
      <iframe
        bind:this={evictionFrame}
        on:load={handleLoad}
        src="evictions.html"
        title="3-D map"
        class="w-full h-full border-0"
        style="transform: translateY(25px); width:100%; height: 85%; border-width:0px"
      />
    </svelte:fragment>
    <div style="height: 200vh;">
      <div class="viz2" style="bottom:5%">
        {#if evictionProgress > 30 && evictionProgress < 44}
          <div class="caption">
            <p class="visual_caption" in:fade>
              By 2019 cracks are already forming in Boston’s housing system, and
              between 2019 and 2020 revealed how fragile things had become.When
              the pandemic hit, millions struggled to pay rent. Although
              emergency protections tried to aid people, once those protections
              expired during mid 2021, the
              <span class="highlight"
                ><strong>eviction filings drastically increased</strong></span
              >. In Boston specifically, they only grew worse.
            </p>
          </div>
        {:else if evictionProgress > 45 && evictionProgress < 64}
          <div class="caption">
            <p class="visual_caption" in:fade>
              Today, eviction filings average over 3,000 cases a month, more
              than 15% higher than pre-pandemic levels. In Boston, eviction
              rates tied to non-payment of rent surged dramatically.
            </p>
            <p class="visual_caption" in:fade>
              The COVID-era protections may have ended, but their removal
              exposed deeper, ongoing wounds in Boston’s housing fabric:
              <span class="highlight"
                ><strong
                  >rising rents, stagnant wages, and the growing influence of
                  corporate landlords</strong
                ></span
              >.
            </p>
          </div>
        {:else if evictionProgress > 65 && evictionProgress < 84}
          <div class="caption">
            <p class="visual_caption" in:fade>
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

  <div
    id="actionables"
    style="display: flex; justify-content: center; align-items: center; height: 100vh; font-size: 30px; text-align: center; gap: 50px"
  >
    <img
      src={images.housing2}
      alt="Housing getting more pricey"
      style="max-height: 600px; height: 600px; object-fit: cover; border-radius: 25%;"
    />
    <div>
      <p>These aren’t isolated cases. They are the</p>
      <p>
        <span class="highlight"><strong>predictable</strong></span>
        and <span class="highlight"><strong>preventable</strong></span>
      </p>
      <p>consequences of a system where housing is treated more as a</p>
      <p>
        <span class="highlight"><strong>commodity</strong></span> than a
        <span class="highlight"><strong>human right</strong></span>
      </p>
    </div>
  </div>

  <!-- CALL TO ACTION -->
  <div
    style="height: 100vh; display: flex; align-items: center; gap: 20px; text-align: center; font-size: 30px"
  >
    <div data-aos="flip-down" data-aos-delay="100">
      <p>
        The eviction crisis isn’t just numbers on a map — it's a mirror
        reflecting a city at a crossroads.
      </p>
      <p>Boston faces two choices:</p>
      <p>Will we allow profit to eclipse people?</p>
      <p>
        Or will we <span class="highlight"
          ><strong>fight to rebuild</strong></span
        > a city where stability, dignity, and homeownership are within reach for
        everyone?
      </p>
    </div>
    <img
      src={images.fight}
      alt="Weighing people being evicted with profit"
      style="max-width: 500px; width: 500px; object-fit: cover; border-radius: 15%;"
    />
  </div>

  <div
    style="height: 310vh; align-items: center; gap: 20px; text-align: center;"
  >
    <div
      style="position: relative; height: 110vh; display: flex; flex-direction: column; align-items: center; gap: 20px; text-align: center; font-size:30px;"
    >
      <!-- Main centered image -->
      <img
        src={images.bostonArt}
        alt="Watercolor painting of Boston skyline"
        style="max-width: 800px; width: 800px; object-fit: cover; border-radius: 15%;"
      />

      <!-- Paragraphs -->
      <div>
        <p data-aos="zoom-in" data-aos-delay="200">
          <span class="highlight"
            ><strong>It's not too late -- but action is urgent</strong></span
          >
        </p>
        <p>
          A home should be a place to dream, not a battleground for survival.
        </p>
        <p>The future of Boston depends on it.</p>
      </div>
    </div>
    <hr />
    <!-- Actionables -->
    <div
      style="position: relative; height: 105vh; flex-direction: column; align-items: center; gap: 20px; text-align: left; font-size:30px; padding-left:70px"
    >
      <h2 style="text-align: center">What can you do?</h2>
      <h3 data-aos="fade-up" data-aos-delay="200">
        <span>
          <strong>Advocate for stronger tenant protections.</strong>
        </span>
      </h3>
      <p data-aos="fade-up" data-aos-delay="200">
        <a href="https://www.clvu.org" target="_blank" style="color: #1D3557;"
          >Join City Life/Vida Urbana</a
        > – a grassroots group fighting evictions in Boston.
      </p>
      <p data-aos="fade-up" data-aos-delay="200">
        <a
          href="https://www.boston.gov/departments/city-council"
          target="_blank"
          style="color: #1D3557;">Contact your City Councilor</a
        > to support rent stabilization and right-to-counsel legislation.
      </p>

      <h3 data-aos="fade-up" data-aos-delay="200">
        <span>
          <strong>Support affordable housing initiatives.</strong>
        </span>
      </h3>
      <p data-aos="fade-up" data-aos-delay="200">
        <a
          href="https://www.bostonhousing.org"
          target="_blank"
          style="color: #1D3557;">Volunteer with Boston Housing Authority</a
        > to support low-income tenants.
      </p>
      <p data-aos="fade-up" data-aos-delay="200">
        <a
          href="https://www.mahahome.org"
          target="_blank"
          style="color: #1D3557;">Support MAHA</a
        > to expand affordable homeownership opportunities in Boston.
      </p>

      <h3 data-aos="fade-up" data-aos-delay="200">
        <span>
          <strong>Hold corporate landlords accountable.</strong>
        </span>
      </h3>
      <p data-aos="fade-up" data-aos-delay="200">
        Support transparency efforts like rental registries and <em
          >Just Cause Eviction</em
        > legislation.
      </p>
      <div style="justify-content: center; display: flex;">
        <img
          src={images.bost}
          alt="Boston skyline line art "
          style="max-width: 55%; width: 100%; object-fit: cover; margin-top: -150px"
        />
      </div>
    </div>
    <!-- Ending -->
    <div
      data-aos="flip-left"
      data-aos-delay="400"
      style="height: 80vh; display: flex; flex-direction: column; align-items: center; gap: 20px; text-align: center; font-size:30px; margin-left: 50px; padding-bottom: 20px; background: #FFF8DC; border: 5px solid #d5912f;"
    >
      <h2>A Big Thank You To..!</h2>
      <p style="width: 60%">
        This project was developed with guidance and feedback from the
        <a href="https://www.mapc.org/">Metropolitan Area Planning Commission</a
        > (MAPC).
      </p>
      <p style="width: 60%">
        We are also grateful to the MIT Vis & Society staffs and students that
        provide valuable feedback.
      </p>
      <p style="width: 80%">
        Data source: MAPC, USA IPUMS, US Census, Property Assessment Data for
        the City of Boston, Boston Neighborhood Boundaries Approximated by 2020
        Census Block Groups, Block Group Census for Boston.
      </p>
    </div>
  </div>
</body>

<style>
  .highlight {
    color: #d5912f;
  }
  .red {
    color: #8b0000;
  }
  .caption {
    background-color: #f8f1e5;
    width: 90%;
    outline: 2px solid black;
    padding-left: 2em;
    padding-right: 2em;
  }
  /* .visual_caption {
    font-size: 24px;
    padding-left: 1em;
    padding-right: 1em;
  } */
  .visual_caption_text {
    margin-bottom: 0;
  }
  .navbar-progress {
    position: fixed;
    top: 0;
    left: 1%;
    height: 5px;
    background-color: #d5912f;
    z-index: 2000;
    transition: width 0.1s ease-out;
  }
  nav {
    display: flex;
    justify-content: center;
    align-items: center;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    background-color: rgba(255, 255, 255, 0.95);
    z-index: 1000;
    padding: 3px;
  }

  nav ul {
    list-style-type: none;
    display: flex;
    gap: 1rem;
    padding: 10px 20px;
    margin: 0;
  }

  nav ul li a {
    text-decoration: none;
    color: black;
    padding: 8px 12px;
    transition:
      color 0.2s,
      background-color 0.2s;
  }

  nav ul li a:hover {
    color: #d5912f;
  }

  nav ul li a.active {
    color: #d5912f;
    font-weight: bold;
    border-bottom: 3px solid #d5912f;
  }

  * {
    /* outline: 1px solid red; */
    box-sizing: border-box;
  }
  :global(html) {
    overflow-x: clip;
  }
  body {
    color: #000000;
    font-family: "Libre Franklin", sans-serif;
  }

  h1,
  h2,
  h3 {
    color: #d5912f;
    font-family: "Merriweather", serif;
  }

  a {
    color: #1d3557;
  }

  hr {
    border-color: #d5912f;
  }
  body::before {
    content: "";
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    width: auto;
    height: auto;
    background-image: url("/images/houseBackground.jpeg");
    background-repeat: repeat;
    background-size: 400px 400px;
    opacity: 0.05;
    z-index: -1;
    pointer-events: none;
  }
  .walker {
    position: fixed;
    bottom: 10%;
    left: 10%;
    width: 150px;
    transition: transform 0.1s linear;
    pointer-events: none;
    z-index: 10;
  }
  .viz2 {
    position: fixed;
    bottom: 1%;
    left: 5%;
    width: 90%;
    display: flex;
    flex-direction: column;
    justify-content: flex-end;
    align-items: center;
    text-align: center;
    pointer-events: none;
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
    border-radius: 50%;
  }

  .scrolling-text {
    height: 200vh;
    width: 60%;
    margin: 0 auto;
    padding-top: 50vh;
    display: flex;
    flex-direction: column;
    gap: 3em;
    font-size: 30px;
    text-align: center;
  }

  .corp-text {
    position: relative;
    display: flex;
    flex: 1;
    justify-content: center;
  }

  .corp-text .visual_caption {
    font-size: 100%;
    background-color: #f8f1e5;
    width: 95%;
    outline: 2px solid black;
    display: flex;
    justify-content: center;
    /* height: 70%; */
    flex-direction: column;
    margin-top: 30px;
    margin-bottom: 3%;
    text-align: center;
  }

  .small-corp-text {
    /* font-size: medium; */
    font-size: 75%;
    /* position: fixed; */
    /* margin-bottom: 20px */
  }
</style>
