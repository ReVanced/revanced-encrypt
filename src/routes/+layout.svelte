<script lang="ts">
	import favicon from '$lib/assets/favicon.ico';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" type="image/x-icon" href={favicon} />
</svelte:head>

{@render children()}
