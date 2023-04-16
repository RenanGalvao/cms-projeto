<script lang="ts">
	import '$lib/scss/dashboard.scss';
	import AppContainer from '$components/app-container.svelte';
	import AppContent from '$components/app-content.svelte';
	import CollectionHeader from '$components/collection-header.svelte';
	import CollectionRow from '$components/collection-row.svelte';

	import { getContext, onMount } from 'svelte';
	import type { Pagination, RowCell, TokenDto } from '$lib/types';
	import { delay, friendlyDateString, fromPaginationToQuery } from '$lib/utils/functions';

	import { generateMessages } from '$components/toast.svelte';
	import axios from '$lib/axios';
	import Axios from 'axios';
	import type { UserStore } from '$src/lib/store/user';
	import type { PageData } from './$types';

	export let data: PageData;

	let tokenData: TokenDto[] = [];
	let userStore = getContext<UserStore>('userStore');
	let totalCount = 1;
	let totalPages = 1;
	let isReady = false;

	let isLoading = true;
	let messages: any[] = [];
	let itemsSelected: string[] = [];

	onMount(async () => {
		const accessToken = userStore.get('accessToken');
		axios.setAuth(accessToken);
		isReady = true;
	});

	// Pagination config
	let pagination = {
		itemsPerPage: 20,
		page: 1,
		deleted: false,
		orderKey: 'createdAt',
		orderValue: 'desc',
		search: '',
	} as Pagination;
	let searchInput = '';

	$: queryString = fromPaginationToQuery(pagination);
	$: queryString, loadData();

	async function loadData() {
		while (!isReady) {
			await delay(50);
		}

		try {
			isLoading = true;

			const res = await axios.get(`token?${queryString}`);
			tokenData = res.data.data;
			totalCount = res.headers.get('x-total-count');
			totalPages = res.headers.get('x-total-pages');

			isLoading = false;
		} catch (error) {
			isLoading = false;
			if (error instanceof Axios.AxiosError) {
				messages = generateMessages([{ message: error.response?.data.message }]);
			} else {
				console.warn(error);
			}
		}
	}

	// App Header
	const appHeader = {
		name: 'Tokens'
	};

	// Collection Header
	const collectionHeader = [
		{
			label: 'Token',
			key: 'token'
		},
		{
			label: 'E-mail',
			key: 'email'
		},
		{
			label: 'Usado',
			key: 'used'
		},
		{
			label: 'Tipo',
			key: 'tokenType'
		},
		{
			label: 'Payload',
			key: 'payload'
		},
		{
			label: 'Expiração',
			key: 'expiration'
		},
		{
			label: 'Criado Em',
			key: 'createdAt'
		},
		{
			label: 'Atualizado Em',
			key: 'updatedAt'
		},
		{
			label: 'Deletado Em',
			key: 'deleted'
		}
	];

	$: collectionData = Object.entries(tokenData).map(
		([key, data]) =>
			[
				{
					label: 'id',
					key: 'id',
					value: data.id
				},
				{
					label: 'Token',
					key: 'token',
					value: data.token
				},
				{
					label: 'E-mail',
					key: 'email',
					value: data.email,
					isTag: true
				},
				{
					label: 'Usado',
					key: 'used',
					value: data.used,
					isTag: true
				},
				{
					label: 'Tipo',
					key: 'tokenType',
					value: data.tokenType,
					isTag: true,
				},
				{
					label: 'Payload',
					key: 'payload',
					value: data.payload,
					isJson: true
				},
				{
					label: 'Expiração',
					key: 'expiration',
					value: data.expiration,
					transform: (value: number) => friendlyDateString(new Date(Date.now() + value))
				},
				{
					label: 'Criado Em',
					key: 'createdAt',
					value: data.createdAt,
					transform: friendlyDateString
				},
				{
					label: 'Atualizado Em',
					key: 'updatedAt',
					value: data.updatedAt,
					transform: friendlyDateString
				},
				{
					label: 'Deletado Em',
					key: 'deleted',
					value: data.deleted,
					transform: friendlyDateString
				}
			] as RowCell[]
	);

	function onSort(event: CustomEvent) {
		const key = event.detail;
		let orderValue;

		if (key === pagination.orderKey) {
			orderValue = pagination.orderValue === 'desc' ? 'asc' : 'desc';
		} else {
			orderValue = 'desc';
		}

		pagination = {
			...pagination,
			orderKey: key,
			orderValue
		} as Pagination;
	}

	function onSearchLoad() {
		pagination.search = searchInput
	}

	function onSearchClear() {
		pagination.search = '';
	}
</script>

<svelte:head>
	<title>Tokens</title>
</svelte:head>

<AppContainer {messages} locale={data.locale}>
	<AppContent
		{...appHeader}
		{totalCount}
		showBackButton={false}
		maxPage={totalPages}
		baseRoute={'/token'}
		on:refresh={loadData}
		on:restore={loadData}
		on:remove={loadData}
		on:searchLoad={onSearchLoad}
		on:searchClear={onSearchClear}
		bind:search={searchInput}
		bind:page={pagination.page}
		bind:showDeleted={pagination.deleted}
		bind:itemsSelected
		bind:isLoading
	>
		<CollectionHeader
			columns={collectionHeader}
			showOptions={false}
			on:click={onSort}
			bind:showDeleted={pagination.deleted}
		/>
		{#each collectionData as row, i (row[0].value)}
			<CollectionRow rowCells={row} showOptions={false} bind:showDeleted={pagination.deleted} />
		{/each}
	</AppContent>
</AppContainer>