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

  function formatDuration(totalMins: number) {
    const minutes = Math.round(totalMins);
    const hours = Math.floor(minutes / 60);
    const mins = minutes % 60;
    if (hours === 0) return `${mins} 分钟`;
    if (mins === 0) return `${hours} 小时`;
    return `${hours} 小时 ${mins} 分钟`;
  }

  function formatLabel(totalMins: number) {
    const hours = Math.round(totalMins / 60);
    return `${hours}h`;
  }

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
        // snap to hour
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
    if (pointerStarterX === null || event.pointerId !== pointerId) return;
    const now = performance.now();
    const dx = event.clientX - pointerStarterX;
    const ddx = event.clientX - lastX;

    // velocity in minutes per frame (assuming ~60fps)
    const dt = Math.max(now - lastTime, 5);
    velocity = (ddx / PX_PER_HOUR) * HOUR / (dt / 16.67);

    lastX = event.clientX;
    lastTime = now;

    const raw = startValue + (dx / PX_PER_HOUR) * HOUR;
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

  $: dialAngle = valueToAngle(value);
  $: labelText = formatLabel(value);
  $: assistText = `当前 ${formatDuration(value)}，靠近整点时自动吸附`;
</script>

<style>
  .dial-wrapper {
    position: relative;
    width: 100%;
    max-width: 320px;
    margin: 0 auto;
    touch-action: none;
  }

  .dial-label {
    position: absolute;
    left: 50%;
    top: 0;
    transform: translateX(-50%) translateY(-110%);
    background: rgba(255, 255, 255, 0.95);
    padding: 0.55rem 1rem;
    border-radius: 999px;
    border: 1px solid rgba(148, 163, 184, 0.18);
    box-shadow: 0 18px 40px rgba(15, 23, 42, 0.08);
    font-size: 0.95rem;
    font-weight: 700;
    color: #0f172a;
    white-space: nowrap;
  }

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

  .dial-tick {
    position: absolute;
    left: 50%;
    top: 5%;
    width: 2px;
    height: 12%;
    background: rgba(15, 23, 42, 0.22);
    transform-origin: center bottom;
  }

  .dial-tick.major {
    height: 16%;
    background: rgba(15, 23, 42, 0.44);
  }

  .dial-center {
    position: absolute;
    inset: 22%;
    border-radius: 50%;
    background: radial-gradient(circle at 30% 30%, #ffffff, #e2e8f0 60%, #cbd5e1 100%);
    border: 1px solid rgba(148, 163, 184, 0.24);
    box-shadow: inset 0 4px 10px rgba(15, 23, 42, 0.05);
  }

  .dial-pin {
    position: absolute;
    left: 50%;
    top: 14%;
    width: 14px;
    height: 14px;
    margin-left: -7px;
    border-radius: 50%;
    background: #fff;
    border: 3px solid #2563eb;
    box-shadow: 0 6px 18px rgba(37, 99, 235, 0.22);
  }

  .dial-hint {
    margin-top: 0.9rem;
    text-align: center;
    color: #64748b;
    font-size: 0.82rem;
  }
</style>

<div class="dial-wrapper">
  <div class="dial-label">{labelText}</div>
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
    <div class="dial-inner" style="--dial-rotate: {dialAngle}deg">
      <div class="dial-ring">
        <div class="dial-surface" style="background-image: url({knobImg})">
        </div>
      </div>
    </div>
  </div>
  <div class="dial-hint">{assistText}</div>
</div>