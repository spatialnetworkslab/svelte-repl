<script>
	import { getContext, onMount } from 'svelte';
	import SplitPane from '../SplitPane.svelte';
	import CodeMirror from '../CodeMirror.svelte';
	import Message from '../Message.svelte';

	const { bundle, selected, handle_change, navigate, register_module_editor } = getContext('REPL');

	export let errorLoc;

	let editor;
	onMount(() => {
		register_module_editor(editor);
	});

	export function focus() {
		editor.focus();
	}
</script>

<style>
	.editor-wrapper {
		z-index: 5;
		background: var(--back-light);
		display: flex;
		flex-direction: column;
	}

	.editor {
		flex: 1 1 auto;
	}

	:global(.columns) .editor-wrapper {
		/* make it easier to interact with scrollbar */
		padding-right: 8px;
		height: auto;
		/* height: 100%; */
	}

	section[slot] {
		overflow: auto;
	}

</style>

<div class="editor-wrapper">
	<SplitPane type="vertical" pos={80}>
		<div slot="a">
			<div class="editor">
				<CodeMirror
					bind:this={editor}
					{errorLoc}
					on:change={handle_change}
				/>
			</div>
		</div>

		<section slot="b">
			<div class="info">
				{#if $bundle}
					{#if $bundle.error}
						<Message kind="error" details={$bundle.error} filename="{$selected.name}.{$selected.type}"/>
					{:else if $bundle.warnings.length > 0}
						{#each $bundle.warnings as warning}
							<Message kind="warning" details={warning} filename="{$selected.name}.{$selected.type}"/>
						{/each}
					{/if}
				{/if}
			</div>
		</section>
	</SplitPane>

</div>