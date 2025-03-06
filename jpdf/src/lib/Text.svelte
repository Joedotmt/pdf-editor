<script>
  import { onMount, createEventDispatcher } from "svelte";
  import Toolbar from "./Toolbar.svelte";
  import { pannable } from "./utils/pannable.js";
  import { tapout } from "./utils/tapout.js";
  import { timeout } from "./utils/helper.js";
  import { Fonts } from "./utils/prepareAssets.js";
  
  export let size;
  export let text;
  export let lineHeight;
  export let x;
  export let y;
  export let fontFamily;
  export let pageScale = 1;
  
  const Families = Object.keys(Fonts);
  const dispatch = createEventDispatcher();
  
  let startX;
  let startY;
  let editable;
  let _size = size;
  let _lineHeight = lineHeight;
  let _fontFamily = fontFamily;
  let dx = 0;
  let dy = 0;
  let operation = "";
  
  // Variables for line height resizing.
  let initialLineHeight;
  let initialWidth;
  let initialFontSize;
  let textWidth = "500px"//'auto'; // Add this for controlling text width
  
  function handlePanStart(event) {
    startX = event.detail.x;
    startY = event.detail.y;
    operation = "move";
  }
  
  function handlePanMove(event) {
    dx = (event.detail.x - startX) / pageScale;
    dy = (event.detail.y - startY) / pageScale;
  }
  
  function handlePanEnd(event) {
    if (dx === 0 && dy === 0) {
      return editable.focus();
    }
    dispatch("update", {
      x: x + dx,
      y: y + dy
    });
    dx = 0;
    dy = 0;
    if (operation === "move") {
      operation = "edit";
    }
  }
  
  // Handlers for adjusting line height via the bottom-right handle.
  function handleLineHeightPanStart(event) {
    startY = event.detail.y;
    initialLineHeight = _lineHeight;
  }
  
  function handleLineHeightPanMove(event) {
  const deltaY = (event.detail.y - startY) / pageScale;
  // Adjust sensitivity as needed; dividing by 100 allows finer adjustments
  _lineHeight = Math.max(0, initialLineHeight + deltaY / 1000);
}

  
  function handleLineHeightPanEnd(event) {
    dispatch("update", { lineHeight: _lineHeight });
  }

  function handleWidthPanStart(event) {
    startX = event.detail.x;
    initialWidth = editable.clientWidth;
  }
  
  function handleWidthPanMove(event) {
    const deltaX = (event.detail.x - startX) / pageScale;
    textWidth = Math.max(1, initialWidth + deltaX) + 'px';
  }
  
  function handleWidthPanEnd(event) {
    dispatch("update", { width: parseInt(textWidth) });
  }
  
  function handleFontSizePanStart(event) {
    startX = event.detail.x;
    initialFontSize = _size;
    initialWidth = editable.clientWidth;
  }
  
  function handleFontSizePanMove(event) {
    const deltaX = (event.detail.x - startX) / pageScale;
    const scale = 1 + (deltaX / (initialFontSize * 5));
    _size = Math.max(1, initialFontSize * scale);
    // Scale the width proportionally with the font size
    textWidth = Math.max(1, initialWidth * scale) + 'px';
  }
  
  function handleFontSizePanEnd(event) {
    dispatch("update", { 
      size: _size,
      width: parseInt(textWidth)
    });
  }
  
  function onFocus() {
    operation = "edit";
  }
  
  async function onBlur() {
    if (operation !== "edit" || operation === "tool") return;
    editable.blur();
    sanitize();
    dispatch("update", {
      lines: extractLines(),
      width: editable.clientWidth
    });
    operation = "";
  }
  
  async function onPaste(e) {
    const pastedText = e.clipboardData.getData("text");
    document.execCommand("insertHTML", false, pastedText);
    await timeout();
    sanitize();
  }
  
  function onKeydown(e) {
    const childNodes = Array.from(editable.childNodes);
    if (e.keyCode === 13) {
      e.preventDefault();
      const selection = window.getSelection();
      const focusNode = selection.focusNode;
      const focusOffset = selection.focusOffset;
      if (focusNode === editable) {
        editable.insertBefore(
          document.createElement("br"),
          childNodes[focusOffset]
        );
      } else if (focusNode instanceof HTMLBRElement) {
        editable.insertBefore(document.createElement("br"), focusNode);
      } else if (focusNode.textContent.length !== focusOffset) {
        document.execCommand("insertHTML", false, "<br>");
      } else {
        let br = focusNode.nextSibling;
        if (br) {
          editable.insertBefore(document.createElement("br"), br);
        } else {
          br = editable.appendChild(document.createElement("br"));
          br = editable.appendChild(document.createElement("br"));
        }
        selection.collapse(br, 0);
      }
    }
  }
  
  function onFocusTool() {
    operation = "tool";
  }
  
  async function onBlurTool() {
    if (operation !== "tool" || operation === "edit") return;
    dispatch("update", {
      lines: extractLines(),
      lineHeight: _lineHeight,
      size: _size,
      fontFamily: _fontFamily
    });
    operation = "";
  }
  
  function sanitize() {
    let weirdNode;
    while (
      (weirdNode = Array.from(editable.childNodes).find(
        node => !["#text", "BR"].includes(node.nodeName)
      ))
    ) {
      editable.removeChild(weirdNode);
    }
  }
  
  function onChangeFont() {
    dispatch("selectFont", {
      name: _fontFamily
    });
  }
  
  function render() {
    editable.innerHTML = text;
    editable.focus();
  }
  
  function extractLines() {
    const nodes = editable.childNodes;
    const lines = [];
    let lineText = "";
    for (let index = 0; index < nodes.length; index++) {
      const node = nodes[index];
      if (node.nodeName === "BR") {
        lines.push(lineText);
        lineText = "";
      } else {
        lineText += node.textContent;
      }
    }
    lines.push(lineText);
    return lines;
  }
  
  function onDelete() {
    dispatch("delete");
  }
  
  onMount(render);
</script>

<style>
    /* Chrome, Safari, Edge, Opera */
  input::-webkit-outer-spin-button,
  input::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
  }
  input[type=number] {
    background-color: white;
    -moz-appearance: textfield;
  }
  .font-family {
    display: block;
    appearance: none;
    height: calc(var(--spacing) * 6);
    width: 100%;
    background: white;
    padding-left: 2px;
    padding-right: 8px;
    line-height: 1.25;
  }
  #enijeijiejiejjifn {
    user-select: none !important;
  }
  .handle {
    background: #6750A4;
    translate: -20px 0px;
    width: 20px;
    top: -1px;
    height: calc(100% + 2px);
    border-radius: 5px;
    border-top-right-radius: 0;
    opacity: 1;
    border-bottom-right-radius: 0;
    display: flex;
    justify-content: center;
    align-items: center;
  }
  .handle::before {
    content: "::";
    font-size: 1.2em;
    font-weight: bolder;
    color: white;
  }
</style>

<svelte:options immutable={true} />

{#if operation != ""}
  <Toolbar>
    <div role="button" tabindex="0"
      use:tapout
      on:tapout={onBlurTool}
      on:mousedown={onFocusTool}
      on:touchstart={onFocusTool}
      class="gap-2 fixed z-10 top-0 left-0 right-0 h-12 flex justify-center items-center bg-[#e7d3f5] flex justify-between bg-surface">
      <div class="mr-2 flex items-center">
        <img src="/line_height.svg" class="w-6 mr-2" alt="Line height" />
        <input
          type="number"
          min="0"
          max="100"
          step="0.0000000000001"
          class="h-6 w-20 text-center flex-shrink-0 rounded-sm text-left"
          bind:value={_lineHeight} />
      </div>
      <div class="mr-2 flex items-center">
        <img src="/text.svg" class="w-6 mr-2" alt="Font size" />
        <button
          on:click={() => _size = Math.max(0, _size - 1)}
          class="h-6 w-6 bg-white rounded-l-sm border border-gray-300 hover:bg-gray-100 flex items-center justify-center">
          <span class="text-lg font-bold">-</span>
        </button>
        <input
          style="display:none;"
          type="number"
          min="0"
          max="1200"
          step="1"
          class="h-6 w-12 text-center flex-shrink-0 border-y border-gray-300"
          bind:value={_size} />
        <button
          on:click={() => _size = Math.min(1200, _size + 1)}
          class="h-6 w-6 bg-white rounded-r-sm border border-gray-300 hover:bg-gray-100 flex items-center justify-center">
          <span class="text-lg font-bold">+</span>
        </button>
      </div>
      <div class="mr-2 flex items-center">
        <img src="/text-family.svg" class="w-4 mr-2" alt="Font family" />
        <div class="relative w-32 md:w-40">
          <select
            bind:value={_fontFamily}
            on:change={onChangeFont}
            class="font-family">
            {#each Families as family}
              <option value={family}>{family}</option>
            {/each}
          </select>
          <div class="pointer-events-none absolute inset-y-0 right-0 flex items-center px-2 text-gray-700">
            <svg class="fill-current h-4 w-4" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
              <path d="M9.293 12.95l.707.707L15.657 8l-1.414-1.414L10 10.828 5.757 6.586 4.343 8z" />
            </svg>
          </div>
        </div>
      </div>
      <button
        on:click={onDelete}
        class="hover:bg-[#f8e8ff] flex items-center justify-center w-15 rounded-full border border-outline text-on-surface hover:bg-surface-variant cursor-pointer transition-colors disabled:cursor-not-allowed disabled:border-outline-variant disabled:text-outline-variant">
        <img class="w-full h-full" src="/delete2.svg" alt="delete object" />
      </button>
    </div>
  </Toolbar>
{/if}

<div
  use:tapout
  on:tapout={onBlur}
  class="absolute left-0 top-0 select-none hover:outline-dotted hover:outline"
  style="transform: translate({x + dx}px, {y + dy}px); {operation != '' ? 'outline: dashed #6750A4 1px;' : ''}">
  <div class="relative">
    {#if operation != ''}
      <div
        use:pannable
        on:panstart={handlePanStart}
        on:panmove={handlePanMove}
        on:panend={handlePanEnd}
        class="handle absolute left-0 top-0 w-4 h-full cursor-move"
        title="Drag to move">
      </div>
      <div
      data-direction="middle-right"
      use:pannable
      on:panstart={handleLineHeightPanStart}
      on:panmove={handleLineHeightPanMove}
      on:panend={handleLineHeightPanEnd}
      class="absolute right-[-2.1vw] top-1/2 w-[2vw] h-[2vw] rounded-full
          cursor-pointer transform -translate-y-1/2 opacity-20 hover:opacity-50 active:opacity-100">
      <img src="/line_height.svg" class="w-6 mr-2" style="pointer-events: none;" alt="Line height">
    </div>
      <div
        data-direction="top-right"
        use:pannable
        on:panstart={handleWidthPanStart}
        on:panmove={handleWidthPanMove}
        on:panend={handleWidthPanEnd}
        style="translate: 0px -100%;"
        class="absolute right-[-2.1vw] top-0 w-[2vw] h-[2vw] rounded-full
            cursor-ew-resize transform opacity-20 hover:opacity-50 active:opacity-100">
        <img src="/width.svg" class="w-6 mr-2" style="pointer-events: none;" alt="Width">
      </div>

      <div
        data-direction="bottom-right"
        use:pannable
        on:panstart={handleFontSizePanStart}
        on:panmove={handleFontSizePanMove}
        on:panend={handleFontSizePanEnd}
        style="translate: 0px 100%;"
        class="absolute right-[-2.1vw] bottom-0 w-[2vw] h-[2vw] rounded-full
            cursor-ew-resize transform opacity-20 hover:opacity-50 active:opacity-100">
        <img src="/text.svg" class="w-6 mr-2" style="pointer-events: none;" alt="Font size">
      </div>
    {/if}
    <div
      bind:this={editable}
      on:focus={onFocus}
      on:keydown={onKeydown}
      on:paste|preventDefault={onPaste}
      contenteditable="true"
      spellcheck="false"
      class="outline-none whitespace-no-wrap py-1 px-2"
      id="enijeijiejiejjifn"
      style="font-size: {_size}px; font-family: '{_fontFamily}', serif; line-height: {_lineHeight}; width: {textWidth}; -webkit-user-select: text; white-space: pre-wrap;">
    </div>
  </div>
</div>
