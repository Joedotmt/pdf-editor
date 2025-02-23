<script>
  import { createEventDispatcher } from "svelte";
  import { pannable } from "./utils/pannable.js";
  const dispatch = createEventDispatcher();
  let canvas;
  let x = 0;
  let y = 0;
  let path = "";
  let minX = Infinity;
  let maxX = 0;
  let minY = Infinity;
  let maxY = 0;
  let paths = [];
  let drawing = false;
  function handlePanStart(event) {
    if (event.detail.target !== canvas) {
      return (drawing = false);
    }
    drawing = true;
    x = event.detail.x;
    y = event.detail.y;
    minX = Math.min(minX, x);
    maxX = Math.max(maxX, x);
    minY = Math.min(minY, y);
    maxY = Math.max(maxY, y);
    paths.push(["M", x, y]);
    path += `M${x},${y}`;
  }
  function handlePanMove(event) {
    if (!drawing) return;
    x = event.detail.x;
    y = event.detail.y;
    minX = Math.min(minX, x);
    maxX = Math.max(maxX, x);
    minY = Math.min(minY, y);
    maxY = Math.max(maxY, y);
    paths.push(["L", x, y]);
    path += `L${x},${y}`;
  }
  function handlePanEnd() {
    drawing = false;
  }
  function finish() {
    if (!paths.length) return;
    const dx = -(minX - 10);
    const dy = -(minY - 10);
    const width = maxX - minX + 20;
    const height = maxY - minY + 20;
    dispatch("finish", {
      originWidth: width,
      originHeight: height,
      path: paths.reduce((acc, cur) => {
        return acc + cur[0] + (cur[1] + dx) + "," + (cur[2] + dy);
      }, "")
    });
  }
  function cancel() {
    dispatch("cancel");
  }
</script>

<div
  bind:this={canvas}
  use:pannable
  on:panstart={handlePanStart}
  on:panmove={handlePanMove}
  on:panend={handlePanEnd}
  class="relative w-full h-full select-none">
  <div class="absolute right-0 bottom-0 mr-4 mb-4 flex">
    <button
      on:click={cancel}
      class="whitespace-no-wrap px-6 py-1 rounded-2xl text-black font-bold outline mx-3 hover:bg-secondary/90 transition">
      Cancel
    </button>
    <button
      on:click={finish}
      class="whitespace-no-wrap px-6 py-1 rounded-2xl bg-[#6750A4] text-white font-bold hover:bg-secondary/90 transition">
      Done
    </button>
  </div>
  <svg class="w-full h-full pointer-events-none">
    <path
      stroke-width="5"
      stroke-linejoin="round"
      stroke-linecap="round"
      d={path}
      stroke="black"
      fill="none" />
  </svg>
</div>
