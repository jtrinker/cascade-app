<script lang="ts">
  import { onMount } from 'svelte';
  
  interface Point { days_ahead: number; strike: number; net_delta: number }
  interface Resp { spot: number; data: Point[] }

  let data: Resp | null = null;
  let loading = true;

  onMount(async () => {
    const r = await fetch('http://127.0.0.1:8000/api/delta-cascade');
    data = await r.json();
    loading = false;
  });
</script>

<svelte:head><title>Cascade — Dealer Net Delta</title></svelte:head>

<div class="min-h-screen bg-[#0d1117] text-white">
  <div class="max-w-7xl mx-auto px-4 py-12">
    <h1 class="text-center text-6xl font-black text-cyan-400 mb-4">Cascade</h1>
    <p class="text-center text-gray-400 mb-12">Live Dealer Net Delta Exposure</p>

    {#if loading}
      <div class="text-center text-2xl text-gray-500 py-40">Loading...</div>
    {:else if data}
      {@const spot = data.spot}
      {@const points = data.data}

      {@const minStrike = spot - 70}
      {@const maxStrike = spot + 70}
      {@const x = (s: number) => 100 + ((s - minStrike) / 140) * 1300}
      {@const maxAbs = Math.max(...points.map(p => Math.abs(p.net_delta)))}
      {@const y = (d: number) => 350 - (d / (maxAbs * 1.2)) * 300}

      {@const byDay = Array.from({length:8},(_,i)=> 
        points.filter(p=>p.days_ahead===i).sort((a,b)=>a.strike-b.strike)
      )}

      <!-- Cumulative (yellow line) -->
      {@const map = new Map<number,number>()}
      {#each points as p}
        {@const prev = map.get(p.strike)??0}
        {@const _ = map.set(p.strike, prev + p.net_delta)}
      {/each}
      {@const strikes = [...map.keys()].sort((a,b)=>a-b)}
      {@const cumu = strikes.reduce((a,s)=>(a.push({strike:s, cum:a.length?a[a.length-1].cum + (map.get(s)??0):map.get(s)??0}),a), [] as {strike:number;cum:number}[]) }

      <div class="bg-gray-950/90 backdrop-blur-xl rounded-3xl border border-gray-800 shadow-2xl overflow-hidden">
        <div class="text-center py-6 bg-gray-900/60 border-b border-gray-800">
          <div class="text-4xl font-bold text-cyan-400">SPY ${spot.toFixed(2)}</div>
        </div>

        <div class="p-8">
          <svg viewBox="0 0 1500 750" class="w-full">
            <rect width="1500" height="750" fill="#0d1117"/>

            <defs>
              <linearGradient id="g"><stop offset="0%" stop-color="#10b981" stop-opacity="0.2"/><stop offset="85%" stop-color="#10b981" stop-opacity="0.95"/></linearGradient>
              <linearGradient id="r"><stop offset="0%" stop-color="#ef4444" stop-opacity="0.2"/><stop offset="85%" stop-color="#ef4444" stop-opacity="0.95"/></linearGradient>
            </defs>

            <!-- Grid -->
            {#each Array(15) as _,i}
              <line x1={100+i*100} y1="80" x2={100+i*100} y2="670" stroke="#1f2937"/>
              <text x={100+i*100} y="700" fill="#4b5563" font-size="15" text-anchor="middle">${(minStrike+i*10).toFixed(0)}</text>
            {/each}
            {#each [-0.18,-0.12,-0.06,0,0.06,0.12,0.18] as l,i}
              {@const yy=350+i*100-300}
              <line x1="100" y1={yy} x2="1400" y2={yy} stroke="#1f2937"/>
              <text x="80" y={yy+5} fill="#4b5563" font-size="14" text-anchor="end">{l>0?'+' : ''}{l.toFixed(2)}B</text>
            {/each}

            <line x1="100" y1="350" x2="1400" y2="350" stroke="#60a5fa" stroke-width="2" stroke-dasharray="8,5"/>
            <line x1={x(spot)} y1="80" x2={x(spot)} y2="670" stroke="#60a5fa" stroke-width="4"/>
            <text x={x(spot)} y="70" fill="#60a5fa" font-size="18" text-anchor="middle" font-weight="bold">SPOT</text>

            <!-- SOLID Day-0 mountains -->
            <path d={`M ${x(minStrike)},350 ${byDay[0].filter(p=>p.strike<=spot).map(p=>`L ${x(p.strike)},${y(p.net_delta)}`).join(' ')} L ${x(spot)},350 Z`} fill="url(#g)"/>
            <path d={`M ${x(spot)},350 ${byDay[0].filter(p=>p.strike>=spot).map(p=>`L ${x(p.strike)},${y(p.net_delta)}`).join(' ')} L ${x(maxStrike)},350 Z`} fill="url(#r)"/>

            <!-- Thin decay lines (Days 1–7) -->
            {#each byDay.slice(1) as dayPts, i}
              {@const op = Math.max(0.08, 0.9 - (i+1)*0.12)}
              <path d={`M ${x(spot)},350 ${dayPts.map(p=>`L ${x(p.strike)},${y(p.net_delta)}`).join(' ')}`}
                fill="none"
                stroke={dayPts[0].net_delta>0?'#10b981':'#ef4444'}
                stroke-width="2"
                opacity={op}
                stroke-linecap="round"/>
            {/each}

            <!-- YELLOW CUMULATIVE — NOW IMPOSSIBLE TO MISS -->
            {#if cumu.length}
              <!-- Glow layer -->
              <path d={`M ${x(cumu[0].strike)},${y(0)} ${cumu.map(p=>`L ${x(p.strike)},${y(p.cum)}`).join(' ')}`}
                fill="none" stroke="#fbbf24" stroke-width="20" opacity="0.5"/>
              <!-- Main line -->
              <path d={`M ${x(cumu[0].strike)},${y(0)} ${cumu.map(p=>`L ${x(p.strike)},${y(p.cum)}`).join(' ')}`}
                fill="none" stroke="#fbbf24" stroke-width="6" stroke-linecap="round"/>
            {/if}
          </svg>

          <div class="flex justify-center gap-12 mt-10 text-lg text-gray-300">
            <div class="flex items-center gap-3"><div class="w-8 h-5 bg-gradient-to-t from-emerald-600 to-emerald-400 rounded"></div><span>Dealers Buy on Dip</span></div>
            <div class="flex items-center gap-3"><div class="w-8 h-5 bg-gradient-to-t from-red-600 to-red-400 rounded"></div><span>Dealers Sell on Rip</span></div>
            <div class="flex items-center gap-3"><div class="w-12 h-1.5 bg-yellow-400 rounded"></div><span>Cumulative Gamma Exposure</span></div>
          </div>
        </div>
      </div>
    {/if}
  </div>
</div>

<style>
  :global(body,html){background:#0d1117;margin:0}
</style>