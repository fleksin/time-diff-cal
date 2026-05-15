<script lang="ts">
  import DialKnob from "$lib/DialKnob.svelte";

  function pad(n: number) {
    return String(n).padStart(2, "0");
  }

  function parseT(t: string): { h: number; m: number } | null {
    if (!t) return null;
    const parts = t.split(":");
    if (parts.length !== 2) return null;
    const h = parseInt(parts[0]!, 10);
    const m = parseInt(parts[1]!, 10);
    if (isNaN(h) || isNaN(m) || h < 0 || h > 23 || m < 0 || m > 59) return null;
    return { h, m };
  }

  function calcEnd(start: string, addMins: number): string {
    const t = parseT(start);
    if (!t) return "";
    const total = ((t.h * 60 + t.m + addMins) % (24 * 60) + 24 * 60) % (24 * 60);
    return `${pad(Math.floor(total / 60))}:${pad(total % 60)}`;
  }

  function calcDiff(from: string, to: string): number {
    const a = parseT(from);
    const b = parseT(to);
    if (!a || !b) return 0;
    let d = b.h * 60 + b.m - (a.h * 60 + a.m);
    if (d < 0) d += 24 * 60;
    return d;
  }

  function fmtDuration(totalMins: number): string {
    const h = Math.floor(totalMins / 60);
    const m = totalMins % 60;
    if (h === 0) return `${m} 分钟`;
    if (m === 0) return `${h} 小时`;
    return `${h} 小时 ${m} 分钟`;
  }

  let initTime = new Date().getHours() + ':' + new Date().getMinutes();

  let startTime = $state(initTime);
  let durationMins = $state(120);
  let endTime = $state(calcEnd(startTime, durationMins));
  let endManual = $state(false);

  let crossesMidnight = $derived.by(() => {
    const t = parseT(startTime);
    return t ? t.h * 60 + t.m + durationMins >= 24 * 60 : false;
  });

  function onStartInput(e: Event) {
    const val = (e.target as HTMLInputElement).value;
    startTime = val;
    if (endManual) {
      durationMins = calcDiff(val, endTime);
    } else {
      endTime = calcEnd(val, durationMins);
    }
  }

  function onEndInput(e: Event) {
    const val = (e.target as HTMLInputElement).value;
    endTime = val;
    endManual = true;
    durationMins = calcDiff(startTime, val);
  }

  function setDuration(mins: number) {
    durationMins = mins;
    endManual = false;
    endTime = calcEnd(startTime, mins);
  }

  const PRESETS = [
    { label: "30分", mins: 30 },
    { label: "1小时", mins: 60 },
    { label: "1.5小时", mins: 90 },
    { label: "2小时", mins: 120 },
    { label: "4小时", mins: 240 },
    { label: "8小时", mins: 480 },
    { label: "12小时", mins: 720 },
    { label: "24小时", mins: 1440 },
  ];
</script>

<div class="min-h-screen bg-gradient-to-br from-slate-50 to-blue-50 flex flex-col items-center justify-start py-8 px-4">
  <div class="w-full max-w-sm">

    <!-- Header -->
    <div class="text-center mb-6">
      <h1 class="text-3xl font-bold text-slate-800 tracking-tight">差时计算器</h1>
      <p class="text-slate-400 text-sm mt-1">转动转盘 · 计算时间差</p>
    </div>

    <!-- Time inputs card -->
    <div class="bg-white rounded-2xl shadow-lg shadow-blue-100/40 border border-slate-100 p-5 mb-4">
      <div class="flex items-center gap-3">

        <!-- Start time -->
        <div class="flex-1">
          <div class="text-[10px] font-semibold text-slate-400 uppercase tracking-widest mb-1.5">
            开始时间
          </div>
          <div class="rounded-xl border-2 border-blue-200 bg-blue-50 focus-within:border-blue-400 transition-colors">
            <input
              type="time"
              value={startTime}
              oninput={onStartInput}
              class="w-full bg-transparent text-slate-800 text-lg font-semibold px-3 py-2.5 outline-none"
            />
          </div>
        </div>

        <!-- Middle badge -->
        <div class="flex flex-col items-center pt-5 min-w-[52px]">
          <span class="text-slate-300 text-base leading-none">→</span>
          <div class="mt-1.5 bg-blue-600 text-white text-[10px] font-bold px-2 py-0.5 rounded-full whitespace-nowrap">
            {fmtDuration(durationMins)}
          </div>
        </div>

        <!-- End time -->
        <div class="flex-1">
          <div class="text-[10px] font-semibold text-slate-400 uppercase tracking-widest mb-1.5">
            结束时间
          </div>
          <div class="rounded-xl border-2 transition-colors {endManual ? 'border-emerald-300 bg-emerald-50 focus-within:border-emerald-400' : 'border-slate-200 bg-slate-50 focus-within:border-blue-300'}">
            <input
              type="time"
              value={endTime}
              oninput={onEndInput}
              class="w-full bg-transparent text-slate-800 text-lg font-semibold px-3 py-2.5 outline-none"
            />
          </div>
          {#if crossesMidnight}
            <p class="text-amber-500 text-[10px] mt-1 text-center">跨越午夜</p>
          {/if}
        </div>

      </div>
    </div>

    <!-- Dial card -->
    <div class="bg-white rounded-2xl shadow-lg shadow-blue-100/40 border border-slate-100 p-5 mb-4">
      <div class="text-[10px] font-semibold text-slate-400 uppercase tracking-widest mb-3 text-center">
        时长转盘
      </div>
      <DialKnob value={durationMins} max={2880} onchange={setDuration} />
    </div>

    <!-- Presets -->
    <div class="bg-white rounded-2xl shadow-lg shadow-blue-100/40 border border-slate-100 p-4 mb-4">
      <div class="flex flex-wrap gap-2 justify-center">
        {#each PRESETS as p}
          <button
            onclick={() => setDuration(p.mins)}
            class="text-xs px-3 py-1.5 rounded-lg border font-medium transition-all {durationMins === p.mins
              ? 'bg-blue-600 text-white border-blue-600 shadow-sm shadow-blue-200'
              : 'bg-slate-50 text-slate-600 border-slate-200 hover:border-blue-300 hover:text-blue-600 hover:bg-blue-50'}"
          >
            {p.label}
          </button>
        {/each}
      </div>
    </div>

    <!-- Result summary -->
    <div class="bg-blue-600 rounded-2xl p-5 text-white shadow-lg shadow-blue-300/40">
      <div class="flex items-center justify-between">
        <div>
          <p class="text-blue-200 text-[10px] font-semibold uppercase tracking-widest mb-1">计算结果</p>
          <p class="text-2xl font-bold">{fmtDuration(durationMins)}</p>
          <p class="text-blue-200 text-sm mt-0.5">
            {startTime} → {endTime}{crossesMidnight ? "（次日）" : ""}
          </p>
        </div>
        <div class="text-right">
          <p class="text-blue-200 text-[10px] font-semibold uppercase tracking-widest mb-1">合计分钟</p>
          <p class="text-3xl font-bold font-mono">{durationMins}</p>
          <p class="text-blue-200 text-xs">分钟</p>
        </div>
      </div>
    </div>

    <p class="text-center text-xs text-slate-400 mt-4">
      拖动转盘选择时长 · 或手动填写结束时间反算
    </p>

  </div>
</div>
