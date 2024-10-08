<!--
	@component
	A wrapped textarea component integrated with Carta. It handles the highlighting
	and propagates events to the Carta instance.	
-->

<script lang="ts">
	import { onMount } from 'svelte';
	import type { Carta } from '../carta';
	import type { TextAreaProps } from '../textarea-props';
	import { debounce } from '../utils';
	import { BROWSER } from 'esm-env';
	import { removeTemporaryNodes, speculativeHighlightUpdate } from '../speculative';

	/**
	 * The Carta instance to use.
	 */
	export let carta: Carta;
	/**
	 * The editor content.
	 */
	export let value = '';
	/**
	 * The placeholder text for the textarea.
	 */
	export let placeholder = '';
	/**
	 * The element of the wrapper div.
	 */
	export let elem: HTMLDivElement;
	/**
	 * Additional textarea properties.
	 */
	export let props: TextAreaProps = {};

	let textarea: HTMLTextAreaElement;
	let highlightElem: HTMLDivElement;
	let highlighted = value;
	let mounted = false;
	let prevValue = value;

	/**
	 * Manually resize the textarea to fit the content, so that it
	 * always perfectly overlaps the highlighting overlay.
	 */
	export const resize = () => {
		if (!mounted || !textarea) return;
		textarea.style.height = '0';
		textarea.style.height = textarea.scrollHeight + 'px';
		textarea.scrollTop = 0;

		const isFocused = document.activeElement === textarea;
		if (!isFocused) return;
		if (!carta.input) return;
		const coords = carta.input.getCursorXY();
		if (!coords) return;

		if (
			coords.top < 0 ||
			coords.top + carta.input.getRowHeight() >= elem.scrollTop + elem.clientHeight
		)
			elem.scrollTo({ top: coords?.top, behavior: 'instant' });
	};

	const focus = () => {
		// Allow text selection
		const selectedText = window.getSelection()?.toString();
		if (selectedText) return;

		textarea?.focus();
	};

	/**
	 * Highlight the text in the textarea.
	 * @param text The text to highlight.
	 */
	const highlight = async (text: string) => {
		const highlighter = await carta.highlighter();
		if (!highlighter) return;
		let html: string;

		const hl = await import('$lib/internal/highlight');
		const { isSingleTheme } = hl;

		if (isSingleTheme(highlighter.theme)) {
			// Single theme
			html = highlighter.codeToHtml(text, {
				lang: highlighter.lang,
				theme: highlighter.theme
			});
		} else {
			// Dual theme
			html = highlighter.codeToHtml(text, {
				lang: highlighter.lang,
				themes: highlighter.theme
			});
		}

		removeTemporaryNodes(highlightElem);

		if (carta.sanitizer) {
			highlighted = carta.sanitizer(html);
		} else {
			highlighted = html;
		}

		requestAnimationFrame(resize);
	};

	const debouncedHighlight = debounce(highlight, 250);

	/**
	 * Highlight the nested languages in the markdown, loading the necessary
	 * languages if needed.
	 */
	const highlightNestedLanguages = debounce(async (text: string) => {
		const highlighter = await carta.highlighter();

		const hl = await import('$lib/internal/highlight');
		const { loadNestedLanguages } = hl;

		if (!highlighter) return;
		const { updated } = await loadNestedLanguages(highlighter, text);
		if (updated) debouncedHighlight(text);
	}, 300);

	const onValueChange = (value: string) => {
		if (highlightElem) {
			speculativeHighlightUpdate(highlightElem, prevValue, value);
			requestAnimationFrame(resize);
		}

		debouncedHighlight(value);

		highlightNestedLanguages(value);
	};

	$: if (BROWSER) onValueChange(value);

	onMount(() => {
		mounted = true;
		// Resize once the DOM is updated.
		requestAnimationFrame(resize);
	});
	onMount(() => {
		carta.$setInput(textarea, elem, () => {
			value = textarea.value;
		});
	});
</script>

<div role="tooltip" id="editor-unfocus-suggestion">
	Press ESC then TAB to move the focus off the field
</div>
<div
	on:click={focus}
	on:keydown={focus}
	on:scroll
	role="textbox"
	tabindex="-1"
	class="carta-input"
	bind:this={elem}
>
	<div class="carta-input-wrapper">
		<div
			class="carta-highlight carta-font-code"
			tabindex="-1"
			aria-hidden="true"
			bind:this={highlightElem}
		>
			<!-- eslint-disable-line svelte/no-at-html-tags -->{@html highlighted}
		</div>

		<textarea
			spellcheck="false"
			class="carta-font-code"
			aria-multiline="true"
			aria-describedby="editor-unfocus-suggestion"
			tabindex="0"
			{placeholder}
			{...props}
			bind:value
			bind:this={textarea}
			on:scroll={() => (textarea.scrollTop = 0)}
			on:keydown={() => (prevValue = value)}
		/>
	</div>

	{#if mounted}
		<slot />
	{/if}
</div>

<style>
	.carta-input {
		position: relative;
	}

	.carta-input-wrapper {
		position: relative;
		font-family: monospace;
	}

	textarea {
		position: relative;
		width: 100%;
		max-width: 100%;
		min-height: 100%;

		overflow-y: hidden;
		resize: none;

		padding: 0;
		margin: 0;
		border: 0;

		color: transparent;
		background: transparent;

		outline: none;
		tab-size: 4;
	}

	.carta-highlight {
		position: absolute;
		left: 0;
		right: 0;
		top: 0;
		bottom: 0;
		margin: 0;
		user-select: none;
		height: fit-content;

		padding: inherit;
		margin: inherit;

		word-wrap: break-word;
		white-space: pre-wrap;
		word-break: break-word;
	}

	:global(.carta-highlight .shiki) {
		margin: 0;
		tab-size: 4;
		background-color: transparent !important;
	}

	:global(.carta-highlight *) {
		font-family: inherit;
		font-size: inherit;

		word-wrap: break-word;
		white-space: pre-wrap;
		word-break: break-word;
	}

	#editor-unfocus-suggestion {
		position: absolute;
		width: 1px;
		height: 1px;
		padding: 0;
		margin: -1px;
		overflow: hidden;
		clip: rect(0, 0, 0, 0);
		border: 0;
	}
</style>
