<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import heroImage from '$assets/img/cpt.jpg';

	let container: HTMLDivElement;
	let scrollY = 0;

	const MOBILE_DENSITY = 0.007; // Larger, chunkier characters for small screens
	const DESKTOP_DENSITY = 0.005; // Fine, detailed characters for large screens
	const BREAKPOINT = 768;

	// CONFIG
	const FADE_DISTANCE = 300;
	const DENSITY = 0.007;

	$: asciiOpacity = Math.min(scrollY / FADE_DISTANCE, 1);

	// --- (Insert Font Texture Helper Here if not imported) ---
	// Keeping the helper function for copy-paste completeness
	function createFontTexture() {
		const canvas = document.createElement('canvas');
		const size = 1024;
		canvas.width = size;
		canvas.height = size;
		const ctx = canvas.getContext('2d');
		if (!ctx) return null;
		ctx.fillStyle = '#000000';
		ctx.fillRect(0, 0, size, size);
		ctx.fillStyle = '#ffffff';
		const chars = "$@B%8&WM#*oahkbdpqwmZO0QLCJUYXzcvunxrjft/\|()1{}[]?-_+~<>i!lI;:,^`' ."; //'.\'`^",:;Il!i><~+_-?][}{1)(|\\/tfjrxnuvczXYUJCLQ0OZmwqpdbkhao*#MW&8%B@$';
		const charSize = 64;
		ctx.font = 'bold 64px monospace';
		ctx.textAlign = 'center';
		ctx.textBaseline = 'middle';
		for (let i = 0; i < 256; i++) {
			const char = chars[Math.floor((i / 256) * chars.length)] || ' ';
			const x = (i % 16) * charSize + charSize / 2;
			const y = Math.floor(i / 16) * charSize + charSize / 2;
			ctx.fillText(char, x, y);
		}
		const texture = new THREE.CanvasTexture(canvas);
		texture.minFilter = THREE.NearestFilter;
		texture.magFilter = THREE.NearestFilter;
		return texture;
	}
	// --------------------------------------------------------

	// Shader Code (Unchanged)
	const VertexShader = `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `;

	const FragmentShader = `
    uniform sampler2D tDiffuse;
    uniform sampler2D tFont;
    uniform float iResolutionX;
    uniform float iResolutionY;
    uniform float iDensity;
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
		// 1. Measure Container, NOT Window
		// This is the key fix. It grabs the full 'lvh' height from CSS.
		const width = container.clientWidth;
		const height = container.clientHeight;

		const scene = new THREE.Scene();
		const camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);
		const renderer = new THREE.WebGLRenderer({ alpha: true, antialias: false });

		renderer.setSize(width, height);
		renderer.setPixelRatio(window.devicePixelRatio);
		container.appendChild(renderer.domElement);

		const cameraDistance = 10;
		camera.position.z = cameraDistance;

		let plane: THREE.Mesh;
		const fontTexture = createFontTexture();
		const textureLoader = new THREE.TextureLoader();

		const uniforms = {
			tDiffuse: { value: null },
			tFont: { value: fontTexture },
			iResolutionX: { value: width },
			iResolutionY: { value: height },
			iDensity: { value: DESKTOP_DENSITY }
		};

		// --- UPDATED RESIZE LOGIC ---
		const resize = () => {
			// Measure the CONTAINER, not the window
			const newWidth = container.clientWidth;
			const newHeight = container.clientHeight;

			camera.aspect = newWidth / newHeight;
			camera.updateProjectionMatrix();
			renderer.setSize(newWidth, newHeight);

			uniforms.iResolutionX.value = newWidth;
			uniforms.iResolutionY.value = newHeight;

			if (!plane || !uniforms.tDiffuse.value) return;

			const image = uniforms.tDiffuse.value.image;
			const vFOV = THREE.MathUtils.degToRad(camera.fov);
			const visibleHeight = 2 * Math.tan(vFOV / 2) * cameraDistance;
			const visibleWidth = visibleHeight * camera.aspect;

			const screenAspect = newWidth / newHeight;
			const imageAspect = image.width / image.height;

			if (screenAspect > imageAspect) {
				plane.scale.set(visibleWidth, visibleWidth / imageAspect, 1);
			} else {
				plane.scale.set(visibleHeight * imageAspect, visibleHeight, 1);
			}
		};

		textureLoader.load(heroImage, (texture) => {
			uniforms.tDiffuse.value = texture;
			const geometry = new THREE.PlaneGeometry(1, 1);
			const material = new THREE.ShaderMaterial({
				uniforms: uniforms,
				vertexShader: VertexShader,
				fragmentShader: FragmentShader,
				transparent: true
			});
			plane = new THREE.Mesh(geometry, material);
			scene.add(plane);
			resize();
		});

		window.addEventListener('resize', resize);

		function animate() {
			requestAnimationFrame(animate);
			if (asciiOpacity > 0) {
				renderer.render(scene, camera);
			}
		}
		animate();

		return () => {
			window.removeEventListener('resize', resize);
			renderer.dispose();
			if (container) container.innerHTML = '';
		};
	});
</script>

<svelte:window bind:scrollY />

<section class="relative h-[100lvh] w-full overflow-hidden bg-black">
	<img
		src={heroImage}
		alt="Hero Background"
		class="absolute inset-0 z-0 h-full w-full object-cover"
		style="opacity: {1 - asciiOpacity}; will-change: opacity;"
	/>

	<div
		bind:this={container}
		class="pointer-events-none absolute inset-0 z-10"
		style="opacity: {asciiOpacity}; will-change: opacity;"
	></div>

	<div class="pointer-events-none relative z-20 h-[100svh] w-full">
		<div
			class="mx-auto flex h-full w-full flex-col items-start justify-end px-6 pb-24 md:pb-12 lg:max-w-[133.33vh]"
		>
			<div class="pointer-events-auto max-w-lg text-white">
				<h1 class="text-5xl leading-tight font-bold mix-blend-difference">Meow Meow Meow</h1>
				<p class="mt-4 text-xl text-gray-200 mix-blend-difference">Woof Woof</p>
			</div>
		</div>
	</div>
</section>
