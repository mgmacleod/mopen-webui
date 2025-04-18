<script lang="ts">
	import { getContext } from 'svelte';
	const i18n = getContext('i18n');

	import Models from './Models.svelte';
	import Running from './Running.svelte';

	let activeTab: 'models' | 'running' = 'models';
</script>

<div class="flex flex-col h-full">
	<!-- Tab Navigation -->
	<div class="border-b border-gray-200 dark:border-gray-800">
		<nav class="flex space-x-4 px-4" aria-label={$i18n.t('Ollama Management Tabs')}>
			<button
				class="px-3 py-2 text-sm font-medium rounded-t-lg {activeTab === 'models'
					? 'text-primary border-b-2 border-primary'
					: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300'}"
				on:click={() => (activeTab = 'models')}
				aria-current={activeTab === 'models' ? 'page' : undefined}
			>
				{$i18n.t('Models')}
			</button>
			<button
				class="px-3 py-2 text-sm font-medium rounded-t-lg {activeTab === 'running'
					? 'text-primary border-b-2 border-primary'
					: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300'}"
				on:click={() => (activeTab = 'running')}
				aria-current={activeTab === 'running' ? 'page' : undefined}
			>
				{$i18n.t('Running')}
			</button>
		</nav>
	</div>

	<!-- Tab Content -->
	<div class="flex-1 overflow-hidden">
		{#if activeTab === 'models'}
			<Models />
		{:else if activeTab === 'running'}
			<Running />
		{/if}
	</div>
</div>
