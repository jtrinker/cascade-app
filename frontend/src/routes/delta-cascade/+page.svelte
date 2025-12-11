<script lang="ts">
  import { onMount } from 'svelte';
  import ChartWrapper from '$lib/components/ChartWrapper.svelte';

  let data: any = null;
  let loading = true;

  onMount(async () => {
    const res = await fetch('http://127.0.0.1:8000/api/delta-cascade');
    data = await res.json();
    loading = false;
  });
</script>

<ChartWrapper title="Delta Cascade — LIVE & FLAWLESS">
  {#if loading}
    <div class="text-center py-40 text-2xl text-gray-500">Loading live data...</div>
  {:else if data}
    {@const spot = data.spot}
    {@const maxAbs = Math.max(...data.data.map((d:any) => Math.abs(d.net_delta)))}
    {@const spotX = 120 + ((spot - (spot - 70)) / 140) * 1200}

    <div class="bg-surface/70 backdrop-blur-xl rounded-3xl border border-border p-12 shadow-2xl">
      <svg viewBox="0 0 1400 700" class="w-full">
        <rect width="1400" height="700" fill="#0d1117"/>

        <!-- Grid -->
        {#each Array(15) as _, i}
          {@const x = 120 + i * 80}
          <line x1={x} y1="80" x2={x} y2="620" stroke="#30363d"/>
          <text x={x} y="650" text-anchor="middle" fill="#8b949e" font-size="13">
            ${(spot-70 + i*10).toFixed(0)}
          </text>
        {/each}
        {#each [-3,-2,-1,1,2,3] as i}
          {@const y = 350 + i * -70}
          <line x1="120" y1={y} x2="1320" y2={y} stroke="#30363d"/>
          <text x="100" y={y+5} text-anchor="end" fill="#8b949e" font-size="13">
            {i>0?'+' : ''}${(i*maxAbs/1e9).toFixed(1)}B
          </text>
        {/each}

        <!-- Zero + spot line -->
        <line x1="120" y1="350" x2="1320" y2="350" stroke="#58a6ff" stroke-width="2"/>
        <line x1={spotX} y1="80" x2={spotX} y2="620" stroke="#58a6ff" stroke-width="6" stroke-dasharray="14,10"/>
        <text x={spotX} y="60" text-anchor="middle" fill="#58a6ff" font-size="28" font-weight="900">
          SPY ${spot.toFixed(2)}
        </text>

        <!-- ONE CLEAN PATH PER DAY — green left, red right, perfect visibility -->
        {#each Array(8) as _, day}
          {@const dayPoints = data.data
            .filter((d:any) => d.days_ahead === day)
            .sort((a:any,b:any) => a.strike - b.strike)}
          {@const opacity = Math.max(0.35, 1 - day * 0.09)}
          {@const baseWidth = 8 - day * 0.7}

          <!-- Green part (strikes < spot) -->
          {@const greenPoints = dayPoints.filter((d:any) => d.strike <= spot)}
          {#if greenPoints.length > 1}
            <path
              d={"M" + greenPoints.map((d:any) => {
                const x = 120 + ((d.strike - (spot-70)) / 140) * 1200;
                const y = 350 - (d.net_delta / (maxAbs*1.2)) * 270;
                return `${x},${y}`;
              }).join(' L ')}
              fill="none"
              stroke="#10b981"
              stroke-width={baseWidth}
              opacity={opacity}
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          {/if}

          <!-- Red part (strikes > spot) -->
          {@const redPoints = dayPoints.filter((d:any) => d.strike >= spot)}
          {#if redPoints.length > 1}
            <path
              d={"M" + redPoints.map((d:any) => {
                const x = 120 + ((d.strike - (spot-70)) / 140) * 1200;
                const y = 350 - (d.net_delta / (maxAbs*1.2)) * 270;
                return `${x},${y}`;
              }).join(' L ')}
              fill="none"
              stroke="#ef4444"
              stroke-width={baseWidth}
              opacity={opacity}
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          {/if}
        {/each}

        <!-- Legend -->
        <g transform="translate(920,80)">
          <rect x="0" y="0" width="440" height="100" rx="24" fill="#161b22" opacity="0.95"/>
          <text x="40" y="45" fill="#10b981" font-size="18" font-weight="700">Positive Delta — Dealers Buy on Rise</text>
          <text x="40" y="85" fill="#ef4444" font-size="18" font-weight="700">Negative Delta — Dealers Sell on Rise</text>
        </g>
      </svg>
    </div>
  {/if}
</ChartWrapper>

<style>
  text { font-family: 'Geist Mono', 'JetBrains Mono', monospace; }
</style>