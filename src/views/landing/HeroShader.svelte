<script lang="ts">
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  export let imageSrc: string;
  export let title: string = "";
  export let subtitle: string = "";

  let container: HTMLDivElement;
  let section: HTMLElement; // The reference to the section itself
  let asciiOpacity = 0;

  // CONFIG
  const FADE_DISTANCE = 300; // Pixels of scrolling 'past' the top to fully fade
  const DENSITY = 0.007;

  // --- (Insert createFontTexture Helper Here) ---
  function createFontTexture() {
    const canvas = document.createElement('canvas');
    const size = 1024;
    canvas.width = size;
    canvas.height = size;
    const ctx = canvas.getContext('2d');
    if(!ctx) return null;
    ctx.fillStyle = '#000000'; ctx.fillRect(0, 0, size, size);
    ctx.fillStyle = '#ffffff'; 
    const chars = " .'`^\",:;Il!i><~+_-?][}{1)(|\\/tfjrxnuvczXYUJCLQ0OZmwqpdbkhao*#MW&8%B@$";
    const charSize = 64; 
    ctx.font = 'bold 64px monospace'; 
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    for (let i = 0; i < 256; i++) {
        const char = chars[Math.floor((i / 256) * chars.length)] || ' ';
        const x = (i % 16) * charSize + charSize / 2;
        const y = Math.floor(i / 16) * charSize + charSize / 2;
        ctx.fillText(char, x, y);
    }
    const texture = new THREE.CanvasTexture(canvas);
    texture.minFilter = THREE.NearestFilter; texture.magFilter = THREE.NearestFilter;
    return texture;
  }
  // ----------------------------------------------

  // --- SHADERS (Same as before) ---
  const VertexShader = `
    varying vec2 vUv;
    void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
  `;
  const FragmentShader = `
    uniform sampler2D tDiffuse; uniform sampler2D tFont;
    uniform float iResolutionX; uniform float iResolutionY; uniform float iDensity;
    varying vec2 vUv;
    void main() {
      vec2 uv = vUv;
      float aspectRatio = iResolutionX / iResolutionY;
      vec2 cellCount = vec2(1.0 / iDensity, 1.0 / iDensity / aspectRatio);
      vec2 gridUV = floor(uv * cellCount) / cellCount;
      vec2 cellUV = fract(uv * cellCount);
      vec4 color = texture2D(tDiffuse, gridUV);
      float gray = 0.299 * color.r + 0.587 * color.g + 0.114 * color.b;
      float charIndex = floor(gray * 255.0);
      float colIndex = mod(charIndex, 16.0);
      float rowIndex = 15.0 - floor(charIndex / 16.0);
      vec2 fontUV = (vec2(colIndex, rowIndex) + cellUV) / 16.0;
      vec4 charColor = texture2D(tFont, fontUV);
      if (charColor.r < 0.5) discard;
      gl_FragColor = vec4(color.rgb, 1.0);
    }
  `;

  onMount(() => {
    const width = container.clientWidth;
    const height = container.clientHeight;

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ alpha: true, antialias: false });
    
    renderer.setSize(width, height);
    renderer.setPixelRatio(window.devicePixelRatio);
    container.appendChild(renderer.domElement);
    
    camera.position.z = 10;

    let plane: THREE.Mesh;
    const textureLoader = new THREE.TextureLoader();
    const fontTexture = createFontTexture();

    const uniforms = {
        tDiffuse: { value: null }, tFont: { value: fontTexture },
        iResolutionX: { value: width }, iResolutionY: { value: height },
        iDensity: { value: DENSITY }
    };

    const resize = () => {
      const w = container.clientWidth;
      const h = container.clientHeight;
      camera.aspect = w / h; camera.updateProjectionMatrix();
      renderer.setSize(w, h);
      uniforms.iResolutionX.value = w; uniforms.iResolutionY.value = h;
      
      // Responsive Density
      //uniforms.iDensity.value = w < 768 ? 0.04 : 0.015;

      if (!plane || !uniforms.tDiffuse.value) return;
      const img = uniforms.tDiffuse.value.image;
      
      const vFOV = THREE.MathUtils.degToRad(camera.fov);
      const visH = 2 * Math.tan(vFOV / 2) * 10;
      const visW = visH * camera.aspect;
      const screenAsp = w / h;
      const imgAsp = img.width / img.height;

      if (screenAsp > imgAsp) plane.scale.set(visW, visW / imgAsp, 1);
      else plane.scale.set(visH * imgAsp, visH, 1);
    };

    textureLoader.load(imageSrc, (texture) => {
      uniforms.tDiffuse.value = texture;
      plane = new THREE.Mesh(
        new THREE.PlaneGeometry(1, 1),
        new THREE.ShaderMaterial({ uniforms, vertexShader: VertexShader, fragmentShader: FragmentShader, transparent: true })
      );
      scene.add(plane);
      resize();
    });

    window.addEventListener('resize', resize);

    // --- SCROLL LOGIC ---
    let frameId: number;
    function animate() {
      frameId = requestAnimationFrame(animate);

      if (section) {
        // Get position relative to viewport
        const rect = section.getBoundingClientRect();
        
        // Logic:
        // rect.top > 0: Element is below top of screen (Entering or Centered) -> 0% Effect
        // rect.top < 0: Element is going off screen (Leaving) -> Increase Effect
        
        const scrolledPast = -rect.top; // Positive number when scrolling down past element
        
        if (scrolledPast > 0) {
            // We are scrolling past it, calculate fade
            asciiOpacity = Math.min(scrolledPast / FADE_DISTANCE, 1);
        } else {
            // We haven't reached the top of this element yet
            asciiOpacity = 0;
        }

        // OPTIMIZATION:
        // 1. Only render if effect is visible (opacity > 0)
        // 2. AND if the element is actually still on screen (rect.bottom > 0)
        //    This stops it from rendering once it has scrolled completely off the top.
        if (asciiOpacity > 0 && rect.bottom > 0) {
            renderer.render(scene, camera);
        }
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

<section bind:this={section} class="relative h-[100lvh] w-full overflow-hidden bg-gray-900 border-b border-gray-800">
  
  <img 
    src={imageSrc} 
    alt={title} 
    class="absolute inset-0 z-0 h-full w-full object-cover"
    style="opacity: {1 - asciiOpacity}; will-change: opacity;" 
  />

  <div 
    bind:this={container} 
    class="absolute inset-0 z-10 pointer-events-none"
    style="opacity: {asciiOpacity}; will-change: opacity;"
  ></div>

  <div class="relative z-20 h-[100svh] w-full pointer-events-none">
    <div class="mx-auto flex h-full w-full flex-col justify-end items-start px-6 pb-24 md:pb-12 lg:max-w-[133.33vh]">
      <div class="max-w-lg text-white pointer-events-auto">
        <h1 class="text-5xl font-bold leading-tight mix-blend-difference">{title}</h1>
        <p class="mt-4 text-xl text-gray-200 mix-blend-difference">{subtitle}</p>
      </div>
    </div>
  </div>

</section>