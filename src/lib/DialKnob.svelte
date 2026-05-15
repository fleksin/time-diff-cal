<script lang="ts">
  interface Props {
    value: number;
    max?: number;
    min?: number;
    onchange?: (v: number) => void;
  }

  let { value, max = 2880, min = 0, onchange }: Props = $props();

  const CX = 150, CY = 155, R = 100, TW = 14;
  const SD = 225, SPAN = 270;

  function xy(deg: number, r = R): [number, number] {
    const rad = (deg * Math.PI) / 180;
    return [CX + r * Math.sin(rad), CY - r * Math.cos(rad)];
  }

  function arcPath(fromDeg: number, spanDeg: number, r = R): string {
    if (spanDeg <= 0) return "";
    const [x1, y1] = xy(fromDeg, r);
    const [x2, y2] = xy(fromDeg + spanDeg, r);
    const large = spanDeg > 180 ? 1 : 0;
    return `M ${x1.toFixed(2)} ${y1.toFixed(2)} A ${r} ${r} 0 ${large} 1 ${x2.toFixed(2)} ${y2.toFixed(2)}`;
  }

  let frac = $derived(Math.max(0, Math.min(1, (value - min) / (max - min))));
  let activeDeg = $derived(SD + frac * SPAN);
  let hx = $derived(xy(activeDeg)[0]);
  let hy = $derived(xy(activeDeg)[1]);
  let bgArc = $derived(arcPath(SD, SPAN));
  let fgArc = $derived(frac > 0.002 ? arcPath(SD, frac * SPAN) : "");

  let hours = $derived(Math.floor(value / 60));
  let mins = $derived(value % 60);

  const majorTicks = [0, 6, 12, 18, 24, 30, 36, 42, 48].map((hr) => {
    const deg = SD + (hr / 48) * SPAN;
    const [x1, y1] = xy(deg, R - TW / 2 - 7);
    const [x2, y2] = xy(deg, R + TW / 2 + 7);
    const [lx, ly] = xy(deg, R + TW / 2 + 23);
    return { hr, x1, y1, x2, y2, lx, ly };
  });

  const minorTicks = [3, 9, 15, 21, 27, 33, 39, 45].map((hr) => {
    const deg = SD + (hr / 48) * SPAN;
    const [x1, y1] = xy(deg, R - TW / 2 - 4);
    const [x2, y2] = xy(deg, R + TW / 2 + 4);
    return { x1, y1, x2, y2 };
  });

  let svgEl: SVGSVGElement;
  let dragging = $state(false);

  function getAngle(e: PointerEvent): number {
    const rect = svgEl.getBoundingClientRect();
    const sc = 300 / rect.width;
    const dx = (e.clientX - rect.left) * sc - CX;
    const dy = CY - (e.clientY - rect.top) * sc;
    let deg = Math.atan2(dx, dy) * (180 / Math.PI);
    return deg < 0 ? deg + 360 : deg;
  }

  function angleToValue(deg: number): number {
    let rel = deg - SD;
    if (rel < -20) rel += 360;
    rel = Math.max(0, Math.min(SPAN, rel));
    return Math.round(min + (rel / SPAN) * (max - min));
  }

  function ondown(e: PointerEvent) {
    e.preventDefault();
    dragging = true;
    svgEl.setPointerCapture(e.pointerId);
    const v = angleToValue(getAngle(e));
    onchange?.(v);
  }

  function onmove(e: PointerEvent) {
    if (!dragging) return;
    const v = angleToValue(getAngle(e));
    onchange?.(v);
  }

  function onup(_e: PointerEvent) {
    dragging = false;
  }
</script>

<svg
  bind:this={svgEl}
  viewBox="0 0 300 310"
  class="w-full max-w-[300px] mx-auto select-none touch-none"
  style="cursor: {dragging ? 'grabbing' : 'grab'}"
  onpointerdown={ondown}
  onpointermove={onmove}
  onpointerup={onup}
  onpointerleave={onup}
  role="slider"
  tabindex="0"
  aria-valuemin={min}
  aria-valuemax={max}
  aria-valuenow={value}
>
  <defs>
    <filter id="handle-shadow">
      <feDropShadow dx="0" dy="2" stdDeviation="4" flood-color="rgba(59,130,246,0.45)" />
    </filter>
    <filter id="arc-glow">
      <feDropShadow dx="0" dy="0" stdDeviation="3" flood-color="rgba(59,130,246,0.5)" />
    </filter>
  </defs>

  <!-- Background track -->
  <path
    d={bgArc}
    fill="none"
    stroke="#e2e8f0"
    stroke-width={TW}
    stroke-linecap="round"
  />

  <!-- Active track -->
  {#if fgArc}
    <path
      d={fgArc}
      fill="none"
      stroke="#3b82f6"
      stroke-width={TW}
      stroke-linecap="round"
      filter="url(#arc-glow)"
    />
  {/if}

  <!-- Minor ticks -->
  {#each minorTicks as t}
    <line
      x1={t.x1} y1={t.y1} x2={t.x2} y2={t.y2}
      stroke="#cbd5e1"
      stroke-width="1.5"
      stroke-linecap="round"
    />
  {/each}

  <!-- Major ticks + labels -->
  {#each majorTicks as t}
    <line
      x1={t.x1} y1={t.y1} x2={t.x2} y2={t.y2}
      stroke="#94a3b8"
      stroke-width="2"
      stroke-linecap="round"
    />
    <text
      x={t.lx}
      y={t.ly + 4}
      text-anchor="middle"
      font-size="10"
      font-family="ui-monospace, monospace"
      fill="#94a3b8"
    >{t.hr}h</text>
  {/each}

  <!-- Handle -->
  <circle
    cx={hx}
    cy={hy}
    r="13"
    fill="white"
    stroke="#3b82f6"
    stroke-width="3"
    filter="url(#handle-shadow)"
  />
  <circle cx={hx} cy={hy} r="4.5" fill="#3b82f6" />

  <!-- Center: hour number -->
  <text
    x={CX}
    y={CY - 18}
    text-anchor="middle"
    font-size="52"
    font-weight="700"
    fill="#0f172a"
    font-family="Inter, system-ui, sans-serif"
  >{hours}</text>

  <!-- "h" unit -->
  <text
    x={CX + 36}
    y={CY - 24}
    text-anchor="start"
    font-size="18"
    fill="#64748b"
    font-family="Inter, system-ui, sans-serif"
  >h</text>

  <!-- Minutes -->
  {#if mins > 0}
    <text
      x={CX}
      y={CY + 16}
      text-anchor="middle"
      font-size="22"
      fill="#475569"
      font-family="ui-monospace, monospace"
    >{String(mins).padStart(2, "0")}m</text>
  {/if}

  <!-- Label -->
  <text
    x={CX}
    y={CY + (mins > 0 ? 46 : 26)}
    text-anchor="middle"
    font-size="11"
    fill="#94a3b8"
    font-family="Inter, system-ui, sans-serif"
    letter-spacing="3"
  >时长</text>
</svg>
