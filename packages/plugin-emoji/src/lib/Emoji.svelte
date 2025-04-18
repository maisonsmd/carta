<script lang="ts">
	import { run } from 'svelte/legacy';

	import type { Carta } from 'carta-md';
	import { onDestroy, onMount } from 'svelte';
	import * as nodeEmoji from 'node-emoji';
	import type { TransitionConfig } from 'svelte/transition';

	interface Props {
		carta: Carta;
		inTransition: (node: Element) => TransitionConfig;
		outTransition: (node: Element) => TransitionConfig;
		maxResults: number;
	}

	let { carta, inTransition, outTransition, maxResults }: Props = $props();

	let visible = $state(false);
	let filter = $state('');
	let colonPosition = 0;
	let hoveringIndex = $state(0);
	let emojis: { emoji: string; name: string }[] = $state([]);
	let emojisElements: HTMLButtonElement[] = $state(Array(maxResults));

	onMount(() => {
		carta.input?.textarea.addEventListener('keydown', handleKeyDown);
		carta.input?.textarea.addEventListener('input', handleKeyInput);
		carta.input?.textarea.addEventListener('click', hide);
		carta.input?.textarea.addEventListener('blur', hideAfterDelay);
	});

	onDestroy(() => {
		carta.input?.textarea.removeEventListener('keydown', handleKeyDown);
		carta.input?.textarea.removeEventListener('input', handleKeyInput);
		carta.input?.textarea.removeEventListener('click', hide);
		carta.input?.textarea.removeEventListener('blur', hideAfterDelay);
	});

	function hideAfterDelay() {
		const el = document.activeElement;
		const ignoreBlur = el && el.tagName === 'TEXTAREA' && el.id === 'md';
		if (ignoreBlur) return;

		setTimeout(hide, 250);
	}

	function hide() {
		visible = false;
	}

	function handleKeyDown(e: KeyboardEvent) {
		if (!carta.input) return;
		if (visible) {
			if (e.key === ' ' || e.key === 'Escape' || (e.key === 'Backspace' && filter === '')) {
				// Close
				visible = false;
			} else if (e.key === 'Enter') {
				if (filter.length > 0) {
					// Complete emoji
					const emoji = emojis.at(hoveringIndex);
					if (emoji) {
						e.preventDefault();
						selectEmoji(emoji);
					}
				}
				visible = false;
			} else {
				// Check for arrows
				if (e.key === 'ArrowUp') {
					if (filter === '') return (visible = false);
					e.preventDefault();
					hoveringIndex = getIndexOfEmojiElementInPrevRow();
				} else if (e.key === 'ArrowDown') {
					if (filter === '') return (visible = false);
					e.preventDefault();
					hoveringIndex = getIndexOfEmojiElementInNextRow();
				} else if (e.key === 'ArrowLeft') {
					if (filter === '') return (visible = false);
					e.preventDefault();
					hoveringIndex = (emojis.length + hoveringIndex - 1) % emojis.length;
				} else if (e.key === 'ArrowRight') {
					if (filter === '') return (visible = false);
					e.preventDefault();
					hoveringIndex = (emojis.length + hoveringIndex + 1) % emojis.length;
				}
			}
		}
	}

	function handleKeyInput(e: Event) {
		if (!carta.input) return;
		const event = e as InputEvent;

		if (event.inputType === 'insertText' && event.data === ':') {
			if (!visible) {
				// open
				visible = true;
				colonPosition = carta.input.textarea.selectionStart - 1;
				filter = '';
				return;
			} else {
				// close
				visible = false;
				return;
			}
		}
		if (!visible) return;
		if (carta.input.textarea.selectionStart < colonPosition) {
			visible = false;
			return;
		}
		if (
			event.inputType === 'insertText' ||
			event.inputType === 'insertCompositionText' ||
			event.inputType === 'deleteContentBackward'
		) {
			filter = carta.input.textarea.value.slice(
				colonPosition + 1,
				carta.input.textarea.selectionStart
			);

			const lastChar = filter.length ? filter[filter.length - 1] : '';

			if (lastChar === ' ') {
				visible = false;
				return;
			}

			emojis = filter.length ? nodeEmoji.search(filter).slice(0, maxResults) : [];
			hoveringIndex = 0;
		}
	}

	function getIndexOfEmojiElementInPrevRow() {
		if (emojisElements.at(hoveringIndex)) {
			let index = hoveringIndex;
			let el = emojisElements[index];
			const startPos = {
				top: el.offsetTop,
				left: el.offsetLeft,
				right: el.offsetLeft + el.offsetWidth
			};
			let prevIndex, prevPos;
			for (;;) {
				prevIndex = index - 1;
				if (prevIndex < 0 || !emojisElements.at(prevIndex)) return index;
				index = prevIndex;

				el = emojisElements[index];
				prevPos = {
					top: el.offsetTop,
					left: el.offsetLeft,
					right: el.offsetLeft + el.offsetWidth
				};

				if (prevPos.top === startPos.top) continue;
				if (prevPos.left > startPos.right) continue;
				return index;
			}
		}
		return hoveringIndex;
	}

	function getIndexOfEmojiElementInNextRow() {
		if (emojisElements.at(hoveringIndex)) {
			let index = hoveringIndex;
			let el = emojisElements[index];
			const startPos = {
				top: el.offsetTop,
				left: el.offsetLeft,
				right: el.offsetLeft + el.offsetWidth
			};
			let nextIndex, nextPos;
			for (;;) {
				nextIndex = index + 1;
				if (nextIndex >= emojisElements.length || !emojisElements.at(nextIndex)) return index;
				index = nextIndex;

				el = emojisElements[index];
				nextPos = {
					top: el.offsetTop,
					left: el.offsetLeft,
					right: el.offsetLeft + el.offsetWidth
				};

				if (nextPos.top === startPos.top) continue;
				if (nextPos.right < startPos.left) continue;
				return index;
			}
		}
		return hoveringIndex;
	}

	function selectEmoji(emoji: { emoji: string; name: string }) {
		if (!carta.input) return;
		// Remove slash and filter
		carta.input.removeAt(colonPosition, filter.length + 1);
		carta.input.insertAt(colonPosition, ':' + emoji.name + ':');
		const newPosition = colonPosition + 2 + emoji.name.length;
		carta.input.textarea.setSelectionRange(newPosition, newPosition);
		carta.input.update();
	}

	run(() => {
		// Scroll to make hovering emoji always visible
		const hovering = emojisElements.at(hoveringIndex);
		if (hovering) {
			const snipElem = emojisElements[hoveringIndex];
			snipElem?.scrollIntoView({
				behavior: 'smooth',
				block: 'nearest'
			});
		}
	});
</script>

{#if visible && filter.length > 0 && emojis.length > 0}
	<div class="carta-emoji" in:inTransition out:outTransition use:carta.bindToCaret>
		{#each emojis as emoji, i}
			<button
				class={i === hoveringIndex ? 'carta-active' : ''}
				title={emoji.name}
				onclick={() => selectEmoji(emoji)}
				bind:this={emojisElements[i]}
			>
				{emoji.emoji}
			</button>
		{/each}
	</div>
{/if}
