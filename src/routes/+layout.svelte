<script lang="ts">
	import "./layout.css";
	import favicon from "$lib/assets/favicon.svg";

	let { children } = $props();
</script>

<svelte:head>
	<title>PhasorFlow — Kalkulator Phasor & Impedansi Elektro</title>
	<meta
		name="description"
		content="Alat bantu hitung bilangan kompleks dan impedansi paralel untuk mahasiswa Teknik Elektro. Dibuat oleh ersync."
	/>
	<link rel="icon" href={favicon} />

	<meta property="og:title" content="PhasorFlow | EE Scratchpad" />
	<meta
		property="og:description"
		content="Solusi cepat hitung Phasor & Rangkaian AC. Step-by-step pengerjaan disertakan."
	/>
	<meta property="og:type" content="website" />
	<meta property="og:url" content="https://phasor.ersync.my.id" />

	<meta name="theme-color" content="#2563eb" />

	<link rel="apple-touch-icon" href="/pwa-192x192.png" />
	<meta name="apple-mobile-web-app-capable" content="yes" />
	<meta name="apple-mobile-web-app-status-bar-style" content="default" />
</svelte:head>

{@render children()}
