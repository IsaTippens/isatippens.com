<script>
    import Header from '$lib/components/Header.svelte';

    import Hero from '$views/landing/HeroShader.svelte';
</script>

<svelte:head>
    <style>
        /* Hide scrollbar for the whole page */
        html, body {
            scrollbar-width: none; /* Firefox */
            -ms-overflow-style: none; /* IE/Edge */
        }
        html::-webkit-scrollbar, body::-webkit-scrollbar {
            display: none; /* Chrome/Safari */
        }
    </style>
</svelte:head>

<Hero />
<Hero />

