<script lang="ts">
	import Code from '$lib/components/code/Code.svelte';
	import { track } from '$lib/components/header-tracker/headers.svelte';

	let { data } = $props();
</script>

<svelte:head>
	<title>Slash Plugin - Carta</title>
	<meta name="description" content="Slash Plugin for Carta. Adds support for slash commands" />
</svelte:head>

<span class="section">Plugins</span>

<h1 class="title">Slash</h1>

<p>
	This plugin adds support for <strong>Slash</strong> commands.
</p>

<h2 use:track id="installation">Installation</h2>

<Code code={data.codeBlocks.installation} />

<h2 use:track id="setup">Setup</h2>

<h3>Styles</h3>

<p>Import the default theme, or create you own:</p>

<Code code={data.codeBlocks.styles} />

<h3>Extension</h3>

<Code code={data.codeBlocks.extension} />

<h2 use:track id="options">Options</h2>

<p>
	Here are the options you can pass to <code>slash()</code>:
</p>

<Code code={data.codeBlocks.options} />
