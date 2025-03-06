<script>
  import { onMount, createEventDispatcher } from "svelte";
  import { pannable } from "./utils/pannable.js";
  import { readAsArrayBuffer } from "../lib/utils/asyncReader.js";

  export let originWidth;
  export let originHeight;
  export let width;
  export let x;
  export let y;
  export let pageScale = 1;
  export let path;
  const dispatch = createEventDispatcher();

  let startX;
  let startY;
  let svg;
  let operation = "";
  let dx = 0;
  let dy = 0;
  let dw = 0;
  let direction = "";
  const ratio = originWidth / originHeight;

  // Edit mode state and handlers
  let isEditMode = false;
  function enterEditMode() {
    isEditMode = true;
  }
  function exitEditMode(event) {
    if (!event.currentTarget.contains(event.relatedTarget)) {
      isEditMode = false;
    }
  }

  async function render() {
    svg.setAttribute("viewBox", `0 0 ${originWidth} ${originHeight}`);
  }

  function handlePanMove(event) {
    const _dx = (event.detail.x - startX) / pageScale;
    const _dy = (event.detail.y - startY) / pageScale;
    if (operation === "move") {
      dx = _dx;
      dy = _dy;
    } else if (operation === "scale") {
      if (direction === "left-top") {
        let d = Math.min(_dx, _dy * ratio);
        dx = d;
        dw = -d;
        dy = d / ratio;
      }
      if (direction === "right-bottom") {
        let d = Math.max(_dx, _dy * ratio);
        dw = d;
      }
    }
  }

  function handlePanEnd() {
    if (operation === "move") {
      dispatch("update", {
        x: x + dx,
        y: y + dy
      });
      dx = 0;
      dy = 0;
    } else if (operation === "scale") {
      dispatch("update", {
        x: x + dx,
        y: y + dy,
        width: width + dw,
        scale: (width + dw) / originWidth
      });
      dx = 0;
      dy = 0;
      dw = 0;
      direction = "";
    }
    operation = "";
  }

  function handlePanStart(event) {
    startX = event.detail.x;
    startY = event.detail.y;
    if (event.detail.target === event.currentTarget) {
      operation = "move";
    } else {
      operation = "scale";
      direction = event.detail.target.dataset.direction;
    }
  }

  function onDelete() {
    dispatch("delete");
  }

  onMount(render);
</script>

<svelte:options immutable={true} />

<div
  class="absolute left-0 top-0 select-none {isEditMode ? 'shadow-lg' : 'hover:outline-dotted hover:outline'}"
  style="width: {width + dw}px; height: {(width + dw) / ratio}px; transform: translate({x + dx}px, {y + dy}px);"
  tabindex="0"
  on:focus={enterEditMode}
  on:blur={exitEditMode}>

  {#if isEditMode}
    <div
      use:pannable
      on:panstart={handlePanStart}
      on:panmove={handlePanMove}
      on:panend={handlePanEnd}
      class="absolute w-full h-full cursor-move border border-gray-400 border-dashed"
      class:cursor-grabbing={operation === 'move'}
      class:operation>
      <div
        data-direction="left-top"
        class="absolute w-[2vw] h-[2vw] bg-blue-300 rounded-full left-0 top-0 cursor-nwse-resize transform
        -translate-x-1/2 -translate-y-1/2  " />
      <div
        data-direction="right-bottom"
        class="absolute w-[2vw] h-[2vw] bg-blue-300 rounded-full right-0 bottom-0 cursor-nwse-resize transform
        translate-x-1/2 translate-y-1/2  " />
    </div>
    <div
      on:click={onDelete}
      class="absolute left-0 top-0 right-0 w-[2vw] h-[2vw] m-auto rounded-full bg-white cursor-pointer transform -translate-y-1/2">
      <img class="w-full h-full" src="/delete.svg" alt="delete object" />
    </div>
  {/if}

  <svg bind:this={svg} width="100%" height="100%">
    <path
      stroke-width="5"
      stroke-linejoin="round"
      stroke-linecap="round"
      stroke="black"
      fill="none"
      d={path} />
  </svg>
</div>

<style>
  .operation {
    background-color: rgba(0, 0, 0, 0.1);
  }
</style>
