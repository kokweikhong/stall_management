<script lang="ts">
	import '../app.css';
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<main>
	{@render children()}
</main>

<style lang="postcss">
	@reference "tailwindcss";

	main {
		/* container style */
		@apply mx-auto max-w-4xl p-4;
	}
</style>
