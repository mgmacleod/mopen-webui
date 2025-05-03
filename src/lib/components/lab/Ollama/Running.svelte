<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { onMount, onDestroy, getContext } from 'svelte';
	const i18n = getContext('i18n');

	import Spinner from '$lib/components/common/Spinner.svelte';
	import { getRunningModels, stopModel } from '$lib/apis/ollama';
	import ServerSelector from './ServerSelector.svelte';

	type ModelDetails = {
		parent_model: string;
		format: string;
		family: string;
		families: string[];
		parameter_size: string;
		quantization_level: string;
	};

	type RunningModelResponse = {
		name: string;
		model: string;
		size: number;
		size_vram: number;
		digest: string;
		details: ModelDetails;
		expires_at: string;
	};

	type RunningModel = RunningModelResponse & {
		url: string;
	};

	let loading = true;
	let runningModels: RunningModel[] = [];
	let selectedModel: RunningModel | null = null;
	let refreshInterval: ReturnType<typeof setInterval>;
	let selectedServer: string | null = null;
	let servers: string[] = [];

	// Calculate total VRAM and RAM usage
	$: totalVRAM = filteredModels.reduce((sum, model) => sum + model.size_vram, 0);
	$: totalRAM = filteredModels.reduce((sum, model) => {
		const ramUsage = Math.max(0, model.size - model.size_vram);
		return sum + ramUsage;
	}, 0);

	// Check if any models are using RAM
	$: hasRAMUsage = filteredModels.some((model) => model.size > model.size_vram);

	// Filter models based on selected server
	$: filteredModels = selectedServer
		? runningModels.filter((model) => model.url === selectedServer)
		: runningModels;

	// Format bytes to human readable size (GB, MB, etc)
	function formatSize(bytes: number): string {
		const units = ['B', 'KB', 'MB', 'GB', 'TB'];
		let size = bytes;
		let unitIndex = 0;
		while (size >= 1024 && unitIndex < units.length - 1) {
			size /= 1024;
			unitIndex++;
		}
		return `${size.toFixed(1)} ${units[unitIndex]}`;
	}

	// Calculate percentage of total size
	function formatPercentage(part: number, total: number): string {
		if (total === 0) return '0%';
		return `${Math.round((part / total) * 100)}%`;
	}

	// Format expiry time to human readable format or "Forever" if far in the future
	function formatExpiry(expiryDate: string): string {
		const expiry = new Date(expiryDate);
		const now = new Date();
		const diff = expiry.getTime() - now.getTime();
		// If more than 10 years in the future, show "Forever"
		if (diff > 315576000000) {
			return 'Forever';
		}
		return expiry.toLocaleString();
	}

	async function fetchRunningModels() {
		try {
			const data = await getRunningModels(localStorage.token);
			// Transform the response into a flat array of running models
			runningModels = Object.entries(data).flatMap(([url, response]) => {
				// Each response is an array of models
				return (response as { models: RunningModelResponse[] }).models.map((model) => ({
					...model,
					url
				}));
			});
			// Update servers list with formatted labels
			servers = Object.keys(data);
			// Keep selected model updated if it exists in the new list
			if (selectedModel) {
				selectedModel = runningModels.find((m) => m.digest === selectedModel?.digest) || null;
			}
		} catch (error) {
			toast.error($i18n.t('Failed to fetch running models: {0}', { values: [error] }));
		} finally {
			loading = false;
		}
	}

	async function handleStopModel(model: RunningModel) {
		try {
			// Find the server index for the model
			const serverIndex = servers.indexOf(model.url);
			if (serverIndex === -1) {
				throw new Error('Server not found');
			}
			await stopModel(localStorage.token, model.model, serverIndex);
			toast.success($i18n.t('Stopped model: {0}', { values: [model.name] }));
			if (selectedModel?.digest === model.digest) {
				selectedModel = null;
			}
			await fetchRunningModels();
		} catch (error) {
			toast.error($i18n.t('Failed to stop model: {0}', { values: [error] }));
		}
	}

	onMount(() => {
		fetchRunningModels();
		refreshInterval = setInterval(fetchRunningModels, 5000); // Refresh every 5 seconds
	});

	onDestroy(() => {
		if (refreshInterval) clearInterval(refreshInterval);
	});
</script>

<div class="h-full p-4">
	<div class="mb-4 flex justify-between items-center">
		<h2 class="text-lg font-medium text-gray-800 dark:text-gray-200">
			{$i18n.t('Running Models')}
		</h2>
		{#if !loading && runningModels.length > 0}
			<div class="text-sm text-gray-600 dark:text-gray-300 space-x-4">
				<span>{$i18n.t('Total VRAM')}: {formatSize(totalVRAM)}</span>
				{#if hasRAMUsage}
					<span>{$i18n.t('Total RAM')}: {formatSize(totalRAM)}</span>
				{/if}
			</div>
		{/if}
	</div>

	{#if servers.length > 1}
		<ServerSelector {servers} bind:selectedServer on:serverChange={() => (selectedModel = null)} />
	{/if}

	{#if loading}
		<div class="flex items-center justify-center h-64">
			<Spinner size="lg" />
		</div>
	{:else if filteredModels.length === 0}
		<div class="flex items-center justify-center h-64 text-gray-500 dark:text-gray-400">
			{#if selectedServer}
				{$i18n.t('No models running on selected server')}
			{:else}
				{$i18n.t('No models currently running')}
			{/if}
		</div>
	{:else}
		<div class="space-y-4">
			<!-- List View -->
			<div class="overflow-x-auto">
				<table class="w-full text-sm">
					<thead>
						<tr
							class="text-left text-gray-500 dark:text-gray-400 border-b border-gray-200 dark:border-gray-800"
						>
							<th class="pb-2 font-medium">Name</th>
							<th class="pb-2 font-medium">ID</th>
							{#if !selectedServer}
								<th class="pb-2 font-medium">Server</th>
							{/if}
							<th class="pb-2 font-medium">Size</th>
							<th class="pb-2 font-medium">VRAM</th>
							{#if hasRAMUsage}
								<th class="pb-2 font-medium">RAM</th>
							{/if}
							<th class="pb-2 font-medium">Until</th>
							<th class="pb-2 font-medium"></th>
						</tr>
					</thead>
					<tbody>
						{#each filteredModels as model}
							<tr
								class="border-b border-gray-100 dark:border-gray-800 hover:bg-gray-50 dark:hover:bg-gray-900 cursor-pointer {selectedModel?.digest ===
								model.digest
									? 'bg-gray-50 dark:bg-gray-900'
									: ''}"
								on:click={() =>
									(selectedModel = selectedModel?.digest === model.digest ? null : model)}
							>
								<td class="py-3 font-medium">{model.name}</td>
								<td class="py-3 font-mono">{model.digest.slice(0, 12)}</td>
								{#if !selectedServer}
									<td class="py-3">{model.url}</td>
								{/if}
								<td class="py-3">{formatSize(model.size)}</td>
								<td class="py-3">
									{formatSize(model.size_vram)}
									<span class="text-gray-500 dark:text-gray-400 ml-1">
										({formatPercentage(model.size_vram, model.size)})
									</span>
								</td>
								{#if hasRAMUsage}
									<td class="py-3">
										{formatSize(Math.max(0, model.size - model.size_vram))}
										<span class="text-gray-500 dark:text-gray-400 ml-1">
											({formatPercentage(model.size - model.size_vram, model.size)})
										</span>
									</td>
								{/if}
								<td class="py-3">{formatExpiry(model.expires_at)}</td>
								<td class="py-3">
									<button
										class="px-2 py-1 text-sm font-medium text-red-600 hover:text-red-700 dark:text-red-400 dark:hover:text-red-300"
										on:click|stopPropagation={() => handleStopModel(model)}
									>
										{$i18n.t('Stop')}
									</button>
								</td>
							</tr>
						{/each}
					</tbody>
				</table>
			</div>

			<!-- Detail View -->
			{#if selectedModel}
				<div class="mt-6 bg-white dark:bg-gray-800 rounded-lg shadow p-4">
					<div class="space-y-4">
						<div class="flex items-center justify-between">
							<h3 class="text-lg font-medium text-gray-900 dark:text-gray-100">
								{selectedModel.name}
							</h3>
							<button
								class="px-3 py-1.5 text-sm font-medium text-red-600 hover:text-red-700 dark:text-red-400 dark:hover:text-red-300"
								on:click={() => handleStopModel(selectedModel)}
							>
								{$i18n.t('Stop Model')}
							</button>
						</div>

						<div class="grid grid-cols-2 gap-4">
							<div>
								<h4 class="text-sm font-medium text-gray-500 dark:text-gray-400 mb-2">
									{$i18n.t('Model Information')}
								</h4>
								<div class="space-y-2">
									<div>
										<span class="text-sm text-gray-500 dark:text-gray-400">{$i18n.t('ID')}:</span>
										<span class="ml-2 font-mono">{selectedModel.digest}</span>
									</div>
									<div>
										<span class="text-sm text-gray-500 dark:text-gray-400">{$i18n.t('Size')}:</span>
										<span class="ml-2">{formatSize(selectedModel.size)}</span>
									</div>
									<div>
										<span class="text-sm text-gray-500 dark:text-gray-400">VRAM:</span>
										<span class="ml-2">{formatSize(selectedModel.size_vram)}</span>
									</div>
									<div>
										<span class="text-sm text-gray-500 dark:text-gray-400"
											>{$i18n.t('Server')}:</span
										>
										<span class="ml-2">{selectedModel.url}</span>
									</div>
									<div>
										<span class="text-sm text-gray-500 dark:text-gray-400"
											>{$i18n.t('Expires')}:</span
										>
										<span class="ml-2">{formatExpiry(selectedModel.expires_at)}</span>
									</div>
								</div>
							</div>

							<div>
								<h4 class="text-sm font-medium text-gray-500 dark:text-gray-400 mb-2">
									{$i18n.t('Model Details')}
								</h4>
								<div class="space-y-2">
									{#if selectedModel.details.parent_model}
										<div>
											<span class="text-sm text-gray-500 dark:text-gray-400"
												>{$i18n.t('Parent Model')}:</span
											>
											<span class="ml-2">{selectedModel.details.parent_model}</span>
										</div>
									{/if}
									<div>
										<span class="text-sm text-gray-500 dark:text-gray-400"
											>{$i18n.t('Format')}:</span
										>
										<span class="ml-2 uppercase">{selectedModel.details.format}</span>
									</div>
									<div>
										<span class="text-sm text-gray-500 dark:text-gray-400"
											>{$i18n.t('Family')}:</span
										>
										<span class="ml-2">{selectedModel.details.family}</span>
									</div>
									<div>
										<span class="text-sm text-gray-500 dark:text-gray-400"
											>{$i18n.t('Parameter Size')}:</span
										>
										<span class="ml-2">{selectedModel.details.parameter_size}</span>
									</div>
									<div>
										<span class="text-sm text-gray-500 dark:text-gray-400"
											>{$i18n.t('Quantization')}:</span
										>
										<span class="ml-2">{selectedModel.details.quantization_level}</span>
									</div>
								</div>
							</div>
						</div>
					</div>
				</div>
			{/if}
		</div>
	{/if}
</div>
