<script lang="ts">
  import knobImg from './Knob.png?url';

  export let value = 120;
  export let max = 2880;
  export let min = 0;
  export let onchange: ((v: number) => void) | undefined;

  const HOUR = 60; // minutes per hour
  const PX_PER_HOUR = 8; // pixels per hour of swipe
  const FRICTION = 0.9; // deceleration per frame
  const MIN_SPEED = 60; // minimum momentum speed in minutes/frame

  // axis constants
  const AXIS_HOUR_STEP = 2; // show a label every 2 hours
  const AXIS_HOUR_RANGE = 48; // total range in hours (max 2880 min = 48 h)
  const HOUR_PX = 48; // pixels per hour on the axis

  let dialEl: HTMLElement;
  let pointerId: number | null = null;

  // momentum
  let momentumRaf: number | null = null;
  let velocity = 0; // minutes per frame
  let lastX = 0;
  let lastTime = 0;

  let pointerStarterX: number | null = null;
  let startValue = 0;

  function clamp(v: number) {
    return Math.max(min, Math.min(max, v));
  }

  function valueToAngle(v: number) {
    const ratio = (clamp(v) - min) / (max - min);
    return 225 + ratio * 270;
  }

  // pixel offset of the current value's tick mark from the start of the axis
  $: axisOffset = (value / HOUR) * HOUR_PX;

  function snapToHour(raw: number) {
    const nearestHour = Math.round(raw / 60) * 60;
    return clamp(nearestHour);
  }

  function stopMomentum() {
    if (momentumRaf !== null) {
      cancelAnimationFrame(momentumRaf);
      momentumRaf = null;
    }
    velocity = 0;
  }

  function startMomentum() {
    stopMomentum();

    function tick() {
      velocity *= FRICTION;
      if (Math.abs(velocity) < MIN_SPEED) {
        const snapped = snapToHour(value);
        onchange?.(snapped);
        velocity = 0;
        momentumRaf = null;
        return;
      }
      const newVal = clamp(value + velocity);
      onchange?.(snapToHour(newVal));
      momentumRaf = requestAnimationFrame(tick);
    }

    momentumRaf = requestAnimationFrame(tick);
  }

  function handlePointerDown(event: PointerEvent) {
    event.preventDefault();
    stopMomentum();
    pointerStarterX = event.clientX;
    pointerId = event.pointerId;
    dialEl.setPointerCapture(pointerId);
    lastX = event.clientX;
    lastTime = performance.now();
    startValue = value;
  }

  function handlePointerMove(event: PointerEvent) {
    const scale = 4;
    if (pointerStarterX === null || event.pointerId !== pointerId) return;
    const now = performance.now();
    let dx = event.clientX - pointerStarterX;
    dx = dx / scale;
    const ddx = event.clientX - lastX;

    const dt = Math.max(now - lastTime, 5);
    velocity = (ddx / PX_PER_HOUR) * HOUR / (dt / 16.67);

    lastX = event.clientX;
    lastTime = now;

    // swipe from right to left (dx negative) => value increases
    let raw = startValue - (dx / PX_PER_HOUR) * HOUR;
    onchange?.(clamp(Math.round(raw)));
  }

  function handlePointerUp(event: PointerEvent) {
    if (event.pointerId !== pointerId) return;
    pointerStarterX = null;
    dialEl.releasePointerCapture(pointerId);
    pointerId = null;

    if (Math.abs(velocity) >= MIN_SPEED) {
      startMomentum();
    } else {
      const snapped = snapToHour(value);
      onchange?.(snapped);
      velocity = 0;
    }
  }

  // pre-build axis tick marks
  let tickMarks: Array<{hour: number; major: boolean}> = [];
  for (let h = 0; h <= AXIS_HOUR_RANGE; h += 1) {
    tickMarks.push({ hour: h, major: h % AXIS_HOUR_STEP === 0 });
  }

  // current hour (for highlighting the active tick)
  $: currentHour = Math.round(value / HOUR);
  $: dialAngle = valueToAngle(value);
</script>

<style>
  .dial-wrapper {
    position: relative;
    width: 100%;
    max-width: 320px;
    margin: 0 auto;
    touch-action: none;
  }

  /* -------- Horizontal axis above knob -------- */
  .axis-wrapper {
    position: relative;
    width: 100%;
    height: 36px;
    margin-bottom: 0.5rem;
    overflow: hidden;
  }

  .axis-track {
    position: absolute;
    left: 50%;
    top: 0;
    height: 100%;
    transform: translateX(-50%) translateX(calc(-1 * var(--axis-scroll)));
    transition: transform 80ms ease-out;
    will-change: transform;
  }

  .axis-tick {
    position: absolute;
    top: 8px;
    width: 1px;
    height: 10px;
    background: rgba(15, 23, 42, 0.12);
  }

  .axis-tick.major {
    height: 16px;
    background: rgba(15, 23, 42, 0.28);
    top: 2px;
  }

  .axis-tick.active {
    background: #2563eb;
    width: 2px;
    height: 24px;
    top: -2px;
  }

  .axis-label {
    position: absolute;
    top: 22px;
    font-size: 0.6rem;
    color: #94a3b8;
    white-space: nowrap;
    font-variant-numeric: tabular-nums;
    transform: translateX(-50%);
    user-select: none;
  }

  .axis-label.active {
    color: #2563eb;
    font-weight: 700;
    font-size: 0.75rem;
  }

  .axis-indicator {
    position: absolute;
    left: 50%;
    top: 0;
    width: 2px;
    height: 100%;
    background: #2563eb;
    transform: translateX(-50%);
    pointer-events: none;
    z-index: 2;
  }

  .axis-indicator::after {
    content: '';
    position: absolute;
    left: 50%;
    bottom: -4px;
    width: 8px;
    height: 8px;
    background: #2563eb;
    border-radius: 50%;
    transform: translateX(-50%);
  }

  /* current value badge above indicator */
  .axis-value-badge {
    position: absolute;
    left: 50%;
    top: -2px;
    transform: translateX(-50%) translateY(-100%);
    background: #2563eb;
    color: white;
    font-size: 0.65rem;
    font-weight: 700;
    padding: 1px 6px;
    border-radius: 4px;
    white-space: nowrap;
    pointer-events: none;
    z-index: 3;
  }

  /* -------- Knob -------- */
  .dial-root {
    position: relative;
    width: 100%;
    padding-top: 100%;
    margin-bottom: -70%;
  }

  .dial-inner {
    position: absolute;
    inset: 0;
    border-radius: 50%;
    background: radial-gradient(circle at 50% 40%, #f8fafc, #e2e8f0 45%, #cbd5e1 100%);
    box-shadow: inset 0 12px 24px rgba(15, 23, 42, 0.08), 0 12px 30px rgba(15, 23, 42, 0.05);
    overflow: hidden;
    clip-path: inset(0 0 70% 0);
  }

  /* bottom fade overlay for smooth blending into white background */
  .dial-inner::after {
    content: '';
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    height: 30%;
    top: 0;
    background: linear-gradient(to top, rgba(255,255,255,1) 0%, rgba(255,255,255,0) 100%);
    pointer-events: none;
  }

  .dial-ring {
    position: absolute;
    inset: 0;
    display: grid;
    place-items: center;
    transform: rotate(var(--dial-rotate));
    transition: transform 80ms ease-out;
  }

  .dial-surface {
    width: 88%;
    height: 88%;
    border-radius: 50%;
    background-color: white;
    background-size: contain;
    background-repeat: no-repeat;
    background-position: center;
    border: 1px solid rgba(148, 163, 184, 0.24);
    box-shadow: inset 0 8px 16px rgba(15, 23, 42, 0.08);
    position: relative;
  }

  .dial-hint {
    margin-top: 0.9rem;
    text-align: center;
    color: #64748b;
    font-size: 0.82rem;
  }
</style>

<div class="dial-wrapper">
  <!-- Horizontal time axis above knob -->
  <div class="axis-wrapper">
    <div class="axis-indicator"></div>
    <div class="axis-value-badge">{currentHour}h</div>
    <div class="axis-track" style="--axis-scroll: {axisOffset}px">
      {#each tickMarks as tick}
        {@const isActive = tick.hour === currentHour}
        {@const leftPx = tick.hour * HOUR_PX}
        <div
          class="axis-tick {tick.major ? 'major' : ''} {isActive ? 'active' : ''}"
          style="left: {leftPx}px"
        ></div>
        {#if tick.major}
          <span
            class="axis-label {isActive ? 'active' : ''}"
            style="left: {leftPx}px"
          >{tick.hour}h</span>
        {/if}
      {/each}
    </div>
  </div>

  <div
    class="dial-root"
    bind:this={dialEl}
    role="slider"
    tabindex="0"
    aria-valuemin={min}
    aria-valuemax={max}
    aria-valuenow={value}
    on:pointerdown={handlePointerDown}
    on:pointermove={handlePointerMove}
    on:pointerup={handlePointerUp}
    on:pointercancel={handlePointerUp}
    on:pointerleave={handlePointerUp}
  >
    <div class="dial-inner" style="--dial-rotate: -{dialAngle}deg">
      <div class="dial-ring">
        <div class="dial-surface" style="background-image: url({knobImg})">
        </div>
      </div>
    </div>
  </div>
  <div class="dial-hint">左右滑动调整时长</div>
</div>