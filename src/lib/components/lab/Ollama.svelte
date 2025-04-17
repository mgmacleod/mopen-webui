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
	import QuickTestModal from '$lib/components/lab/QuickTestModal.svelte';

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
	let showQuickTest = false;

	// New structured model creation state
	let newModel = {
		name: '',
		from: '',
		template: '',
		system: '',
		parameters: [] as Parameter[],
		messages: [] as Message[]
	};

	// Add the parameters constant at the top of the script section
	const OLLAMA_PARAMETERS = [
		{
			name: 'mirostat',
			description:
				'Enable Mirostat sampling for controlling perplexity. (0 = disabled, 1 = Mirostat, 2 = Mirostat 2.0)',
			type: 'int',
			default: '0'
		},
		{
			name: 'mirostat_eta',
			description:
				'Influences how quickly the algorithm responds to feedback. Lower value = slower adjustments.',
			type: 'float',
			default: '0.1'
		},
		{
			name: 'mirostat_tau',
			description:
				'Controls balance between coherence and diversity. Lower value = more focused text.',
			type: 'float',
			default: '5.0'
		},
		{
			name: 'num_ctx',
			description: 'Sets the size of the context window used to generate the next token.',
			type: 'int',
			default: '2048'
		},
		{
			name: 'repeat_last_n',
			description:
				'Sets how far back to look back to prevent repetition. (0 = disabled, -1 = num_ctx)',
			type: 'int',
			default: '64'
		},
		{
			name: 'repeat_penalty',
			description: 'Sets how strongly to penalize repetitions. Higher value = stronger penalty.',
			type: 'float',
			default: '1.1'
		},
		{
			name: 'temperature',
			description: 'The temperature of the model. Higher value = more creative responses.',
			type: 'float',
			default: '0.8'
		},
		{
			name: 'seed',
			description: 'Random number seed for generation. Same seed + prompt = same response.',
			type: 'int',
			default: '0'
		},
		{
			name: 'stop',
			description: 'Stop sequences to use. Model will stop generating when encountered.',
			type: 'string',
			default: ''
		},
		{
			name: 'num_predict',
			description: 'Maximum number of tokens to predict. (-1 = infinite generation)',
			type: 'int',
			default: '-1'
		},
		{
			name: 'top_k',
			description: 'Reduces probability of nonsense. Higher value = more diverse answers.',
			type: 'int',
			default: '40'
		},
		{
			name: 'top_p',
			description: 'Works with top-k. Higher value = more diverse text.',
			type: 'float',
			default: '0.9'
		},
		{
			name: 'min_p',
			description: 'Minimum probability for a token, relative to most likely token.',
			type: 'float',
			default: '0.0'
		}
	];

	// Helper function to parse parameters string into structured format
	const parseParameters = (paramsStr: string): Parameter[] => {
		if (!paramsStr) return [];
		const params: Parameter[] = [];
		const lines = paramsStr.split('\n');

		for (const line of lines) {
			// Skip empty lines
			if (!line.trim()) continue;

			// Split by whitespace and filter out empty strings
			const parts = line.split(/\s+/).filter(Boolean);
			if (parts.length < 2) continue;

			const name = parts[0];
			// Join remaining parts and trim whitespace and quotes
			const rawValue = parts.slice(1).join(' ').trim();
			// Remove surrounding quotes if present
			const value = rawValue.replace(/^["'](.*)["']$/, '$1');

			if (!name || !value) continue;

			// Try to infer type
			let type: 'int' | 'float' | 'string' = 'string';
			if (/^\d+$/.test(value)) type = 'int';
			else if (/^\d*\.\d+$/.test(value)) type = 'float';

			// For 'stop' parameter, always create a new entry
			// For other parameters, update existing or create new
			if (name === 'stop') {
				params.push({ name, value, type });
			} else {
				const existingParam = params.find((p) => p.name === name);
				if (existingParam) {
					// For non-stop parameters, update the existing value
					// This shouldn't happen in practice but keeping it for safety
					existingParam.value = value;
				} else {
					params.push({ name, value, type });
				}
			}
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
			console.log('Raw model info from API:', info); // Debug log
			selectedModelInfo = {
				system: info.system || '',
				template: info.template || '',
				parameters: info.parameters || '',
				model_info: info.model_info || {},
				capabilities: info.capabilities || [],
				details: info.details || {},
				modified_at: info.modified_at || null
			};
			console.log('Processed model info:', selectedModelInfo); // Debug log
		} catch (error) {
			toast.error(`Failed to fetch model info: ${error}`);
			selectedModelInfo = null;
		}
	};

	const createNewModel = async () => {
		if (!newModel.name) {
			toast.error($i18n.t('Model name is required'));
			return false;
		}

		if (!newModel.from) {
			toast.error($i18n.t('Base model is required'));
			return false;
		}

		creatingModel = true;
		try {
			// Convert parameters to the expected format
			const parameters = newModel.parameters.reduce((acc, param) => {
				let value: string | number = param.value;
				if (param.type === 'int') value = parseInt(param.value);
				if (param.type === 'float') value = parseFloat(param.value);

				// Special handling for 'stop' parameter to allow multiple values
				if (param.name === 'stop') {
					if (!acc[param.name]) {
						acc[param.name] = [value];
					} else if (Array.isArray(acc[param.name])) {
						acc[param.name].push(value);
					}
				} else {
					acc[param.name] = value;
				}
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
							creatingModel = false; // Reset on error
							return false;
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
				creatingModel = false; // Reset on success
				return true;
			}
		} catch (error) {
			toast.error(`Failed to create model: ${error}`);
			creatingModel = false; // Reset on error
		}
		creatingModel = false; // Reset in all cases
		return false;
	};

	// Replace the addParameter function
	const addParameter = () => {
		if (!newModel.parameters) {
			newModel.parameters = [];
		}
		newModel.parameters = [
			...newModel.parameters,
			{
				name: '',
				value: '',
				type: 'string' // Default to string type
			}
		];
	};

	// Add a watch on param.name to update type when parameter is selected
	$: {
		newModel.parameters.forEach((param, index) => {
			if (param.name) {
				const paramDef = OLLAMA_PARAMETERS.find((p) => p.name === param.name);
				if (paramDef && paramDef.type !== param.type) {
					// Update the type when parameter is selected
					newModel.parameters[index] = {
						...param,
						type: paramDef.type
					};
				}
			}
		});
	}

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

	// Function to load selected model details into the form
	const loadSelectedModelDetails = () => {
		if (!selectedModelInfo) return;

		console.log('Loading model details. Selected model info:', selectedModelInfo);

		newModel.name = selectedModelId;
		newModel.from = selectedModelInfo.details?.parent_model || '';
		newModel.template = selectedModelInfo.template || '';
		newModel.system = selectedModelInfo.system || '';

		// Convert and validate parameters
		console.log('Raw parameters string:', selectedModelInfo.parameters);
		const parsedParams = parseParameters(selectedModelInfo.parameters);
		console.log('Parsed parameters:', parsedParams);

		newModel.parameters = parsedParams
			.filter((param) => {
				const isValid = OLLAMA_PARAMETERS.some((p) => p.name === param.name);
				if (!isValid) {
					console.log('Filtering out unknown parameter:', param);
				}
				return isValid;
			})
			.map((param) => {
				const paramDef = OLLAMA_PARAMETERS.find((p) => p.name === param.name);
				console.log('Processing parameter:', param, 'Definition:', paramDef);

				// Convert value to the correct type
				let value = param.value;
				if (paramDef?.type === 'int') {
					value = parseInt(param.value).toString();
				} else if (paramDef?.type === 'float') {
					value = parseFloat(param.value).toString();
				}
				return {
					name: param.name,
					value,
					type: paramDef?.type || 'string'
				};
			});

		console.log('Final converted parameters:', newModel.parameters);

		// Reset messages since they're not part of the model definition
		newModel.messages = [];
	};

	// Update the createAndLoadModel function to handle timing properly
	const createAndLoadModel = async () => {
		const modelName = newModel.name; // Store the name before creation
		const success = await createNewModel();

		if (success && modelName) {
			// Wait a brief moment for the model list to update
			setTimeout(() => {
				selectedModels = [modelName];
			}, 100);
		}

		creatingModel = false; // Reset the creating state
	};
</script>

<div class="flex h-full w-full">
	<!-- Left Panel - Model Selection and Details -->
	<div class="w-1/2 border-r border-gray-200 dark:border-gray-800 flex flex-col overflow-hidden">
		<!-- Model Selector -->
		<div class="p-4 border-b border-gray-200 dark:border-gray-800 flex items-center gap-2">
			<div class="flex-1">
				<ModelSelector bind:selectedModels showSetDefault={false} />
			</div>
			{#if selectedModelId}
				<button
					class="flex items-center gap-1.5 px-3 py-1.5 text-sm font-medium text-gray-700 dark:text-gray-200 bg-gray-100 dark:bg-gray-800 hover:bg-gray-200 dark:hover:bg-gray-700 rounded-lg transition-colors"
					on:click={() => {
						showQuickTest = true;
					}}
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						viewBox="0 0 16 16"
						fill="currentColor"
						class="size-4"
					>
						<path
							d="M8.5 1.75a.75.75 0 0 0-1 0L4 5.251V3.5a.75.75 0 0 0-1.5 0v3.5a.75.75 0 0 0 .75.75H6.5a.75.75 0 0 0 0-1.5H4.749L8 2.749l3.251 3.502H9.5a.75.75 0 0 0 0 1.5h3.25a.75.75 0 0 0 .75-.75V3.5a.75.75 0 0 0-1.5 0v1.751L8.5 1.75Z"
						/>
						<path
							d="M12 8.5a.75.75 0 0 1 .75.75v1.751l3.5 3.502a.75.75 0 0 1-1 1.124L12 12.126V13.5a.75.75 0 0 1-1.5 0v-3.25a.75.75 0 0 1 .75-.75H12Z"
						/>
						<path
							d="M4 8.5a.75.75 0 0 0-.75.75v3.25a.75.75 0 0 0 1.5 0v-1.374l3.25 3.501a.75.75 0 0 0 1-1.124l-3.5-3.502V9.25A.75.75 0 0 0 4.75 8.5H4Z"
						/>
					</svg>
					Quick Test
				</button>
			{/if}
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
							<h3 class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-2">
								{$i18n.t('Model Information')}
							</h3>
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
										{#if selectedModelInfo.model_info['general.basename']}
											<div>
												<span class="text-gray-500 dark:text-gray-400">Base Name:</span>
												<span class="ml-2 font-mono"
													>{selectedModelInfo.model_info['general.basename']}</span
												>
											</div>
										{/if}
										{#if selectedModelInfo.model_info['general.license']}
											<div>
												<span class="text-gray-500 dark:text-gray-400">License:</span>
												<span class="ml-2 font-mono"
													>{selectedModelInfo.model_info['general.license']}</span
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
								{#if selectedModelInfo.model_info['general.base_model.0.name'] || selectedModelInfo.details?.parent_model}
									<div class="bg-gray-50 dark:bg-gray-900 rounded-lg p-3">
										<div class="text-xs font-medium text-gray-500 dark:text-gray-400 mb-2">
											Base Model
										</div>
										<div class="space-y-1 text-sm">
											{#if selectedModelInfo.model_info['general.base_model.0.name']}
												<div>
													<span class="text-gray-500 dark:text-gray-400">Name:</span>
													<span class="ml-2"
														>{selectedModelInfo.model_info['general.base_model.0.name']}</span
													>
												</div>
											{/if}
											{#if selectedModelInfo.details?.parent_model}
												<div>
													<span class="text-gray-500 dark:text-gray-400">Parent Model:</span>
													<span class="ml-2">{selectedModelInfo.details.parent_model}</span>
												</div>
											{/if}
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
							<h3 class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-2">
								{$i18n.t('Capabilities')}
							</h3>
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

					<!-- Modified At -->
					{#if selectedModelInfo.modified_at}
						<div>
							<label
								for="modified-at"
								class="block text-sm font-medium text-gray-500 dark:text-gray-400 mb-1"
							>
								{$i18n.t('Last Modified')}
							</label>
							<div class="text-sm font-mono bg-gray-50 dark:bg-gray-900 p-2 rounded-lg">
								{new Date(selectedModelInfo.modified_at).toLocaleString()}
							</div>
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

				<!-- Parameters -->
				<div>
					<div class="flex justify-between items-center mb-1">
						<h3 class="text-sm font-medium text-gray-700 dark:text-gray-300">
							{$i18n.t('Parameters')}
						</h3>
						<button
							class="text-sm text-primary hover:text-primary-dark"
							on:click={addParameter}
							disabled={creatingModel}
						>
							+ {$i18n.t('Add Parameter')}
						</button>
					</div>
					{#each newModel.parameters as param, i}
						<div class="flex gap-2 mb-2">
							<div class="flex-1">
								<label for="param-name-{i}" class="sr-only">{$i18n.t('Parameter Name')}</label>
								<select
									id="param-name-{i}"
									class="w-full p-2 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 text-sm"
									bind:value={param.name}
									disabled={creatingModel}
								>
									<option value="" disabled selected>{$i18n.t('Select parameter')}</option>
									{#each OLLAMA_PARAMETERS as option}
										<option value={option.name}>{option.name}</option>
									{/each}
								</select>
								{#if param.name}
									<div class="mt-1 text-xs text-gray-500 dark:text-gray-400">
										{OLLAMA_PARAMETERS.find((p) => p.name === param.name)?.description}
										<span class="ml-1 font-mono">({param.type})</span>
									</div>
								{/if}
							</div>
							<div class="flex-1">
								<label for="param-value-{i}" class="sr-only">{$i18n.t('Parameter Value')}</label>
								<input
									type="text"
									id="param-value-{i}"
									class="w-full p-2 rounded-lg bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-800 text-sm"
									bind:value={param.value}
									placeholder={param.name
										? OLLAMA_PARAMETERS.find((p) => p.name === param.name)?.default
										: $i18n.t('Value')}
									disabled={creatingModel}
								/>
							</div>
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
					{#if selectedModelInfo}
						<button
							class="px-4 py-2 bg-gray-100 hover:bg-gray-200 dark:bg-gray-800 dark:hover:bg-gray-700 rounded-lg text-sm"
							on:click={loadSelectedModelDetails}
							disabled={creatingModel}
						>
							{$i18n.t('Load Selected Model')}
						</button>
						{#if selectedModelInfo?.modelfile}
							<button
								class="px-4 py-2 bg-gray-100 hover:bg-gray-200 dark:bg-gray-800 dark:hover:bg-gray-700 rounded-lg text-sm"
								on:click={useSelectedAsTemplate}
								disabled={creatingModel}
							>
								{$i18n.t('Use Selected as Template')}
							</button>
						{/if}
					{/if}
					<button
						class="px-4 py-2 bg-gray-100 hover:bg-gray-200 dark:bg-gray-800 dark:hover:bg-gray-700 rounded-lg text-sm"
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
					<button
						class="px-4 py-2 bg-primary hover:bg-primary-dark text-white rounded-lg text-sm disabled:opacity-50 disabled:cursor-not-allowed"
						on:click={createAndLoadModel}
						disabled={creatingModel}
					>
						{#if creatingModel}
							<div class="flex items-center">
								<Spinner size="sm" class="mr-2" />
								{$i18n.t('Creating and Loading...')}
							</div>
						{:else}
							{$i18n.t('Create and Load Model')}
						{/if}
					</button>
				</div>
			</div>
		</div>
	</div>
</div>

<!-- Add the QuickTestModal component -->
<QuickTestModal
	bind:show={showQuickTest}
	modelId={selectedModelId}
	on:close={() => {
		showQuickTest = false;
	}}
/>
