<script lang="ts">
  export let title: string = "";
</script>

<div class="max-w-7xl mx-auto px-8 py-12">
  <h1 class="text-5xl font-black mb-12 bg-gradient-to-r from-primary to-white bg-clip-text text-transparent">
    {title}
  </h1>
  <slot />
</div>
