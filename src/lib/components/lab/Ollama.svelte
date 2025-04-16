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

	// Model creation form state
	type Parameter = {
		name: string;
		value: string;
		type: 'int' | 'float' | 'string';
	};

	type Message = {
		role: 'system' | 'user' | 'assistant' | 'control';
		content: string;
	};

	let loading = true;
	let ollamaModels = [];
	let selectedModelId = '';
	let selectedModelInfo = null;
	let selectedModels = ['']; // For the ModelSelector component
	let creatingModel = false;

	// New structured model creation state
	let newModel = {
		name: '',
		from: '',
		template: '',
		system: '',
		parameters: [] as Parameter[],
		messages: [] as Message[]
	};

	// Helper function to parse parameters string into structured format
	const parseParameters = (paramsStr: string): Parameter[] => {
		if (!paramsStr) return [];
		const params: Parameter[] = [];
		const lines = paramsStr.split('\n');

		for (const line of lines) {
			const [name, value] = line.split('=').map((s) => s.trim());
			if (!name || !value) continue;

			// Try to infer type
			let type: 'int' | 'float' | 'string' = 'string';
			if (/^\d+$/.test(value)) type = 'int';
			else if (/^\d*\.\d+$/.test(value)) type = 'float';

			params.push({ name, value, type });
		}

		return params;
	};

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
				model_info: info.model_info || {},
				capabilities: info.capabilities || [],
				details: info.details || {}
			};
		} catch (error) {
			toast.error(`Failed to fetch model info: ${error}`);
			selectedModelInfo = null;
		}
	};

	const createNewModel = async () => {
		if (!newModel.name) {
			toast.error($i18n.t('Model name is required'));
			return;
		}

		if (!newModel.from) {
			toast.error($i18n.t('Base model is required'));
			return;
		}

		creatingModel = true;
		try {
			// Convert parameters to the expected format
			const parameters = newModel.parameters.reduce((acc, param) => {
				let value: string | number = param.value;
				if (param.type === 'int') value = parseInt(param.value);
				if (param.type === 'float') value = parseFloat(param.value);
				acc[param.name] = value;
				return acc;
			}, {});

			const modelData = {
				model: newModel.name,
				stream: false,
				from: newModel.from,
				template: newModel.template || undefined,
				system: newModel.system || undefined,
				parameters: Object.keys(parameters).length > 0 ? parameters : undefined,
				messages: newModel.messages.length > 0 ? newModel.messages : undefined
			};

			// Log the request for debugging
			console.log('Creating model with data:', modelData);

			const res = await createModel(localStorage.token, modelData);

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
				newModel = {
					name: '',
					from: '',
					template: '',
					system: '',
					parameters: [],
					messages: []
				};

				toast.success($i18n.t('Model created successfully'));
			}
		} catch (error) {
			toast.error(`Failed to create model: ${error}`);
		}
		creatingModel = false;
	};

	// Helper functions for parameters and messages
	const addParameter = () => {
		newModel.parameters = [...newModel.parameters, { name: '', value: '', type: 'string' }];
	};

	const removeParameter = (index: number) => {
		newModel.parameters = newModel.parameters.filter((_, i) => i !== index);
	};

	const addMessage = () => {
		newModel.messages = [...newModel.messages, { role: 'system', content: '' }];
	};

	const removeMessage = (index: number) => {
		newModel.messages = newModel.messages.filter((_, i) => i !== index);
	};

	// Function to use selected model as template
	const useSelectedAsTemplate = () => {
		if (!selectedModelInfo) return;

		newModel.name = `${selectedModelId}-custom`;
		newModel.from = selectedModelId;
		newModel.template = selectedModelInfo.template || '';
		newModel.system = selectedModelInfo.system || '';
		newModel.parameters = parseParameters(selectedModelInfo.parameters);
		// We don't copy messages as they're typically not part of the model definition
		newModel.messages = [];
	};
</script>

<div class="flex h-full w-full">
	<!-- Left Panel - Model Selection and Details -->
	<div class="w-1/2 border-r border-gray-200 dark:border-gray-800 flex flex-col overflow-hidden">
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
								rows="8"
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

					<!-- Model Info -->
					{#if Object.keys(selectedModelInfo.model_info).length > 0}
						<div>
							<label class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-2">
								{$i18n.t('Model Information')}
							</label>
							<div class="space-y-4">
								<!-- General Info -->
								<div class="bg-gray-50 dark:bg-gray-900 rounded-lg p-3">
									<div class="text-xs font-medium text-gray-500 dark:text-gray-400 mb-2">
										General
									</div>
									<div class="grid grid-cols-2 gap-2 text-sm">
										{#if selectedModelInfo.model_info['general.architecture']}
											<div class="col-span-2">
												<span class="text-gray-500 dark:text-gray-400">Architecture:</span>
												<span class="ml-2 font-mono"
													>{selectedModelInfo.model_info['general.architecture']}</span
												>
											</div>
										{/if}
										{#if selectedModelInfo.model_info['general.size_label']}
											<div>
												<span class="text-gray-500 dark:text-gray-400">Size:</span>
												<span class="ml-2 font-mono"
													>{selectedModelInfo.model_info['general.size_label']}</span
												>
											</div>
										{/if}
										{#if selectedModelInfo.model_info['general.parameter_count']}
											<div>
												<span class="text-gray-500 dark:text-gray-400">Parameters:</span>
												<span class="ml-2 font-mono"
													>{(selectedModelInfo.model_info['general.parameter_count'] / 1e9).toFixed(
														1
													)}B</span
												>
											</div>
										{/if}
										{#if selectedModelInfo.details?.quantization_level}
											<div>
												<span class="text-gray-500 dark:text-gray-400">Q Level:</span>
												<span class="ml-2 font-mono"
													>{selectedModelInfo.details.quantization_level}</span
												>
											</div>
										{/if}
									</div>
								</div>

								<!-- Model Details -->
								{#if selectedModelInfo.model_info[`${selectedModelInfo.model_info['general.architecture']}.context_length`]}
									<div class="bg-gray-50 dark:bg-gray-900 rounded-lg p-3">
										<div class="text-xs font-medium text-gray-500 dark:text-gray-400 mb-2">
											Model Details
										</div>
										<div class="grid grid-cols-2 gap-2 text-sm">
											<div>
												<span class="text-gray-500 dark:text-gray-400">Context Length:</span>
												<span class="ml-2 font-mono">
													{selectedModelInfo.model_info[
														`${selectedModelInfo.model_info['general.architecture']}.context_length`
													].toLocaleString()}
												</span>
											</div>
											{#if selectedModelInfo.model_info[`${selectedModelInfo.model_info['general.architecture']}.attention.head_count`]}
												<div>
													<span class="text-gray-500 dark:text-gray-400">Attention Heads:</span>
													<span class="ml-2 font-mono">
														{selectedModelInfo.model_info[
															`${selectedModelInfo.model_info['general.architecture']}.attention.head_count`
														]}
													</span>
												</div>
											{/if}
											{#if selectedModelInfo.model_info[`${selectedModelInfo.model_info['general.architecture']}.block_count`]}
												<div>
													<span class="text-gray-500 dark:text-gray-400">Block Count:</span>
													<span class="ml-2 font-mono">
														{selectedModelInfo.model_info[
															`${selectedModelInfo.model_info['general.architecture']}.block_count`
														]}
													</span>
												</div>
											{/if}
										</div>
									</div>
								{/if}

								<!-- Base Model -->
								{#if selectedModelInfo.model_info['general.base_model.0.name']}
									<div class="bg-gray-50 dark:bg-gray-900 rounded-lg p-3">
										<div class="text-xs font-medium text-gray-500 dark:text-gray-400 mb-2">
											Base Model
										</div>
										<div class="space-y-1 text-sm">
											<div>
												<span class="text-gray-500 dark:text-gray-400">Name:</span>
												<span class="ml-2"
													>{selectedModelInfo.model_info['general.base_model.0.name']}</span
												>
											</div>
											{#if selectedModelInfo.model_info['general.base_model.0.organization']}
												<div>
													<span class="text-gray-500 dark:text-gray-400">Organization:</span>
													<span class="ml-2"
														>{selectedModelInfo.model_info[
															'general.base_model.0.organization'
														]}</span
													>
												</div>
											{/if}
											{#if selectedModelInfo.model_info['general.base_model.0.repo_url']}
												<div>
													<span class="text-gray-500 dark:text-gray-400">Repository:</span>
													<a
														href={selectedModelInfo.model_info['general.base_model.0.repo_url']}
														target="_blank"
														rel="noopener noreferrer"
														class="ml-2 text-primary hover:text-primary-dark"
													>
														{selectedModelInfo.model_info['general.base_model.0.repo_url']}
													</a>
												</div>
											{/if}
										</div>
									</div>
								{/if}
							</div>
						</div>
					{/if}

					<!-- Capabilities -->
					{#if selectedModelInfo.capabilities && selectedModelInfo.capabilities.length > 0}
						<div>
							<label class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-2">
								{$i18n.t('Capabilities')}
							</label>
							<div class="flex flex-wrap gap-2">
								{#each selectedModelInfo.capabilities as capability}
									<span class="px-2 py-1 text-xs font-medium bg-primary/10 text-primary rounded">
										{capability}
									</span>
								{/each}
							</div>
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
				</div>
			{:else}
				<div class="text-center text-gray-500 dark:text-gray-400 py-8">
					{$i18n.t('Failed to load model details')}
				</div>
			{/if}
		</div>
	</div>

	<!-- Right Panel - Model Editor -->
	<div class="w-1/2 flex flex-col overflow-hidden">
		<div class="p-4 border-b border-gray-200 dark:border-gray-800">
			<h2 class="text-lg font-medium text-gray-800 dark:text-gray-200">
				{$i18n.t('Create New Model')}
			</h2>
		</div>

		<div class="flex-1 overflow-y-auto p-4">
			<!-- New Model Form -->
			<div class="space-y-4">
				<!-- Model Name -->
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
						bind:value={newModel.name}
						placeholder={$i18n.t('e.g. my-custom-model')}
						disabled={creatingModel}
					/>
				</div>

				<!-- Base Model -->
				<div>
					<label
						for="base-model"
						class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
					>
						{$i18n.t('Base Model (FROM)')}
					</label>
					<input
						id="base-model"
						type="text"
						class="w-full p-3 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 outline-none focus:ring-2 focus:ring-primary focus:border-transparent"
						bind:value={newModel.from}
						placeholder={$i18n.t('e.g. llama2 or mistral')}
						disabled={creatingModel}
					/>
				</div>

				<!-- Template -->
				<div>
					<label
						for="template"
						class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
					>
						{$i18n.t('Template')}
					</label>
					<textarea
						id="template"
						class="w-full h-48 p-3 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 font-mono text-sm outline-none focus:ring-2 focus:ring-primary focus:border-transparent"
						bind:value={newModel.template}
						placeholder={$i18n.t('Enter prompt template')}
						disabled={creatingModel}
					></textarea>
				</div>

				<!-- System Prompt -->
				<div>
					<label
						for="system"
						class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
					>
						{$i18n.t('System Prompt')}
					</label>
					<textarea
						id="system"
						class="w-full h-48 p-3 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 font-mono text-sm outline-none focus:ring-2 focus:ring-primary focus:border-transparent"
						bind:value={newModel.system}
						placeholder={$i18n.t('Enter system prompt')}
						disabled={creatingModel}
					></textarea>
				</div>

				<!-- Parameters -->
				<div>
					<div class="flex justify-between items-center mb-1">
						<label
							for="parameters-section"
							class="text-sm font-medium text-gray-700 dark:text-gray-300"
						>
							{$i18n.t('Parameters')}
						</label>
						<button
							id="parameters-section"
							class="text-sm text-primary hover:text-primary-dark"
							on:click={addParameter}
							disabled={creatingModel}
						>
							+ {$i18n.t('Add Parameter')}
						</button>
					</div>
					{#each newModel.parameters as param, i}
						<div class="flex gap-2 mb-2">
							<input
								type="text"
								id="param-name-{i}"
								class="flex-1 p-2 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 text-sm"
								bind:value={param.name}
								placeholder={$i18n.t('Name')}
								disabled={creatingModel}
								aria-label={$i18n.t('Parameter Name')}
							/>
							<input
								type="text"
								id="param-value-{i}"
								class="flex-1 p-2 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 text-sm"
								bind:value={param.value}
								placeholder={$i18n.t('Value')}
								disabled={creatingModel}
								aria-label={$i18n.t('Parameter Value')}
							/>
							<select
								id="param-type-{i}"
								class="w-24 p-2 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 text-sm"
								bind:value={param.type}
								disabled={creatingModel}
								aria-label={$i18n.t('Parameter Type')}
							>
								<option value="string">string</option>
								<option value="int">int</option>
								<option value="float">float</option>
							</select>
							<button
								class="p-2 text-red-500 hover:text-red-600"
								on:click={() => removeParameter(i)}
								disabled={creatingModel}
								aria-label={$i18n.t('Remove Parameter')}
							>
								×
							</button>
						</div>
					{/each}
				</div>

				<!-- Messages -->
				<div>
					<div class="flex justify-between items-center mb-1">
						<label
							for="messages-section"
							class="text-sm font-medium text-gray-700 dark:text-gray-300"
						>
							{$i18n.t('Messages')}
						</label>
						<button
							id="messages-section"
							class="text-sm text-primary hover:text-primary-dark"
							on:click={addMessage}
							disabled={creatingModel}
						>
							+ {$i18n.t('Add Message')}
						</button>
					</div>
					{#each newModel.messages as message, i}
						<div class="flex flex-col gap-2 mb-4 p-3 bg-gray-50 dark:bg-gray-900 rounded-lg">
							<div class="flex gap-2 items-center">
								<select
									id="message-role-{i}"
									class="w-32 p-2 rounded-lg bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-800 text-sm"
									bind:value={message.role}
									disabled={creatingModel}
									aria-label={$i18n.t('Message Role')}
								>
									<option value="system">system</option>
									<option value="user">user</option>
									<option value="assistant">assistant</option>
									<option value="control">control</option>
								</select>
								<button
									class="p-2 text-red-500 hover:text-red-600"
									on:click={() => removeMessage(i)}
									disabled={creatingModel}
									aria-label={$i18n.t('Remove Message')}
								>
									×
								</button>
							</div>
							<textarea
								id="message-content-{i}"
								class="w-full p-2 rounded-lg bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-800 text-sm"
								bind:value={message.content}
								rows="2"
								placeholder={$i18n.t('Message content')}
								disabled={creatingModel}
								aria-label={$i18n.t('Message Content')}
							></textarea>
						</div>
					{/each}
				</div>

				<!-- Action Buttons -->
				<div class="flex justify-end space-x-3">
					{#if selectedModelInfo?.modelfile}
						<button
							class="px-4 py-2 bg-gray-100 hover:bg-gray-200 dark:bg-gray-800 dark:hover:bg-gray-700 rounded-lg text-sm"
							on:click={useSelectedAsTemplate}
							disabled={creatingModel}
						>
							{$i18n.t('Use Selected as Template')}
						</button>
					{/if}
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
		</div>
	</div>
</div>
