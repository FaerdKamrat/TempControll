
<script>
	import '../app.css';
	let { children } = $props();
</script>
 
<svelte:head>
	<meta charset="utf-8" />
	<meta name="viewport" content="width=device-width, initial-scale=1" />
	<meta name="description" content="Arduino Control Panel - Motor, Sensor and LED Matrix Control" />
	<title>Arduino Control Panel</title>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Space+Mono:wght@400;700&family=Inter:wght@300;400;500;600;700&display=swap"
		rel="stylesheet"
		crossorigin="anonymous"
	/>
</svelte:head>
 
{@render children()}
