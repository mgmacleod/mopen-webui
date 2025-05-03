<script lang="ts">
	import { getContext } from 'svelte';
	const i18n = getContext('i18n');

	export let servers: string[] = [];
	export let selectedServer: string | null = null;

	// Event dispatcher for server selection
	import { createEventDispatcher } from 'svelte';
	const dispatch = createEventDispatcher();

	function handleServerChange(event: Event) {
		const select = event.target as HTMLSelectElement;
		selectedServer = select.value;
		dispatch('serverChange', { server: selectedServer });
	}
</script>

<div class="mb-4 flex items-center gap-2">
	<label for="server-select" class="text-sm font-medium text-gray-700 dark:text-gray-300">
		{$i18n.t('Ollama Server')}:
	</label>
	<select
		id="server-select"
		class="flex-1 bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-700 rounded px-3 py-1 text-sm"
		on:change={handleServerChange}
		value={selectedServer}
	>
		<option value="">{$i18n.t('All Servers')}</option>
		{#each servers as server}
			<option value={server}>{server}</option>
		{/each}
	</select>
</div>
