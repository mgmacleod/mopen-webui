<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { getContext, onMount } from 'svelte';
	const i18n = getContext('i18n');

	import { models } from '$lib/stores';
	import { splitStream } from '$lib/utils';
	import { getModels } from '$lib/apis';
	import { createModel, getOllamaModels } from '$lib/apis/ollama';

	import Spinner from '$lib/components/common/Spinner.svelte';

	let loading = true;
	let ollamaModels = [];
	let selectedModel = null;
	let modelfile = '';
	let newModelName = '';
	let newModelFile = '';
	let creatingModel = false;

	onMount(async () => {
		await fetchModels();
	});

	const fetchModels = async () => {
		loading = true;
		try {
			const result = await getOllamaModels(localStorage.token);
			if (result && result.models) {
				ollamaModels = result.models;
			}
		} catch (error) {
			toast.error(`Failed to fetch models: ${error}`);
		}
		loading = false;
	};

	const selectModel = async (model) => {
		selectedModel = model;
		// In a real implementation, we would fetch the modelfile content
		// For now, just create a placeholder
		modelfile = `FROM ${model.model}
TEMPLATE """{{ .System }}
USER: {{ .Prompt }}
ASSISTANT: """
PARAMETER temperature 0.7
PARAMETER top_k 50
PARAMETER top_p 0.95
PARAMETER stop "</s>"
PARAMETER stop "USER:"
PARAMETER stop "ASSISTANT:"`;
	};

	const createNewModel = async () => {
		if (!newModelName) {
			toast.error($i18n.t('Model name is required'));
			return;
		}

		if (!newModelFile) {
			toast.error($i18n.t('Modelfile content is required'));
			return;
		}

		creatingModel = true;
		try {
			const res = await createModel(localStorage.token, {
				name: newModelName,
				modelfile: newModelFile
			});

			if (res && res.ok) {
				const reader = res.body
					.pipeThrough(new TextDecoderStream())
					.pipeThrough(splitStream('\n'))
					.getReader();

				let done = false;
				while (!done) {
					const result = await reader.read();
					done = result.done;
					const value = result.value;

					if (value) {
						try {
							let lines = value.split('\n');

							for (const line of lines) {
								if (line !== '') {
									let data = JSON.parse(line);

									if (data.error) {
										throw data.error;
									}
									if (data.detail) {
										throw data.detail;
									}

									if (data.status) {
										toast.success(data.status);
									}
								}
							}
						} catch (error) {
							console.log(error);
							toast.error(`${error}`);
						}
					}
				}

				// Refresh models
				await fetchModels();
				await models.set(await getModels(localStorage.token));

				// Clear form
				newModelName = '';
				newModelFile = '';

				toast.success($i18n.t('Model created successfully'));
			}
		} catch (error) {
			toast.error(`Failed to create model: ${error}`);
		}
		creatingModel = false;
	};
</script>

<div class="flex h-full w-full">
	<!-- Left Panel - Model List -->
	<div class="w-1/3 border-r border-gray-200 dark:border-gray-800 flex flex-col overflow-hidden">
		<div class="p-4 border-b border-gray-200 dark:border-gray-800">
			<h2 class="text-lg font-medium text-gray-800 dark:text-gray-200">
				{$i18n.t('Available Models')}
			</h2>
		</div>

		<div class="flex-1 overflow-y-auto p-4">
			{#if loading}
				<div class="flex items-center justify-center h-full">
					<Spinner size="lg" />
				</div>
			{:else if ollamaModels.length === 0}
				<div class="text-center text-gray-500 dark:text-gray-400 py-8">
					{$i18n.t('No models found')}
				</div>
			{:else}
				<div class="space-y-2">
					{#each ollamaModels as model}
						<button
							class="w-full p-3 text-left rounded-lg {selectedModel?.model === model.model
								? 'bg-primary/10 border border-primary/30'
								: 'bg-gray-50 dark:bg-gray-900 hover:bg-gray-100 dark:hover:bg-gray-800'}"
							on:click={() => selectModel(model)}
						>
							<div class="font-medium text-gray-800 dark:text-gray-200">
								{model.name}
							</div>
							<div class="text-sm text-gray-500 dark:text-gray-400">
								{model.model}
							</div>
							{#if model.size}
								<div class="text-xs text-gray-500 dark:text-gray-400 mt-1">
									{Math.round(model.size / (1024 * 1024))} MB
								</div>
							{/if}
						</button>
					{/each}
				</div>
			{/if}
		</div>
	</div>

	<!-- Right Panel - Model Editor -->
	<div class="w-2/3 flex flex-col overflow-hidden">
		<div class="p-4 border-b border-gray-200 dark:border-gray-800">
			<h2 class="text-lg font-medium text-gray-800 dark:text-gray-200">
				{selectedModel
					? $i18n.t('Editing: {name}', { name: selectedModel.name })
					: $i18n.t('Create New Model')}
			</h2>
		</div>

		<div class="flex-1 overflow-y-auto p-4">
			{#if selectedModel}
				<!-- Model Details -->
				<div class="mb-4 p-3 bg-gray-50 dark:bg-gray-900 rounded-lg">
					<div class="grid grid-cols-2 gap-2 text-sm">
						<div>
							<span class="text-gray-500 dark:text-gray-400">{$i18n.t('Model ID')}:</span>
							<span class="text-gray-800 dark:text-gray-200">{selectedModel.model}</span>
						</div>
						{#if selectedModel.modified_at}
							<div>
								<span class="text-gray-500 dark:text-gray-400">{$i18n.t('Modified')}:</span>
								<span class="text-gray-800 dark:text-gray-200">
									{new Date(selectedModel.modified_at).toLocaleString()}
								</span>
							</div>
						{/if}
						{#if selectedModel.size}
							<div>
								<span class="text-gray-500 dark:text-gray-400">{$i18n.t('Size')}:</span>
								<span class="text-gray-800 dark:text-gray-200">
									{Math.round(selectedModel.size / (1024 * 1024))} MB
								</span>
							</div>
						{/if}
						{#if selectedModel.format}
							<div>
								<span class="text-gray-500 dark:text-gray-400">{$i18n.t('Format')}:</span>
								<span class="text-gray-800 dark:text-gray-200">{selectedModel.format}</span>
							</div>
						{/if}
					</div>
				</div>

				<!-- Modelfile Editor -->
				<div class="mb-4">
					<label
						for="modelfile-edit"
						class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
					>
						{$i18n.t('Modelfile')}
					</label>
					<textarea
						id="modelfile-edit"
						class="w-full h-96 p-3 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 font-mono text-sm outline-none focus:ring-2 focus:ring-primary focus:border-transparent"
						bind:value={modelfile}
						placeholder={$i18n.t('Enter modelfile content')}
					></textarea>
				</div>

				<!-- Action Buttons -->
				<div class="flex justify-end space-x-3">
					<button
						class="px-4 py-2 bg-gray-100 hover:bg-gray-200 dark:bg-gray-800 dark:hover:bg-gray-700 rounded-lg text-sm"
						on:click={() => {
							selectedModel = null;
							modelfile = '';
						}}
					>
						{$i18n.t('Cancel')}
					</button>
					<button
						class="px-4 py-2 bg-primary hover:bg-primary-dark text-white rounded-lg text-sm"
						on:click={() => {
							newModelName = `${selectedModel.name}-custom`;
							newModelFile = modelfile;
							selectedModel = null;
						}}
					>
						{$i18n.t('Use as Template')}
					</button>
					<button
						class="px-4 py-2 bg-primary hover:bg-primary-dark text-white rounded-lg text-sm"
						on:click={() => {
							// Here we would save the model
							toast.info($i18n.t('Saving model... (not implemented)'));
						}}
					>
						{$i18n.t('Save Changes')}
					</button>
				</div>
			{:else}
				<!-- New Model Form -->
				<div class="space-y-4">
					<div>
						<label
							for="new-model-name"
							class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
						>
							{$i18n.t('Model Name')}
						</label>
						<input
							id="new-model-name"
							type="text"
							class="w-full p-3 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 outline-none focus:ring-2 focus:ring-primary focus:border-transparent"
							bind:value={newModelName}
							placeholder={$i18n.t('e.g. my-custom-model')}
							disabled={creatingModel}
						/>
					</div>

					<div>
						<label
							for="new-model-file"
							class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
						>
							{$i18n.t('Modelfile Content')}
						</label>
						<textarea
							id="new-model-file"
							class="w-full h-96 p-3 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 font-mono text-sm outline-none focus:ring-2 focus:ring-primary focus:border-transparent"
							bind:value={newModelFile}
							placeholder={$i18n.t('Enter modelfile content (e.g. FROM llama3:latest...)')}
							disabled={creatingModel}
						></textarea>
					</div>

					<!-- Action Buttons -->
					<div class="flex justify-end">
						<button
							class="px-4 py-2 bg-primary hover:bg-primary-dark text-white rounded-lg text-sm disabled:opacity-50 disabled:cursor-not-allowed"
							on:click={createNewModel}
							disabled={creatingModel}
						>
							{#if creatingModel}
								<div class="flex items-center">
									<Spinner size="sm" class="mr-2" />
									{$i18n.t('Creating...')}
								</div>
							{:else}
								{$i18n.t('Create Model')}
							{/if}
						</button>
					</div>
				</div>
			{/if}
		</div>
	</div>
</div>
