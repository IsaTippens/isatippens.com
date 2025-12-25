<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { AsciiEffect } from 'three/examples/jsm/effects/AsciiEffect.js';

	// Your working alias
	import heroImage from '$assets/img/hero.jpg';

	let container: HTMLDivElement;
	let scrollY = 0;
	let opacity = 0;

	// Configuration
	const SCROLL_THRESHOLD = 500; // How many pixels to scroll before full ASCII

	// Reactive statement to calculate opacity based on scroll
	$: opacity = Math.min(scrollY / SCROLL_THRESHOLD, 1);

	onMount(() => {
		// 1. Setup Basic Three.js Scene
		const scene = new THREE.Scene();
		const camera = new THREE.PerspectiveCamera(
			75,
			window.innerWidth / window.innerHeight,
			0.1,
			1000
		);
		const renderer = new THREE.WebGLRenderer();

		// 2. Load Texture (Your Image)
		const textureLoader = new THREE.TextureLoader();
		textureLoader.load(heroImage, (texture) => {
			// Create a flat plane with your image
			// We adjust the plane size to match the aspect ratio roughly
			const geometry = new THREE.PlaneGeometry(16, 9);
			const material = new THREE.MeshBasicMaterial({ map: texture });
			const plane = new THREE.Mesh(geometry, material);
			scene.add(plane);
		});

		camera.position.z = 7; // Move camera back to see the plane

		// 3. Initialize AsciiEffect
		// 'invert: true' is often needed for dark backgrounds
		const effect = new AsciiEffect(renderer, ' .:-+*=%@#', { invert: true });
		effect.setSize(window.innerWidth, window.innerHeight);
		effect.domElement.style.color = 'white';
		effect.domElement.style.backgroundColor = 'transparent'; // Important for overlay

		// Append the ASCII DOM element to our container
		container.appendChild(effect.domElement);

		// 4. Animation Loop
		let frameId: number;
		function animate() {
			frameId = requestAnimationFrame(animate);
			// Only render if we can actually see the ASCII layer (optimization)
			if (opacity > 0) {
				effect.render(scene, camera);
			}
		}
		animate();

		// Handle Resize
		const handleResize = () => {
			camera.aspect = window.innerWidth / window.innerHeight;
			camera.updateProjectionMatrix();
			effect.setSize(window.innerWidth, window.innerHeight);
		};
		window.addEventListener('resize', handleResize);

		return () => {
			window.removeEventListener('resize', handleResize);
			cancelAnimationFrame(frameId);
			// Cleanup Three.js resources to prevent memory leaks
			renderer.dispose();
			container.innerHTML = '';
		};
	});
</script>

<svelte:window bind:scrollY />

<section class="relative h-screen w-full overflow-hidden bg-gray-900">
	<img
		src={heroImage}
		alt="Hero Background"
		class="absolute inset-0 z-0 h-full w-full object-cover transition-opacity duration-100 ease-linear"
		style="opacity: {1 - opacity}"
	/>

	<div
		bind:this={container}
		class="pointer-events-none absolute inset-0 z-10 overflow-hidden"
		style="opacity: {opacity}"
	></div>

	<div
		class="relative z-20 mx-auto flex h-full w-full max-w-[133.33vh] flex-col items-start justify-end p-12"
	>
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
