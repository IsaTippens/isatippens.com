<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	export let imageSrc: string;
	export let title: string = 'Nothing OS';
	export let subtitle: string = 'Pure Instinct.';

	// CONFIGURATION
	// 0 = Monochrome, 1 = Red, 2 = RGB
	export let colorMode: 0 | 1 | 2 = 2;
	export let dotColor: string = '#d71920'; // Official Nothing Red

	let container: HTMLDivElement;
	let section: HTMLElement;
	let scrollProgress = 0;

	let preview = false;

	const FADE_DISTANCE = 300;

	// --- SHADER ---
	const VertexShader = `
    varying vec2 vUv;
    void main() { 
      vUv = uv; 
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); 
    }
  `;

	const FragmentShader = `
    uniform sampler2D tDiffuse;
    uniform float iGridSize;    // Grid size in PIXELS (e.g., 10.0)
    uniform int iMode;          // 0=Mono, 1=Single, 2=RGB
    uniform vec3 iColor;
    
    varying vec2 vUv;

    void main() {
      // 1. GRID GENERATION (Screen Space)
      // We use gl_FragCoord.xy which is the pixel coordinate (0,0 to 1920,1080)
      // This ensures circles are ALWAYS circular and uniform size.
      
      vec2 screenPos = gl_FragCoord.xy;
      vec2 gridIndex = floor(screenPos / iGridSize);
      
      // Calculate the center of the dot in pixels
      vec2 centerPos = (gridIndex + 0.5) * iGridSize;
      
      // Distance from current pixel to dot center
      float dist = distance(screenPos, centerPos);
      
      // 2. SAMPLE IMAGE
      // We use vUv (Texture Coordinates) to get the color.
      // This preserves the 'object-cover' scaling handled by Three.js
      vec4 color = texture2D(tDiffuse, vUv);
      
      // Calculate brightness (Luma)
      float brightness = 0.299 * color.r + 0.587 * color.g + 0.114 * color.b;

      // 3. DOT SIZE LOGIC
      // Max radius is half the grid size (minus a tiny padding to keep them separate)
      float maxRadius = (iGridSize * 0.5) * 0.9; 
      float radius = brightness * maxRadius;

      // 4. DRAW CIRCLE
      // smoothstep creates a 1px anti-aliased edge
      float dot = 1.0 - smoothstep(radius - 0.5, radius + 0.5, dist);

      // 5. COLOR MODES
      vec3 finalColor;

      if (iMode == 0) {
          finalColor = vec3(dot); // White
      } 
      else if (iMode == 1) {
          finalColor = iColor * dot; // Custom Color
      } 
      else {
          finalColor = color.rgb * dot; // RGB
      }

      // Optimize: Discard empty pixels (performance boost)
      if (dot < 0.01) discard; 
      
      gl_FragColor = vec4(finalColor, 1.0);
    }
  `;

	onMount(() => {
		// 1. Setup ThreeJS
		const width = container.clientWidth;
		const height = container.clientHeight;

		const scene = new THREE.Scene();
		const camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);
		const renderer = new THREE.WebGLRenderer({ alpha: true, antialias: false }); // Antialias false is faster, shader handles smoothness

		renderer.setSize(width, height);
		renderer.setPixelRatio(window.devicePixelRatio);
		container.appendChild(renderer.domElement);
		camera.position.z = 10;

		let plane: THREE.Mesh;
		const textureLoader = new THREE.TextureLoader();
		const threeColor = new THREE.Color(dotColor);

		const uniforms = {
			tDiffuse: { value: null },
			iGridSize: { value: 14.0 }, // Default pixel size
			iMode: { value: colorMode },
			iColor: { value: new THREE.Vector3(threeColor.r, threeColor.g, threeColor.b) }
		};

		const resize = () => {
			const w = container.clientWidth;
			const h = container.clientHeight;
			camera.aspect = w / h;
			camera.updateProjectionMatrix();
			renderer.setSize(w, h);

			// --- RESPONSIVE DOT SIZE ---
			// Mobile (<768px): 9px dots (Dense, sharp details on small screens)
			// Desktop (>768px): 14px dots (Cleaner, better performance on 4k)
			// You can tweak these numbers to taste.
			uniforms.iGridSize.value = w < 768 ? 9.0 : 14.0;

			if (!plane || !uniforms.tDiffuse.value) return;

			// --- OBJECT COVER LOGIC ---
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
				new THREE.ShaderMaterial({
					uniforms,
					vertexShader: VertexShader,
					fragmentShader: FragmentShader,
					transparent: true
				})
			);
			scene.add(plane);
			resize();
		});

		window.addEventListener('resize', resize);

		// --- SCROLL LOOP ---
		let frameId: number;
		function animate() {
			frameId = requestAnimationFrame(animate);
			if (section) {
				const rect = section.getBoundingClientRect();
				const scrolledPast = -rect.top;
                
				if (preview) {
					scrollProgress = 1;
				} else {
					scrollProgress = scrolledPast > 0 ? Math.min(scrolledPast / FADE_DISTANCE, 1) : 0;
				}

				if (scrollProgress > 0 && rect.bottom > 0) {
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

<section
	bind:this={section}
	class="relative h-[100lvh] w-full overflow-hidden border-b border-gray-900 bg-black"
>
	<img
		src={imageSrc}
		alt={title}
		class="absolute inset-0 z-0 h-full w-full object-cover transition-opacity"
		style="opacity: {1 - scrollProgress}; will-change: opacity;"
	/>

	<div
		bind:this={container}
		class="pointer-events-none absolute inset-0 z-10"
		style="opacity: {scrollProgress}; will-change: opacity;"
	></div>

	<div class="pointer-events-none relative z-20 h-[100svh] w-full">
		<div
			class="mx-auto flex h-full w-full flex-col items-start justify-end px-6 pb-24 md:pb-12 lg:max-w-[133.33vh]"
		>
			<div class="pointer-events-auto max-w-lg text-white mix-blend-difference">
				<h1 class="font-mono text-6xl leading-none font-bold tracking-tighter uppercase">
					{title}
				</h1>
				<p class="mt-4 font-mono text-xl tracking-wide uppercase opacity-80">{subtitle}</p>
			</div>
		</div>
	</div>
</section>
