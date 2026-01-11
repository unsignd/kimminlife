<script lang="ts">
  import logo from '$lib/assets/logo.svg';
  import '../app.css';

  let { children } = $props();
</script>

<svelte:head>
  <link rel="icon" href={logo} />
  <title>시발거래소</title>
</svelte:head>

<div
  class="w-screen h-screen bg-linear-to-b from-[#0b0b0d] to-[#0c0c0d] flex items-center justify-center"
>
  <div
    class="
      h-full
      max-w-250
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
