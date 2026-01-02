<script lang="ts">
  import NDotImage from '$lib/components/NDotImage.svelte';
  import heroImage from '$lib/assets/img/hero.jpg';

  let scrollY = 0;
  let outerContainer: HTMLDivElement;
  let progress = 0;

  // REACTIVE LOGIC
  // We reference 'scrollY' here so Svelte knows to re-run this math every pixel you scroll.
  $: if (outerContainer && scrollY >= 0) {
      
    const rect = outerContainer.getBoundingClientRect();
    const viewportHeight = window.innerHeight;
    
    // Calculate the 'scrollable distance' of the container (Total height - 1 screen height)
    const scrollableDistance = rect.height - viewportHeight;

    // How far have we scrolled past the top of the container?
    // -rect.top is the number of pixels the top of the container has moved up.
    const scrolledPastTop = -rect.top;

    // Normalize to 0.0 -> 1.0 range
    const rawProgress = scrolledPastTop / scrollableDistance;

    // Clamp values so it doesn't go below 0 or above 1
    progress = Math.max(0, Math.min(rawProgress, 1));
  }
</script>

<svelte:window bind:scrollY />

<main class="w-full bg-black">

  <section class="flex h-screen w-full items-center justify-center border-b border-gray-800">
    <h1 class="text-4xl font-bold text-white">Scroll Down to Begin</h1>
  </section>


  <div bind:this={outerContainer} class="relative h-[300vh] w-full">
    
    <div class="sticky top-0 h-screen w-full overflow-hidden">
      
      <NDotImage 
        src={heroImage} 
        {progress} 
        colorMode={2} 
        dotColor="#d71920" 
      />
      
    </div>

    <div class="absolute inset-0 w-full pointer-events-none">
      
      <div class="absolute top-[20vh] left-10 max-w-md border border-white/20 bg-black/40 p-8 text-white backdrop-blur-md">
        <h2 class="text-2xl font-bold text-red-500">Phase 1</h2>
        <p class="mt-2 text-gray-300">The signal is clear. Progress: {(progress * 100).toFixed(0)}%</p>
      </div>

      <div class="absolute top-[140vh] right-10 max-w-md border border-white/20 bg-black/40 p-8 text-right text-white backdrop-blur-md">
        <h2 class="text-2xl font-bold text-red-500">Phase 2</h2>
        <p class="mt-2 text-gray-300">Signal degrading. Artifacts appearing.</p>
      </div>

      <div class="absolute bottom-[20vh] left-10 max-w-md border border-white/20 bg-black/40 p-8 text-white backdrop-blur-md">
        <h2 class="text-2xl font-bold text-red-500">Phase 3</h2>
        <p class="mt-2 text-gray-300">Complete digitization. Connection lost.</p>
      </div>

    </div>

  </div>


  <section class="flex h-screen w-full items-center justify-center border-t border-gray-800 bg-black">
    <h1 class="text-4xl font-bold text-white">End of Transmission</h1>
  </section>

</main>