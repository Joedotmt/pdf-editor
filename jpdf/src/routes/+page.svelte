<script>
    export let data;
    import { onMount } from "svelte";
    import { fly } from "svelte/transition";
    import Tailwind from "$lib/Tailwind.svelte";
    import PDFPage from "$lib/PDFPage.svelte";
    import Image from "$lib/Image.svelte";
    import Text from "$lib/Text.svelte";
    import Drawing from "$lib/Drawing.svelte";
    import DrawingCanvas from "$lib/DrawingCanvas.svelte";
    import prepareAssets, { fetchFont } from "$lib/utils/prepareAssets.js";
    import {
      readAsArrayBuffer,
      readAsImage,
      readAsPDF,
      readAsDataURL
    } from "$lib/utils/asyncReader.js";
    import { ggID } from "$lib/utils/helper.js";
    import { save } from "$lib/utils/PDF.js";
  
    const genID = ggID();
    let pdfFile;
    let pdfName = "";
    let pages = [];
    let pagesScale = [];
    let allObjects = [];
    let currentFont = "Times-Roman";
    let focusId = null;
    let selectedPageIndex = -1;
    let saving = false;
    let addingDrawing = false;
  
    // For test purposes – on mount, load a test PDF
    onMount(async () => {
      try {
        const res = await fetch("/test.pdf");
        const pdfBlob = await res.blob();
        await addPDF(pdfBlob);
        selectedPageIndex = 0;
        setTimeout(() => {
          fetchFont(currentFont);
          prepareAssets();
        }, 5555);
      } catch (e) {
        console.log(e);
      }
    });
  
    async function onUploadPDF(e) {
      const files = e.target.files || (e.dataTransfer && e.dataTransfer.files);
      const file = files[0];
      if (!file || file.type !== "application/pdf") return;
      selectedPageIndex = -1;
      try {
        await addPDF(file);
        selectedPageIndex = 0;
      } catch (e) {
        console.log(e);
      }
    }
  
    async function addPDF(file) {
      try {
        const pdf = await readAsPDF(file);
        pdfName = file.name;
        pdfFile = file;
        const numPages = pdf.numPages;
        pages = Array(numPages).fill().map((_, i) => pdf.getPage(i + 1));
        allObjects = pages.map(() => []);
        pagesScale = Array(numPages).fill(1);
      } catch (e) {
        console.log("Failed to add pdf.");
        throw e;
      }
    }
  
    async function onUploadImage(e) {
      const file = e.target.files[0];
      if (file && selectedPageIndex >= 0) {
        addImage(file);
      }
      e.target.value = null;
    }
  
    async function addImage(file) {
      try {
        // Get data URL to prevent canvas from being tainted
        const url = await readAsDataURL(file);
        const img = await readAsImage(url);
        const id = genID();
        const { width, height } = img;
        const object = {
          id,
          type: "image",
          width,
          height,
          x: 0,
          y: 0,
          payload: img,
          file
        };
        allObjects = allObjects.map((objects, pIndex) =>
          pIndex === selectedPageIndex ? [...objects, object] : objects
        );
      } catch (e) {
        console.log(`Fail to add image.`, e);
      }
    }
  
    function onAddTextField() {
      if (selectedPageIndex >= 0) {
        addTextField();
      }
    }
  
    function addTextField(text = "New Text Field") {
      const id = genID();
      fetchFont(currentFont);
      const object = {
        id,
        text,
        type: "text",
        size: 16,
        width: 0, // recalculate after editing
        lineHeight: 1.4,
        fontFamily: currentFont,
        x: 0,
        y: 0
      };
      allObjects = allObjects.map((objects, pIndex) =>
        pIndex === selectedPageIndex ? [...objects, object] : objects
      );
    }
  
    function onAddDrawing() {
      if (selectedPageIndex >= 0) {
        addingDrawing = true;
      }
    }
  
    function addDrawing(originWidth, originHeight, path, scale = 1) {
      const id = genID();
      const object = {
        id,
        path,
        type: "drawing",
        x: 0,
        y: 0,
        originWidth,
        originHeight,
        width: originWidth * scale,
        scale
      };
      allObjects = allObjects.map((objects, pIndex) =>
        pIndex === selectedPageIndex ? [...objects, object] : objects
      );
    }
  
    function selectFontFamily(event) {
      const name = event.detail.name;
      fetchFont(name);
      currentFont = name;
    }
  
    function selectPage(index) {
      selectedPageIndex = index;
    }
  
    function updateObject(objectId, payload) {
      allObjects = allObjects.map((objects, pIndex) =>
        pIndex === selectedPageIndex
          ? objects.map(object =>
              object.id === objectId ? { ...object, ...payload } : object
            )
          : objects
      );
    }
  
    function deleteObject(objectId) {
      allObjects = allObjects.map((objects, pIndex) =>
        pIndex === selectedPageIndex
          ? objects.filter(object => object.id !== objectId)
          : objects
      );
    }
  
    function onMeasure(scale, i) {
      pagesScale[i] = scale;
    }
  
    // FIXME: Should wait for all objects to finish their async work
    async function savePDF() {
      if (!pdfFile || saving || !pages.length) return;
      saving = true;
      try {
        await save(pdfFile, allObjects, pdfName, pagesScale);
      } catch (e) {
        console.log(e);
      } finally {
        saving = false;
      }
    }
  </script>
  
  <svelte:window
    on:dragenter|preventDefault
    on:dragover|preventDefault
    on:drop|preventDefault={onUploadPDF} />
  
  <Tailwind />
  
  <main class="flex flex-col items-center py-16 bg-[#F7F2FA] min-h-screen">
    <div
      class="gap-2 fixed z-10 top-0 left-0 right-0 h-12 flex justify-center items-center bg-[#e7d3f5] flex justify-between bg-surface">
      <input type="file" name="pdf" id="pdf" on:change={onUploadPDF} class="hidden" />
      <input type="file" id="image" name="image" class="hidden" on:change={onUploadImage} />
      <label
        for="pdf"
        class="whitespace-no-wrap px-6 py-1 rounded-2xl bg-[#6750A4] text-white font-bold hover:bg-secondary/90 transition">
        Choose PDF
      </label>
      


        <label
          for="image"
          class="hover:bg-[#f8e8ff] flex items-center justify-center w-15 rounded-full border border-outline text-on-surface hover:bg-surface-variant cursor-pointer transition-colors
                disabled:cursor-not-allowed disabled:border-outline-variant disabled:text-outline-variant"
          class:cursor-not-allowed={selectedPageIndex < 0}
          class:bg-gray-500={selectedPageIndex < 0}>
          <img src="image.svg" alt="" />
        </label>
      
        <button
          class="hover:bg-[#f8e8ff] flex items-center justify-center w-15 rounded-full border border-outline text-on-surface hover:bg-surface-variant cursor-pointer transition-colors
                disabled:cursor-not-allowed disabled:border-outline-variant disabled:text-outline-variant"
          class:cursor-not-allowed={selectedPageIndex < 0}
          class:text-outline-variant={selectedPageIndex < 0}
          on:click={onAddTextField}>
          <img src="notes.svg" alt="An icon for adding text" />
        </button>
      
        <button
          class="hover:bg-[#f8e8ff] flex items-center justify-center w-15 rounded-full border border-outline text-on-surface hover:bg-surface-variant cursor-pointer transition-colors
                disabled:cursor-not-allowed disabled:border-outline-variant disabled:text-outline-variant"
          on:click={onAddDrawing}
          class:cursor-not-allowed={selectedPageIndex < 0}
          class:text-outline-variant={selectedPageIndex < 0}>
          <img src="gesture.svg" alt="An icon for adding drawing" />
        </button>
      

      <div class="justify-center mr-3 md:mr-4 w-full max-w-xs hidden md:flex">
        <img src="/edit.svg" class="mr-2" alt="a pen, edit pdf name" />
        <input
          placeholder="Rename your PDF here"
          type="text"
          class="flex-grow bg-transparent"
          bind:value={pdfName} />
      </div>
      
      <button
        on:click={savePDF}
        class="whitespace-no-wrap px-6 py-1 rounded-2xl bg-[#6750A4] text-white font-bold hover:bg-secondary/90 transition"
        class:cursor-not-allowed={pages.length === 0 || saving || !pdfFile}
        class:bg-blue-700={pages.length === 0 || saving || !pdfFile}>
        {saving ? 'Saving' : 'Save'}
      </button>
      <a style="display: none;" href="https://github.com/ShizukuIchi/pdf-editor">
        <img
          src="/GitHub-Mark-32px.png"
          alt="A GitHub icon leads to personal GitHub page" />
      </a>
    </div>
  
    {#if addingDrawing}
      <div
        transition:fly={{ y: -200, duration: 500 }}
        class="fixed z-10 top-0 left-0 right-0 border-b border-gray-300 bg-white"
        style="height: 50%;">
        <DrawingCanvas
          on:finish={e => {
            const { originWidth, originHeight, path } = e.detail;
            let scale = 1;
            if (originWidth > 500) {
              scale = 500 / originWidth;
            }
            addDrawing(originWidth, originHeight, path, scale);
            addingDrawing = false;
          }}
          on:cancel={() => (addingDrawing = false)} />
      </div>
    {/if}
  
    {#if pages.length}
      <div class="flex justify-center px-5 w-full md:hidden">
        <img src="/edit.svg" class="mr-2" alt="a pen, edit pdf name" />
        <input
          placeholder="Rename your PDF here"
          type="text"
          class="flex-grow bg-transparent"
          bind:value={pdfName} />
      </div>
      <div class="w-full">
        {#each pages as page, pIndex (page)}
          <div
            class="p-3 w-full flex flex-col items-center overflow-hidden"
            on:mousedown={() => selectPage(pIndex)}
            on:touchstart={() => selectPage(pIndex)}>
            <div
              class="relative outline-solid outline-[#00000055]"
              class:shadow-outline={pIndex === selectedPageIndex}>
              <PDFPage
                on:measure={e => onMeasure(e.detail.scale, pIndex)}
                {page} />
              <div
                class="absolute top-0 left-0 transform origin-top-left"
                style="transform: scale({pagesScale[pIndex]}); touch-action: none;">
                {#each allObjects[pIndex] as object (object.id)}
                  {#if object.type === 'image'}
                    <Image
                      on:update={e => updateObject(object.id, e.detail)}
                      on:delete={() => deleteObject(object.id)}
                      file={object.file}
                      payload={object.payload}
                      x={object.x}
                      y={object.y}
                      width={object.width}
                      height={object.height}
                      pageScale={pagesScale[pIndex]} />
                  {:else if object.type === 'text'}
                    <Text
                      on:update={e => updateObject(object.id, e.detail)}
                      on:delete={() => deleteObject(object.id)}
                      on:selectFont={selectFontFamily}
                      text={object.text}
                      x={object.x}
                      y={object.y}
                      size={object.size}
                      lineHeight={object.lineHeight}
                      fontFamily={object.fontFamily}
                      pageScale={pagesScale[pIndex]} />
                  {:else if object.type === 'drawing'}
                    <Drawing
                      on:update={e => updateObject(object.id, e.detail)}
                      on:delete={() => deleteObject(object.id)}
                      path={object.path}
                      x={object.x}
                      y={object.y}
                      width={object.width}
                      originWidth={object.originWidth}
                      originHeight={object.originHeight}
                      pageScale={pagesScale[pIndex]} />
                  {/if}
                {/each}
              </div>
            </div>
          </div>
        {/each}
      </div>
    {:else}
      <div class="w-full flex-grow flex justify-center items-center">
        <span class="font-bold text-3xl text-gray-500">Drag something here</span>
      </div>
    {/if}
  </main>
  