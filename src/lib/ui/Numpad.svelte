<script lang="ts">
  import { onDestroy, onMount, createEventDispatcher } from "svelte";
  import { XIcon, MoveIcon } from "svelte-feather-icons";

  const STORAGE_KEY = "sshx:numpad-position";
  const PANEL_W = 292;
  const PANEL_H = 324;
  const MARGIN = 12;

  const dispatch = createEventDispatcher<{ press: string; close: void }>();

  let x = 0;
  let y = 0;
  let dragging = false;
  let dragOffsetX = 0;
  let dragOffsetY = 0;
  let pointerId: number | null = null;

  const rows: string[][] = [
    ["7", "8", "9"],
    ["4", "5", "6"],
    ["1", "2", "3"],
    ["0", ".", "Enter", "Backspace"],
  ];

  function clampPosition(nx: number, ny: number) {
    const maxX = Math.max(MARGIN, window.innerWidth - PANEL_W - MARGIN);
    const maxY = Math.max(MARGIN, window.innerHeight - PANEL_H - MARGIN);
    return {
      x: Math.min(Math.max(MARGIN, nx), maxX),
      y: Math.min(Math.max(MARGIN, ny), maxY),
    };
  }

  function savePosition() {
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify({ x, y }));
    } catch {
      // Ignore private-mode storage failures; position memory is a convenience.
    }
  }

  function loadPosition() {
    try {
      const stored = localStorage.getItem(STORAGE_KEY);
      if (stored) {
        const pos = JSON.parse(stored);
        if (typeof pos?.x === "number" && typeof pos?.y === "number") {
          ({ x, y } = clampPosition(pos.x, pos.y));
          return;
        }
      }
    } catch {
      // Fall through to the default thumb-friendly lower-right placement.
    }
    ({ x, y } = clampPosition(window.innerWidth - PANEL_W - 16, window.innerHeight - PANEL_H - 24));
  }

  function startDrag(event: PointerEvent) {
    if (event.button !== 0) return;
    event.preventDefault();
    event.stopPropagation();
    dragging = true;
    pointerId = event.pointerId;
    dragOffsetX = event.clientX - x;
    dragOffsetY = event.clientY - y;
  }

  function onMove(event: PointerEvent) {
    if (!dragging || (pointerId !== null && event.pointerId !== pointerId)) return;
    event.preventDefault();
    const pos = clampPosition(event.clientX - dragOffsetX, event.clientY - dragOffsetY);
    x = pos.x;
    y = pos.y;
  }

  function endDrag(event: PointerEvent) {
    if (!dragging || (pointerId !== null && event.pointerId !== pointerId)) return;
    dragging = false;
    pointerId = null;
    savePosition();
  }

  function handleResize() {
    const pos = clampPosition(x, y);
    x = pos.x;
    y = pos.y;
    savePosition();
  }

  function press(key: string, event: PointerEvent) {
    if (event.button !== 0) return;
    event.preventDefault();
    event.stopPropagation();
    dispatch("press", key);
  }

  onMount(() => {
    loadPosition();
    window.addEventListener("pointermove", onMove);
    window.addEventListener("pointerup", endDrag);
    window.addEventListener("pointercancel", endDrag);
    window.addEventListener("resize", handleResize);
  });

  onDestroy(() => {
    window.removeEventListener("pointermove", onMove);
    window.removeEventListener("pointerup", endDrag);
    window.removeEventListener("pointercancel", endDrag);
    window.removeEventListener("resize", handleResize);
  });
</script>

<div
  class="numpad"
  class:dragging
  style:left={`${x}px`}
  style:top={`${y}px`}
  on:pointerdown={(event) => event.stopPropagation()}
>
  <div class="handle" on:pointerdown={startDrag}>
    <MoveIcon size="16" />
    <span>Numpad</span>
    <button
      class="close"
      title="Hide numpad"
      on:pointerdown={(event) => {
        if (event.button !== 0) return;
        event.preventDefault();
        event.stopPropagation();
        dispatch("close");
      }}
    >
      <XIcon size="16" />
    </button>
  </div>

  <div class="keys">
    {#each rows as row}
      <div class="key-row" style:grid-template-columns={`repeat(${row.length}, minmax(0, 1fr))`}>
        {#each row as key}
          <button
            class="key"
            class:wide={key === "Enter" || key === "Backspace"}
            title={key}
            on:pointerdown={(event) => press(key, event)}
          >
            {key === "Backspace" ? "Bksp" : key}
          </button>
        {/each}
      </div>
    {/each}
  </div>
</div>

<style lang="postcss">
  .numpad {
    @apply fixed z-50 w-[292px] max-w-[calc(100vw-24px)] rounded-lg border border-zinc-700 bg-zinc-900/95 shadow-2xl backdrop-blur-md;
    @apply text-zinc-100 select-none overflow-hidden;
  }

  .numpad.dragging {
    @apply shadow-indigo-900/40;
  }

  .handle {
    @apply flex items-center gap-2 px-3 py-2 border-b border-zinc-700/70 text-sm font-medium text-zinc-300 cursor-move touch-none;
  }

  .close {
    @apply ml-auto rounded-md p-1 text-zinc-400 hover:bg-zinc-700 hover:text-white active:bg-indigo-600;
  }

  .keys {
    @apply flex flex-col gap-2 p-3;
  }

  .key-row {
    @apply grid gap-2;
  }

  .key {
    @apply h-14 rounded-lg bg-zinc-800 text-xl font-semibold text-zinc-100 shadow-sm;
    @apply hover:bg-zinc-700 active:bg-indigo-600 active:text-white;
    @apply focus:outline-none focus:ring-2 focus:ring-indigo-500/70;
    touch-action: manipulation;
  }

  .key.wide {
    @apply text-sm;
  }

  @media (hover: none), (pointer: coarse) {
    .numpad {
      @apply w-[304px];
    }
    .key {
      @apply h-16 text-2xl;
    }
    .key.wide {
      @apply text-base;
    }
  }
</style>
