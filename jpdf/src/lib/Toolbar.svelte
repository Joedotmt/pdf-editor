<script>
  import Portal from "./Portal.svelte";
  let { children } = $props();
</script>

<Portal>
  <div class="fixed z-10 top-0 left-0 right-0 h-12">
    {@render children()}
  </div>
</Portal>
