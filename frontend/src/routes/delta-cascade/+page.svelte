<script lang="ts">
  import ChartWrapper from '$lib/components/ChartWrapper.svelte';
  import { onMount } from 'svelte';

  let data: any = null;
  let loading = true;

  onMount(async () => {
    const res = await fetch('http://localhost:8000/api/delta-cascade');
    data = await res.json();
    loading = false;
  });
</script>

<ChartWrapper title="Delta Cascade • SPY">
  {#if loading}
    <div class="text-center py-40 text-3xl opacity-40">Loading live cascade...</div>
  {:else}
    <div class="bg-surface/50 backdrop-blur-xl rounded-3xl border border-border p-12 overflow-x-auto">
      <svg width="1400" height="740" viewBox="0 0 1400 740" class="min-w-[1200px]">
        {#each data.data as row, i}
          {@const opacity = Math.max(0.2, 1 - row.days_ahead * 0.11)}
          {@const color = row.net_delta > 0 ? '#3fb950' : '#f85149'}
          {@const x = 140 + (row.strike - (data.spot - 70)) * 4.5}
          {@const y = 370 - row.net_delta / 7e6 * 260}

          <circle cx={x} cy={y} r="4.5" fill={color} opacity={opacity} />
          {#if i > 0 && data.data[i-1].days_ahead === row.days_ahead}
            {@const x1 = 140 + (data.data[i-1].strike - (data.spot - 70)) * 4.5}
            {@const y1 = 370 - data.data[i-1].net_delta / 7e6 * 260}
            <line x1={x1} y1={y1} x2={x} y2={y} stroke={color} stroke-width="3" opacity={opacity} />
          {/if}
        {/each}
        {@const spotX = 140 + (data.spot - (data.spot - 70)) * 4.5}
        <line x1={spotX} y1="70" x2={spotX} y2="670" stroke="#58a6ff" stroke-width="6" stroke-dasharray="12,10"/>
        <text x="700" y="50" class="fill-primary font-bold text-2xl">SPY ≈ ${Number(data.spot).toFixed(2)}</text>
      </svg>
    </div>
  {/if}
</ChartWrapper>
