<script lang="ts">
	import { page } from '$app/stores';
	import { user } from '$lib/stores';
	import { getContext, onMount } from 'svelte';

	const i18n = getContext('i18n');

	onMount(() => {
		if ($user?.role !== 'admin') {
			location.href = '/';
		}
	});
</script>

<div class="flex flex-col flex-1 overflow-hidden">
	<div class="p-2 bg-white dark:bg-gray-900 border-b border-gray-100 dark:border-gray-800">
		<div class="flex items-center">
			<div class="text-lg font-semibold text-gray-800 dark:text-gray-200 mr-4">
				{$i18n.t('Lab')}
			</div>
			<div class="flex space-x-2">
				<a
					href="/lab/ollama"
					class="px-3 py-1 text-sm rounded-md {$page.url.pathname.includes('/lab/ollama')
						? 'bg-primary text-white'
						: 'text-gray-700 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-800'}"
				>
					{$i18n.t('Ollama')}
				</a>
				<!-- Add more tabs here as needed -->
			</div>
		</div>
	</div>
	<div class="flex-1 overflow-auto">
		<slot />
	</div>
</div>
