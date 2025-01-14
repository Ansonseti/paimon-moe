<script>
  import { t } from 'svelte-i18n';

  export let wishTotal = {
    'character-event': 0,
    'weapon-event': 0,
    standard: 0,
  };
  export let wishPercentage;

  let current = 'character-event';
  let wishCount = 0;
  let total = 0;
  let data = [];
  let percentage = 0;
  let median = '...';
  let percentageLuck = {
    legendary: 0,
    rare: 0,
  };
  let medianLuck = {
    legendary: '...',
    rare: '...',
  };
  let loading = {
    total: true,
    legendary: true,
    rare: true,
  };

  export async function getData() {
    loading.total = true;
    percentage = 0;
    median = '...';
    wishCount = wishTotal[current];

    try {
      const url = new URL(`${__paimon.env.API_HOST}/wish/summary`);
      const query = new URLSearchParams({ banner: current });
      url.search = query.toString();

      const res = await fetch(url, {
        method: 'GET',
        headers: { 'Content-Type': 'application/json' },
      });

      data = await res.json();

      total = 0;
      const sorted = [];
      for (const item of data) {
        total += item[1];
        sorted.push(...new Array(item[1]).fill(item[0]));
      }
      median = sorted[Math.round(sorted.length / 2)];

      getPercentile(wishCount);
    } catch (err) {
      console.error(err);
    }
  }

  function getPercentile(value) {
    let totalLower = 0;
    for (const item of data) {
      const qty = item[0];
      const amount = item[1];
      totalLower += amount;

      if (qty >= value) break;
    }

    percentage = 100 - (totalLower / total) * 100;
    console.log(totalLower, percentage, value);

    loading.total = false;
  }

  async function getDataLuck(rarity, percentages) {
    loading[rarity] = true;
    percentageLuck[rarity] = 0;
    medianLuck[rarity] = '...';

    try {
      const url = new URL(`${__paimon.env.API_HOST}/wish/summary/luck`);
      const query = new URLSearchParams({ banner: current, rarity });
      url.search = query.toString();

      const res = await fetch(url, {
        method: 'GET',
        headers: { 'Content-Type': 'application/json' },
      });

      data = await res.json();

      total = 0;
      const sorted = [];
      for (const item of data) {
        total += item[1];
        sorted.push(...new Array(item[1]).fill(item[0]));
      }
      medianLuck[rarity] = (sorted[Math.round(sorted.length / 2)] * 100).toFixed(2);

      getPercentileLuck(percentages[current][rarity], rarity);
    } catch (err) {
      console.error(err);
    }
  }

  function getPercentileLuck(value, rarity) {
    let totalLower = 0;
    for (const item of data) {
      const qty = item[0];
      const amount = item[1];
      totalLower += amount;

      if (qty >= value) break;
    }

    percentageLuck[rarity] = 100 - (totalLower / total) * 100;
    console.log(totalLower, percentage, value);

    loading[rarity] = false;
  }

  export function getDataLuckAll(percentages) {
    const sources = percentages === undefined ? wishPercentage : percentages;

    getDataLuck('legendary', sources);
    getDataLuck('rare', sources);
  }

  function change(type) {
    current = type;
    getData();
    getDataLuckAll();
  }

  $: fixedPoint = percentage < 0.1 ? 2 : percentage < 2 ? 1 : 0;
  $: fixedPointLegendary = percentageLuck.legendary < 0.1 ? 2 : percentageLuck.legendary < 2 ? 1 : 0;
  $: fixedPointRare = percentageLuck.rare < 0.1 ? 2 : percentageLuck.rare < 2 ? 1 : 0;
</script>

<div class="flex flex-col items-center bg-item rounded-xl p-4 w-full mt-4">
  <div class="flex flex-col items-center pb-4">
    <p class="text-lg text-white leading-none font-semibold">{$t('wish.rank.title')}</p>
    <p class="text-sm text-gray-400 leading-none">{$t('wish.rank.based')}</p>
  </div>
  <div class="bg-background flex-row items-center justify-center mb-2 p-4 rounded-xl flex relative group w-full">
    <div
      class="group-hover:flex hidden absolute left-0 items-center z-10"
      style="max-width: 80%; bottom: 12px; left: 12px;"
    >
      <div class="bg-gray-200 text-gray-900 px-4 py-1 rounded-xl text-sm md:text-base">
        {$t('wish.rank.medianTotal', { values: { median } })}
      </div>
    </div>
    <div class="text-gray-200 flex-1">
      <p class="text-gray-200">{$t('wish.rank.totalPull')}</p>
      <p class="text-sm text-gray-600 leading-none">
        {#if percentage < 50}
          {$t('wish.rank.moreTotal', { values: { percentage: (100 - percentage).toFixed(fixedPoint) } })}
        {:else}
          {$t('wish.rank.lessTotal', { values: { percentage: percentage.toFixed(0) } })}
        {/if}
      </p>
    </div>
    <div class="pl-1">
      <span class="text-sm font-black text-white opacity-75 block leading-none">TOP</span>
      <span class="text-white font-black text-3xl leading-none block" style="line-height: 0.75;">
        {loading.total ? '...' : percentage.toFixed(fixedPoint)}%
      </span>
    </div>
  </div>
  <div class="bg-background flex-row items-center justify-center mb-2 p-4 rounded-xl flex relative group w-full">
    <div
      class="group-hover:flex hidden absolute left-0 items-center z-10"
      style="max-width: 80%; bottom: 12px; left: 12px;"
    >
      <div class="bg-gray-200 text-gray-900 px-4 py-1 rounded-xl text-sm md:text-base">
        {$t('wish.rank.medianLuck', { values: { median: medianLuck.legendary } })}
      </div>
    </div>
    <div class="text-gray-200 flex-1">
      <p class="text-gray-200">{$t('wish.rank.luck5')}</p>
      <p class="text-sm text-gray-600 leading-none">
        {#if percentageLuck.legendary < 50}
          {$t('wish.rank.moreLuck', {
            values: { percentage: (100 - percentageLuck.legendary).toFixed(fixedPointLegendary) },
          })}
        {:else}
          {$t('wish.rank.lessLuck', { values: { percentage: percentageLuck.legendary.toFixed(0) } })}
        {/if}
      </p>
    </div>
    <div class="pl-1">
      <span class="text-sm font-black text-legendary-from opacity-75 block leading-none">TOP</span>
      <span class="text-legendary-from font-black text-3xl leading-none block" style="line-height: 0.75;">
        {loading.legendary ? '...' : percentageLuck.legendary.toFixed(fixedPointLegendary)}%
      </span>
    </div>
  </div>
  <div class="bg-background flex-row items-center justify-center p-4 rounded-xl flex relative group w-full">
    <div
      class="group-hover:flex hidden absolute left-0 items-center z-10"
      style="max-width: 80%; bottom: 12px; left: 12px;"
    >
      <div class="bg-gray-200 text-gray-900 px-4 py-1 rounded-xl text-sm md:text-base">
        {$t('wish.rank.medianLuck', { values: { median: medianLuck.rare } })}
      </div>
    </div>
    <div class="text-gray-200 flex-1">
      <p class="text-gray-200">{$t('wish.rank.luck4')}</p>
      <p class="text-sm text-gray-600 leading-none">
        {#if percentageLuck.rare < 50}
          {$t('wish.rank.moreLuck', { values: { percentage: (100 - percentageLuck.rare).toFixed(fixedPointRare) } })}
        {:else}
          {$t('wish.rank.lessLuck', { values: { percentage: percentageLuck.rare.toFixed(0) } })}
        {/if}
      </p>
    </div>
    <div class="pl-1">
      <span class="text-sm font-black text-rare-from opacity-75 block leading-none">TOP</span>
      <span class="text-rare-from font-black text-3xl leading-none block" style="line-height: 0.75;">
        {loading.rare ? '...' : percentageLuck.rare.toFixed(fixedPointRare)}%
      </span>
    </div>
  </div>
  <div class="flex -m-1 pt-4">
    <button class="pill m-1 {current === 'character-event' ? 'active' : ''}" on:click={() => change('character-event')}>
      {$t('wish.detail.character')}
    </button>
    <button class="pill m-1 {current === 'weapon-event' ? 'active' : ''}" on:click={() => change('weapon-event')}>
      {$t('wish.detail.weapon')}
    </button>
    <button class="pill m-1 {current === 'standard' ? 'active' : ''}" on:click={() => change('standard')}>
      {$t('wish.types.standard')}
    </button>
  </div>
</div>

<style>
  .pill {
    @apply text-sm;
    @apply rounded-2xl;
    @apply border-2;
    @apply border-white;
    @apply border-opacity-25;
    @apply text-white;
    @apply px-4;
    @apply py-1;
    @apply outline-none;
    @apply transition;
    @apply duration-100;

    &:hover {
      @apply border-primary;
    }

    &.active {
      @apply bg-primary;
      @apply border-primary;
      @apply text-background;
    }
  }
</style>
