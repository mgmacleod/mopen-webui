<script lang="ts">
	import { page } from '$app/stores';
	import { user, showSidebar, WEBUI_NAME } from '$lib/stores';
	import { getContext, onMount } from 'svelte';
	import MenuLines from '$lib/components/icons/MenuLines.svelte';

	const i18n = getContext('i18n');

	onMount(() => {
		if ($user?.role !== 'admin') {
			location.href = '/';
		}
	});
</script>

<svelte:head>
	<title>
		{$i18n.t('Lab')} | {$WEBUI_NAME}
	</title>
</svelte:head>

<div
	class="flex flex-col w-full h-screen max-h-[100dvh] transition-width duration-200 ease-in-out {$showSidebar
		? 'md:max-w-[calc(100%-260px)]'
		: ''} max-w-full"
>
	<nav class="px-2.5 pt-1 backdrop-blur-xl drag-region">
		<div class="flex items-center">
			<div class="{$showSidebar ? 'md:hidden' : ''} flex flex-none items-center self-end">
				<button
					id="sidebar-toggle-button"
					class="cursor-pointer p-1.5 flex rounded-xl hover:bg-gray-100 dark:hover:bg-gray-850 transition"
					on:click={() => {
						showSidebar.set(!$showSidebar);
					}}
					aria-label="Toggle Sidebar"
				>
					<div class="m-auto self-center">
						<MenuLines />
					</div>
				</button>
			</div>

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
	</nav>

	<div class="flex-1 overflow-auto">
		<slot />
	</div>
</div>
