<script>
	import { onMount, setContext, createEventDispatcher } from 'svelte';
	import { writable } from 'svelte/store';
	import SplitPane from './SplitPane.svelte';
	import CodeMirror from './CodeMirror.svelte';
	import ComponentSelector from './Input/ComponentSelector.svelte';
	import ModuleEditor from './Input/ModuleEditor.svelte';
	import Output from './Output/index.svelte';
	import Bundler from './Bundler.js';
	import { is_browser } from './env.js';

	export let workersUrl;
	export let packagesUrl = 'https://unpkg.com';
	export let svelteUrl = `${packagesUrl}/svelte`;
	export let embedded = false;
	export let orientation = 'columns';
	export let relaxed = false;
	export let fixed = false;
	export let fixedPos = 50;
	export let injectedJS = '';
	export let injectedCSS = '';
	export let preprocess;

	export function toJSON() {
		// TODO there's a bug here — Svelte hoists this function because
		// it wrongly things that $components is global. Needs to
		// factor in $ variables when determining hoistability

		svelteUrl; // workaround

		return {
			imports: $bundle.imports,
			components: $components
		};
	}

	export async function set(data) {
		components.set(data.components);
		selected.set(data.components[0]);

		rebundle();

		await module_editor_ready;
		await output_ready;

		injectedCSS = data.css || '';
		module_editor.set($selected.source, $selected.type);
		output.set($selected, $compile_options);
	}

	export function update(data) {
		const { name, type } = $selected || {};

		components.set(data.components);

		const matched_component = data.components.find(file => file.name === name && file.type === type);
		selected.set(matched_component || data.components[0]);

		injectedCSS = data.css || '';

		if (matched_component) {
			module_editor.update(matched_component.source);
			output.update(matched_component, $compile_options);
		} else {
			module_editor.set(matched_component.source, matched_component.type);
			output.set(matched_component, $compile_options);
		}
	}

	if (!workersUrl) {
		throw new Error(`You must supply workersUrl prop to <Repl>`);
	}

	const dispatch = createEventDispatcher();

	const components = writable([]);
	const selected = writable(null);
	const bundle = writable(null);

	const compile_options = writable({
		generate: 'dom',
		dev: false,
		css: false,
		hydratable: false,
		customElement: false,
		immutable: false,
		legacy: false
	});

	let module_editor;
	let output;

	let current_token;
	async function rebundle() {
		const token = current_token = {};
		const preprocessed_components = preprocess ? $components.map(preprocess) : $components;
		const result = await bundler.bundle(preprocessed_components);
		if (result && token === current_token) bundle.set(result);
	}

	// TODO this is a horrible kludge, written in a panic. fix it
	let fulfil_module_editor_ready;
	let module_editor_ready = new Promise(f => fulfil_module_editor_ready = f);

	let fulfil_output_ready;
	let output_ready = new Promise(f => fulfil_output_ready = f);


	setContext('REPL', {
		components,
		selected,
		bundle,
		compile_options,

		rebundle,

		navigate: item => {
			const match = /^(.+)\.(\w+)$/.exec(item.filename);
			if (!match) return; // ???

			const [, name, type] = match;
			const component = $components.find(c => c.name === name && c.type === type);
			handle_select(component);

			// TODO select the line/column in question
		},

		handle_change: event => {
			selected.update(component => {
				// TODO this is a bit hacky — we're relying on mutability
				// so that updating components works... might be better
				// if a) components had unique IDs, b) we tracked selected
				// *index* rather than component, and c) `selected` was
				// derived from `components` and `index`
				component.source = event.detail.value;
				return component;
			});

			components.update(c => c);

			// recompile selected component
			output.update($selected, $compile_options);

			rebundle();

			dispatch('change', {
				components: $components
			});
		},

		register_module_editor(editor) {
			module_editor = editor;
			fulfil_module_editor_ready();
		},

		register_output(handlers) {
			output = handlers;
			fulfil_output_ready();
		},

		request_focus() {
			module_editor.focus();
		}
	});

	function handle_select(component) {
		selected.set(component);
		module_editor.set(component.source, component.type);
		output.set($selected, $compile_options);
	}

	let input;
	let sourceErrorLoc;
	let runtimeErrorLoc; // TODO refactor this stuff — runtimeErrorLoc is unused
	let status = null;

	const bundler = is_browser && new Bundler({
		workersUrl,
		packagesUrl,
		svelteUrl,
		onstatus: message => {
			status = message;
		}
	});

	$: if (output && $selected) {
		output.update($selected, $compile_options);
	}
</script>

<style>
	.container {
		position: relative;
		width: 100%;
		height: 100%;
	}

	.container :global(section) {
		position: relative;
		padding: 42px 0 0 0;
		height: 100%;
		box-sizing: border-box;
	}

	.container :global(section) > :global(*):first-child {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 42px;
		box-sizing: border-box;
	}

	.container :global(section) > :global(*):last-child {
		width: 100%;
		height: 100%;
	}
</style>

<div class="container" class:orientation>
	<SplitPane
		type="{orientation === 'rows' ? 'vertical' : 'horizontal'}"
		pos="{fixed ? fixedPos : orientation === 'rows' ? 50 : 60}"
		{fixed}
	>
		<section slot=a>
			<ComponentSelector {handle_select}/>
			<ModuleEditor bind:this={input} errorLoc="{sourceErrorLoc || runtimeErrorLoc}"/>
		</section>

		<section slot=b style='height: 100%;'>
			<Output {svelteUrl} {workersUrl} {status} {embedded} {relaxed} {injectedJS} {injectedCSS}/>
		</section>
	</SplitPane>
</div>
