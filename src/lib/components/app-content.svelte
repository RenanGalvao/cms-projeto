<script lang="ts">
	import AppHeader from '$components/app-header.svelte';
	import AppActions from '$components/app-actions.svelte';

	import AppContentLoading from '$components/app-content-loading.svelte';
	import { createEventDispatcher } from 'svelte';
	import type { Locales } from '$src/i18n/i18n-types';

	// Component Data - forwarding
	export let name: string;
	export let buttonText: string | null = null;
	export let buttonLink: string | null = null;
	export let page = 1;
	export let maxPage = 1;
	export let totalCount = 1;
	export let showDeleted = false;
	export let itemsSelected: string[] = [];
	export let search = '';
	export let searchMinLength = 3;
	export let searchType: 'text' | 'number' = 'text';
	export let locale: Locales;

	// Component Options - forwarding
	export let isLoading = false;
	export let showBackButton = true;
	export let showRefreshButton = true;
	export let baseRoute = '';

	// Component Options
	export let showActions = true;

	// Events - forwarding
	const dispatch = createEventDispatcher();
	const click = () => dispatch('click');
	const refresh = () => dispatch('refresh');
	const restore = () => dispatch('restore');
	const remove = () => dispatch('remove');
	const searchLoad = () => dispatch('searchLoad');
	const searchClear = () => dispatch('searchClear');

	$: appHeader = {
		name,
		buttonText,
		buttonLink
	};
</script>

<div class="app-content">
	<AppHeader
		{...appHeader}
		{showBackButton}
		{showRefreshButton}
		{locale}
		on:click={click}
		on:refresh={refresh}
	/>
	{#if showActions}
		<AppActions
			{maxPage}
			{totalCount}
			{baseRoute}
			{searchMinLength}
			{searchType}
			{locale}
			bind:search
			bind:itemsSelected
			bind:page
			bind:showDeleted
			on:remove={remove}
			on:restore={restore}
			on:searchLoad={searchLoad}
			on:searchClear={searchClear}
		/>
	{/if}
	<div class="products-area-wrapper tableView">
		<slot />
	</div>
	<AppContentLoading {isLoading} />
</div>
