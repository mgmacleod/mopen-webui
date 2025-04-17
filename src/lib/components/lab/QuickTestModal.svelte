<script lang="ts">
	import { createEventDispatcher } from 'svelte';
	import { toast } from 'svelte-sonner';
	import { splitStream } from '$lib/utils';
	import { generateTextCompletion } from '$lib/apis/ollama';
	import Spinner from '$lib/components/common/Spinner.svelte';

	export let modelId: string;
	export let show = false;

	const dispatch = createEventDispatcher();

	let prompt = '';
	let response = '';
	let loading = false;

	const close = () => {
		dispatch('close');
		show = false;
		prompt = '';
		response = '';
	};

	const testModel = async () => {
		if (!prompt.trim()) {
			toast.error('Please enter a prompt');
			return;
		}

		loading = true;
		response = '';

		try {
			const res = await generateTextCompletion(localStorage.token, modelId, prompt);

			if (res && res.ok) {
				const reader = res.body
					.pipeThrough(new TextDecoderStream())
					.pipeThrough(splitStream('\n'))
					.getReader();

				let shouldContinue = true;
				while (shouldContinue) {
					const { done, value } = await reader.read();
					if (done) {
						shouldContinue = false;
						continue;
					}

					if (value) {
						try {
							const lines = value.split('\n');
							for (const line of lines) {
								if (!line) continue;
								const data = JSON.parse(line);
								if (data.error) {
									throw new Error(data.error);
								}
								if (data.response) {
									response += data.response;
								}
							}
						} catch (error) {
							console.error('Error parsing response:', error);
							throw error;
						}
					}
				}
			} else {
				throw new Error('Failed to generate completion');
			}
		} catch (error) {
			toast.error(`Error: ${error.message}`);
		} finally {
			loading = false;
		}
	};
</script>

{#if show}
	<button
		type="button"
		class="fixed inset-0 bg-black/50 dark:bg-black/70 flex items-center justify-center p-4 z-50"
		on:click|self={close}
		on:keydown={(e) => {
			if (e.key === 'Escape') close();
		}}
		aria-label="Close modal overlay"
	>
		<button
			type="button"
			class="bg-white dark:bg-gray-900 rounded-xl shadow-lg w-full max-w-2xl max-h-[80vh] flex flex-col"
			on:click|stopPropagation={() => {}}
			aria-label="Modal content"
		>
			<!-- Header -->
			<div
				class="p-4 border-b border-gray-200 dark:border-gray-800 flex items-center justify-between"
			>
				<h2 id="modal-title" class="text-lg font-medium">Quick Test: {modelId}</h2>
				<button
					class="text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200"
					on:click={close}
					aria-label="Close modal"
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="h-5 w-5"
						viewBox="0 0 20 20"
						fill="currentColor"
					>
						<path
							fill-rule="evenodd"
							d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z"
							clip-rule="evenodd"
						/>
					</svg>
				</button>
			</div>

			<!-- Content -->
			<div class="flex-1 overflow-y-auto p-4 space-y-4">
				<!-- Prompt Input -->
				<div>
					<label
						for="prompt"
						class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
					>
						Prompt
					</label>
					<textarea
						id="prompt"
						class="w-full h-32 p-3 rounded-lg bg-gray-50 dark:bg-gray-800 border border-gray-200 dark:border-gray-700 outline-none focus:ring-2 focus:ring-primary focus:border-transparent resize-none"
						bind:value={prompt}
						placeholder="Enter your prompt here..."
						disabled={loading}
						on:keydown={(e) => {
							if (e.key === 'Enter' && !e.shiftKey) {
								e.preventDefault();
								if (!loading) {
									testModel();
								}
							}
						}}
					/>
				</div>

				<!-- Response Output -->
				<div>
					<label
						for="response"
						class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
					>
						Response
					</label>
					<div
						id="response"
						class="w-full h-64 p-3 rounded-lg bg-gray-50 dark:bg-gray-800 border border-gray-200 dark:border-gray-700 overflow-y-auto whitespace-pre-wrap font-mono text-sm text-left"
					>
						{#if loading}
							<div class="flex items-center gap-2 text-gray-500 dark:text-gray-400">
								<Spinner size="sm" />
								Generating response...
							</div>
						{:else if response}
							{response}
						{:else}
							<span class="text-gray-400 dark:text-gray-600">Response will appear here...</span>
						{/if}
					</div>
				</div>
			</div>

			<!-- Footer -->
			<div class="p-4 border-t border-gray-200 dark:border-gray-800 flex justify-end gap-3">
				<button
					class="px-4 py-2 text-sm font-medium text-gray-700 dark:text-gray-200 bg-gray-100 dark:bg-gray-800 hover:bg-gray-200 dark:hover:bg-gray-700 rounded-lg transition-colors"
					on:click={close}
					disabled={loading}
				>
					Close
				</button>
				<button
					class="px-4 py-2 text-sm font-medium text-white bg-primary hover:bg-primary-dark rounded-lg transition-colors disabled:opacity-50"
					on:click={testModel}
					disabled={loading}
				>
					{#if loading}
						<div class="flex items-center gap-2">
							<Spinner size="sm" />
							Testing...
						</div>
					{:else}
						Test Model
					{/if}
				</button>
			</div>
		</button>
	</button>
{/if}
