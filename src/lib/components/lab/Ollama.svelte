<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { getContext, onMount } from 'svelte';
	const i18n = getContext('i18n');

	import { models } from '$lib/stores';
	import { splitStream } from '$lib/utils';
	import { getModels } from '$lib/apis';
	import { createModel, getOllamaModels, getOllamaModelInfo } from '$lib/apis/ollama';

	import Spinner from '$lib/components/common/Spinner.svelte';
	import ModelSelector from '$lib/components/chat/ModelSelector.svelte';

	let loading = true;
	let ollamaModels = [];
	let selectedModelId = '';
	let selectedModelInfo = null;
	let newModelName = '';
	let newModelFile = '';
	let creatingModel = false;
	let selectedModels = ['']; // For the ModelSelector component

	onMount(async () => {
		await fetchModels();
	});

	const fetchModels = async () => {
		loading = true;
		try {
			const result = await getOllamaModels(localStorage.token);
			if (result && result.models) {
				ollamaModels = result.models;
				// Update the models store with Ollama models
				models.set(
					ollamaModels.map((model) => ({
						id: model.model,
						name: model.name,
						owned_by: 'ollama',
						...model
					}))
				);
			}
		} catch (error) {
			toast.error(`Failed to fetch models: ${error}`);
		}
		loading = false;
	};

	// Watch for changes to selectedModels[0] and update selectedModelId
	$: if (selectedModels[0] !== selectedModelId) {
		selectedModelId = selectedModels[0];
		if (selectedModelId) {
			selectModel(selectedModelId);
		} else {
			selectedModelInfo = null;
		}
	}

	const selectModel = async (modelId) => {
		if (!modelId) return;

		try {
			const info = await getOllamaModelInfo(localStorage.token, modelId);
			console.log('Model info:', info); // Debug log
			selectedModelInfo = {
				system: info.system || '',
				template: info.template || '',
				parameters: info.parameters || '',
				modelfile: info.modelfile || ''
			};
		} catch (error) {
			toast.error(`Failed to fetch model info: ${error}`);
			selectedModelInfo = null;
		}
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
	<!-- Left Panel - Model Selection and Details -->
	<div class="w-1/3 border-r border-gray-200 dark:border-gray-800 flex flex-col overflow-hidden">
		<!-- Model Selector -->
		<div class="p-4 border-b border-gray-200 dark:border-gray-800">
			<ModelSelector bind:selectedModels showSetDefault={false} />
		</div>

		<!-- Model Details -->
		<div class="flex-1 overflow-y-auto p-4">
			{#if loading}
				<div class="flex items-center justify-center h-full">
					<Spinner size="lg" />
				</div>
			{:else if !selectedModelId}
				<div class="text-center text-gray-500 dark:text-gray-400 py-8">
					{$i18n.t('Select a model to view details')}
				</div>
			{:else if selectedModelInfo}
				<div class="space-y-4">
					<!-- Model Name -->
					<div>
						<label
							for="model-name"
							class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-1"
						>
							{$i18n.t('Model Name')}
						</label>
						<input
							id="model-name"
							type="text"
							readonly
							value={selectedModelId}
							class="w-full text-sm font-mono bg-gray-50 dark:bg-gray-900 p-2 rounded-lg"
						/>
					</div>

					<!-- System Prompt -->
					{#if selectedModelInfo.system}
						<div>
							<label
								for="system-prompt"
								class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-1"
							>
								{$i18n.t('System Prompt')}
							</label>
							<textarea
								id="system-prompt"
								readonly
								rows="3"
								value={selectedModelInfo.system}
								class="w-full text-sm font-mono bg-gray-50 dark:bg-gray-900 p-2 rounded-lg"
							></textarea>
						</div>
					{/if}

					<!-- Template -->
					{#if selectedModelInfo.template}
						<div>
							<label
								for="template"
								class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-1"
							>
								{$i18n.t('Template')}
							</label>
							<textarea
								id="template"
								readonly
								rows="3"
								value={selectedModelInfo.template}
								class="w-full text-sm font-mono bg-gray-50 dark:bg-gray-900 p-2 rounded-lg"
							></textarea>
						</div>
					{/if}

					<!-- Parameters -->
					{#if selectedModelInfo.parameters}
						<div>
							<label
								for="parameters"
								class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-1"
							>
								{$i18n.t('Parameters')}
							</label>
							<textarea
								id="parameters"
								readonly
								rows="3"
								value={selectedModelInfo.parameters}
								class="w-full text-sm font-mono bg-gray-50 dark:bg-gray-900 p-2 rounded-lg"
							></textarea>
						</div>
					{/if}

					<!-- License -->
					{#if selectedModelInfo.license}
						<div>
							<label
								for="license"
								class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-1"
							>
								{$i18n.t('License')}
							</label>
							<textarea
								id="license"
								readonly
								rows="3"
								value={selectedModelInfo.license}
								class="w-full text-sm font-mono bg-gray-50 dark:bg-gray-900 p-2 rounded-lg"
							></textarea>
						</div>
					{/if}

					<!-- Modelfile -->
					{#if selectedModelInfo.modelfile}
						<div>
							<label
								for="modelfile"
								class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-1"
							>
								{$i18n.t('Modelfile')}
							</label>
							<textarea
								id="modelfile"
								readonly
								rows="3"
								value={selectedModelInfo.modelfile}
								class="w-full text-sm font-mono bg-gray-50 dark:bg-gray-900 p-2 rounded-lg"
							></textarea>
						</div>
					{/if}
				</div>
			{:else}
				<div class="text-center text-gray-500 dark:text-gray-400 py-8">
					{$i18n.t('Failed to load model details')}
				</div>
			{/if}
		</div>
	</div>

	<!-- Right Panel - Model Editor -->
	<div class="w-2/3 flex flex-col overflow-hidden">
		<div class="p-4 border-b border-gray-200 dark:border-gray-800">
			<h2 class="text-lg font-medium text-gray-800 dark:text-gray-200">
				{selectedModelId
					? $i18n.t('Editing: {name}', { name: selectedModelId })
					: $i18n.t('Create New Model')}
			</h2>
		</div>

		<div class="flex-1 overflow-y-auto p-4">
			{#if selectedModelId}
				<!-- Model Details -->
				<div class="mb-4 p-3 bg-gray-50 dark:bg-gray-900 rounded-lg">
					<div class="grid grid-cols-2 gap-2 text-sm">
						<div>
							<span class="text-gray-500 dark:text-gray-400">{$i18n.t('Model ID')}:</span>
							<span class="text-gray-800 dark:text-gray-200">{selectedModelId}</span>
						</div>
					</div>
				</div>

				<!-- Modelfile Editor -->
				{#if selectedModelInfo?.modelfile}
					<div class="mb-4">
						<label
							for="modelfile-edit"
							class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
						>
							{$i18n.t('Modelfile')}
						</label>
						<textarea
							id="modelfile-edit"
							readonly
							rows="3"
							value={selectedModelInfo.modelfile}
							class="w-full text-sm font-mono bg-gray-50 dark:bg-gray-900 p-2 rounded-lg"
						></textarea>
					</div>
				{/if}

				<!-- Action Buttons -->
				<div class="flex justify-end space-x-3">
					<button
						class="px-4 py-2 bg-gray-100 hover:bg-gray-200 dark:bg-gray-800 dark:hover:bg-gray-700 rounded-lg text-sm"
						on:click={() => {
							selectedModelId = '';
							selectedModels = [''];
						}}
					>
						{$i18n.t('Cancel')}
					</button>
					<button
						class="px-4 py-2 bg-primary hover:bg-primary-dark text-white rounded-lg text-sm"
						on:click={() => {
							if (selectedModelInfo?.modelfile) {
								newModelName = `${selectedModelId}-custom`;
								newModelFile = selectedModelInfo.modelfile;
							}
							selectedModelId = '';
							selectedModels = [''];
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
