<script>
  import { onMount, onDestroy } from 'svelte';
  let portal;

  onMount(() => {
    if (portal) {
      document.body.appendChild(portal);
    }
  });

  onDestroy(() => {
    if (portal && portal.parentNode) {
      portal.parentNode.removeChild(portal);
    }
  });
</script>

<div bind:this={portal}>
  <slot />
</div>
