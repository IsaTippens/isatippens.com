<script>
    import Header from '$lib/components/Header.svelte';

    import templeImg from '$assets/img/hero.jpg';

    import Hero from '$views/landing/HeroNdot.svelte';
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

<Hero imageSrc={templeImg} title="I'm Isa" subtitle="This is what I do"/>
<Hero imageSrc={templeImg} title="Temple"/>
<Hero imageSrc={templeImg} title="Temple"/>