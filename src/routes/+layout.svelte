<script lang="ts">
  import favicon from '$lib/assets/favicon.svg';
  import '../app.css';

  let { children } = $props();
</script>

<svelte:head>
  <link rel="icon" href={favicon} />
</svelte:head>

<div
  class="w-screen h-screen bg-linear-to-b from-[#0b0b0d] to-[#0c0c0d] flex items-center justify-center"
>
  <div
    class="
      h-full
      max-w-[1000px]
      mx-auto
      px-4 lg:px-8
      text-white
      overflow-x-hidden
      overflow-y-auto
    "
  >
    {@render children()}
  </div>
</div>
