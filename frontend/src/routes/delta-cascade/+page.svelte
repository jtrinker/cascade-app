<script lang="ts">
  import { onMount } from 'svelte';

  type Point = [number, number, number]; // [days_ahead, strike, net_delta_in_millions]

  let data: { spot: number; data: Point[] } | null = null;
  let loading = true;

  onMount(async () => {
    const res = await fetch('http://127.0.0.1:8000/api/delta-cascade');
    data = await res.json();
    loading = false;
  });
</script>

<svelte:head>
  <title>Delta Cascade — Live Dealer Net Delta</title>
</svelte:head>

<div class="min-h-screen bg-[#0d1117] text-white flex flex-col items-center pt-12 pb-20">
  <h1 class="text-6xl md:text-7xl font-black text-cyan-400 mb-4">Delta Cascade</h1>
  <p class="text-xl text-gray-400 mb-12">True Dealer Net Delta + 7-Day Decay • Live SPY</p>

  {#if loading}
    <div class="text-2xl text-gray-500">Loading live options chain...</div>
  {:else if data}
    {@const spot = data.spot}
    {@const points = data.data}

    <div class="w-full max-w-7xl bg-gray-950/90 backdrop-blur-xl rounded-3xl border border-gray-800 shadow-2xl overflow-hidden">
      <div class="text-center py-6 bg-gray-900/70 border-b border-gray-800">
        <div class="text-mb-1 text-4xl font-bold text-cyan-400">SPY ${spot.toFixed(2)}</div>
      </div>

      <div class="p-8">
        <svg viewBox="0 0 1600 900" class="w-full">
          <rect width="1600" height="900" fill="#0d1117" />

          <!-- Strike labels -->
          {#each Array(17) as _, i}
            {@const strike = Math.round(spot - 80 + i * 10)}
            <text x={120 + i * 90} y="860" fill="#6b7280" font-size="16" text-anchor="middle">
              ${strike}
            </text>
          {/each}

          <!-- Zero line -->
          <line x1="100" y1="450" x2="1500" y2="450" stroke="#374151" stroke-width="1" />

          <!-- One line per day -->
          {#each Array(8) as _, day}
            {@const dayPoints = points.filter(p => p[0] === day)}
            {@const opacity = Math.max(0.15, 1 - day * 0.12)}
            {@const width = day === 0 ? 9 : Math.max(2, 8 - day)}
            {@const color = day === 0 ? "#00ffcc" : "#a78bfa"}

            {#if dayPoints.length > 0}
              <path
                d={`
                  M 800,450
                  ${dayPoints
                    .sort((a, b) => a[1] - b[1])
                    .map(p => {
                      const x = 120 + ((p[1] - (spot - 80)) / 160) * 1360;
                      const y = 450 - p[2] * 6;   // ← proper scaling
                      return `L ${x},${y}`;
                    })
                    .join(' ')}
                `}
                fill="none"
                stroke={color}
                stroke-width={width}
                stroke-linecap="round"
                opacity={opacity}
              />
            {/if}
          {/each}

          <!-- Spot line -->
          <line x1="800" y1="80" x2="800" y2="820" stroke="#00ffcc" stroke-width="4" />
          <text x="800" y="70" fill="#00ffcc" font-size="20" text-anchor="middle" font-weight="bold">
            SPOT
          </text>
        </svg>

        <div class="text-center mt-10 text-gray-400 text-lg">
          Day 0 = <span class="text-cyan-400 font-bold">thick cyan</span> → Day 7 = faded purple
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  :global(body, html) {
    background: #0d1117;
    margin: 0;
    padding: 0;
  }
</style>