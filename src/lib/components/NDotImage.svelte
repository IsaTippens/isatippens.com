<script lang="ts">
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  // --- PROPS ---
  export let src: string;
  export let alt: string = "";
  
  // CONFIG
  export let colorMode: 0 | 1 | 2 = 0; // 0=Mono, 1=Custom, 2=RGB
  export let dotColor: string = "#d71920";
  export let gridSize: number | null = null; // null = Auto responsive
  
  // MANUAL CONTROL
  // If provided, we use this value (0 to 1). If undefined, we auto-calculate based on scroll.
  export let progress: number | undefined = undefined;
  
  // INTERNAL
  let container: HTMLDivElement;
  let internalProgress = 0;
  
  // Derived state to switch between Manual and Auto
  $: activeProgress = progress !== undefined ? Math.max(0, Math.min(progress, 1)) : internalProgress;

  const AUTO_FADE_DISTANCE = 400; // Pixels to scroll to trigger effect in Auto mode

  // --- SHADER (Unchanged, optimized for arbitrary sizes) ---
  const VertexShader = `
    varying vec2 vUv;
    void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
  `;

  const FragmentShader = `
    uniform sampler2D tDiffuse;
    uniform float iGridSize;
    uniform int iMode;
    uniform vec3 iColor;
    varying vec2 vUv;

    // A reliable pseudo-random function
    float random(vec2 st) {
        return fract(sin(dot(st.xy, vec2(12.9898,78.233))) * 43758.5453123);
    }

    void main() {
      vec2 screenPos = gl_FragCoord.xy;
      vec2 gridIndex = floor(screenPos / iGridSize);
      vec2 centerPos = (gridIndex + 0.5) * iGridSize;
      float dist = distance(screenPos, centerPos);
      
      vec4 color = texture2D(tDiffuse, vUv);
      float brightness = 0.299 * color.r + 0.587 * color.g + 0.114 * color.b;

      // --- NEW NOISE LOGIC ---
      // 1. Generate a random value (0.0 to 1.0) for this specific DOT
      float noise = random(gridIndex);
      
      // 2. Create a 'Jitter' factor (-0.1 to +0.1)
      // This varies the size of the dot by +/- 10%
      float jitter = (noise - 0.5) * 0.2; 
      
      // 3. Apply jitter to brightness BEFORE calculating radius
      // We clamp it so dots don't disappear completely or overlap
      float organicBrightness = clamp(brightness + jitter, 0.0, 1.0);

      // -----------------------

      // Radius logic using the new organic brightness
      float maxRadius = (iGridSize * 0.5) * 0.9; 
      float radius = organicBrightness * maxRadius;

      float dot = 1.0 - smoothstep(radius - 0.5, radius + 0.5, dist);

      if (dot < 0.01) discard; 

      vec3 finalColor;
      if (iMode == 0) finalColor = vec3(dot); 
      else if (iMode == 1) finalColor = iColor * dot; 
      else finalColor = color.rgb * dot;
      
      gl_FragColor = vec4(finalColor, 1.0);
    }
  `;

  onMount(() => {
    // 1. INIT
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
    const threeColor = new THREE.Color(dotColor);

    const uniforms = {
        tDiffuse: { value: null },
        iGridSize: { value: 14.0 },
        iMode: { value: colorMode },
        iColor: { value: new THREE.Vector3(threeColor.r, threeColor.g, threeColor.b) }
    };

    // 2. RESIZE
    const resize = () => {
      if (!container) return;
      const w = container.clientWidth;
      const h = container.clientHeight;
      camera.aspect = w / h; camera.updateProjectionMatrix();
      renderer.setSize(w, h);
      
      // Auto Grid Size logic (unless overridden by prop)
      if (gridSize) {
        uniforms.iGridSize.value = gridSize;
      } else {
        uniforms.iGridSize.value = w < 768 ? 9.0 : 14.0;
      }

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

    textureLoader.load(src, (texture) => {
      uniforms.tDiffuse.value = texture;
      plane = new THREE.Mesh(
        new THREE.PlaneGeometry(1, 1),
        new THREE.ShaderMaterial({ uniforms, vertexShader: VertexShader, fragmentShader: FragmentShader, transparent: true })
      );
      scene.add(plane);
      resize();
    });

    // 3. OBSERVER (For Auto Mode only)
    const observer = new ResizeObserver(() => resize());
    observer.observe(container);

    // 4. ANIMATION LOOP
    let frameId: number;
    function animate() {
      frameId = requestAnimationFrame(animate);

      // --- AUTO SCROLL LOGIC ---
      // If manual progress is NOT provided, calculate it ourselves
      if (progress === undefined && container) {
        const rect = container.getBoundingClientRect();
        // If element is moving up past the top of viewport...
        const scrolledPast = -rect.top;
        if (scrolledPast > 0) {
            internalProgress = Math.min(scrolledPast / AUTO_FADE_DISTANCE, 1);
        } else {
            internalProgress = 0;
        }
      }

      // RENDER
      // Only render if effect is visible
      if (activeProgress > 0) {
        renderer.render(scene, camera);
      }
    }
    animate();

    return () => {
      observer.disconnect();
      cancelAnimationFrame(frameId);
      renderer.dispose();
      if (container) container.innerHTML = '';
    };
  });
</script>

<div bind:this={container} class="relative h-full w-full overflow-hidden bg-black">
  <img 
    {src} 
    {alt} 
    class="absolute inset-0 z-0 h-full w-full object-cover transition-opacity"
    style="opacity: {1 - activeProgress}; will-change: opacity;" 
  />

  </div>