<script lang="ts">
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { AsciiEffect } from 'three/examples/jsm/effects/AsciiEffect.js';
  
  // Your working alias
  import heroImage from '$assets/img/james.jpg'; 

  let container: HTMLDivElement;
  let scrollY = 0;
  const FADE_DISTANCE = 1000; 

  $: asciiOpacity = Math.min(scrollY / FADE_DISTANCE, 1);

  onMount(() => {
    // Debug: Check if image path is resolving correctly in console
    console.log("Loading image from:", heroImage);

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ alpha: true });
    
    // Move camera back
    const cameraDistance = 10;
    camera.position.z = cameraDistance;

    let plane: THREE.Mesh;
    const textureLoader = new THREE.TextureLoader();
    
    // -- RESIZE LOGIC --
    const resize = () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      effect.setSize(window.innerWidth, window.innerHeight);

      // Guard clause: Ensure plane and texture exist before math
      if (!plane || !(plane.material as THREE.MeshBasicMaterial).map) return;

      const image = (plane.material as THREE.MeshBasicMaterial).map!.image;
      if (!image) return;

      const vFOV = THREE.MathUtils.degToRad(camera.fov);
      const visibleHeight = 2 * Math.tan(vFOV / 2) * cameraDistance;
      const visibleWidth = visibleHeight * camera.aspect;

      const screenAspect = window.innerWidth / window.innerHeight;
      const imageAspect = image.width / image.height;

      if (screenAspect > imageAspect) {
        plane.scale.set(visibleWidth, visibleWidth / imageAspect, 1);
      } else {
        plane.scale.set(visibleHeight * imageAspect, visibleHeight, 1);
      }
    };

    // -- LOAD TEXTURE --
    textureLoader.load(heroImage, (texture) => {
      const geometry = new THREE.PlaneGeometry(1, 1);
      const material = new THREE.MeshBasicMaterial({ map: texture });
      plane = new THREE.Mesh(geometry, material);
      scene.add(plane);
      
      // Force a resize calculation immediately after load
      resize();
    });

    // -- ASCII SETUP --
    const effect = new AsciiEffect(renderer, ' .:-+*=%@#', { invert: true }); //new AsciiEffect(renderer, ' .:-+*=%@#', { invert: true, color: true });
    effect.setSize(window.innerWidth, window.innerHeight);
    effect.domElement.style.color = 'white';
    effect.domElement.style.backgroundColor = 'transparent';
    
    // Important: Ensure the ASCII text table fills the container
    effect.domElement.style.width = '100%';
    effect.domElement.style.height = '100%';
    
    container.appendChild(effect.domElement);
    
    window.addEventListener('resize', resize);

    // -- ANIMATION LOOP --
    let frameId: number;
    function animate() {
      frameId = requestAnimationFrame(animate);
      if (asciiOpacity > 0) {
        effect.render(scene, camera);
      }
    }
    animate();

    return () => {
      window.removeEventListener('resize', resize);
      cancelAnimationFrame(frameId);
      renderer.dispose();
      if (container) container.innerHTML = '';
    };
  });
</script>

<svelte:window bind:scrollY />

<section class="relative h-screen w-full overflow-hidden bg-gray-900">
  
  <img 
    src={heroImage} 
    alt="Hero Background" 
    class="absolute inset-0 z-0 h-full w-full object-cover"
    style="opacity: {1 - asciiOpacity}; will-change: opacity;" 
  />

  <div 
    bind:this={container} 
    class="absolute inset-0 z-10 pointer-events-none overflow-hidden"
    style="opacity: {asciiOpacity}; will-change: opacity;"
  ></div>

  <div class="relative z-20 mx-auto flex h-full w-full max-w-[133.33vh] flex-col justify-end items-start p-12">
    
    <div class="max-w-lg text-white">
      <div class="max-w-fit bg-blue-300 p-4">

                <h1 class="text-5xl font-bold">Welcome to SvelteKit</h1>

            </div>

            <div class="max-w-fit bg-blue-300 p-4">

                <p class="mt-4 text-xl">This is a hero section with a real image tag.</p>

            </div>
    </div>

  </div>

</section>