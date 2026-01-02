<script>
	import NDotImage from '$lib/components/NDotImage.svelte';
	import articleImg from '$assets/img/hero.jpg';
</script>

<div class="h-[100lvh] bg-black py-20">
	<article class="mx-auto max-w-2xl py-20 text-white">
		<h1 class="mb-8 text-4xl font-bold">The Future of Interfaces</h1>

		<p class="mb-8 text-lg text-gray-300">
			Interfaces are evolving beyond pixels. We are moving towards organic representations of
			data...
		</p>

		<div class="my-12 h-96 w-full overflow-hidden rounded-2xl border border-gray-800">
			<NDotImage src={articleImg} colorMode={2} />
		</div>

		<p class="mt-8 text-lg text-gray-300">
			As you scroll past the image above, notice how the data decomposes into its raw grid
			representation...
		</p>
	</article>
</div>
