<script lang="ts">
	import '$lib/scss/dashboard.scss';
	import AppContainer from '$components/app-container.svelte';
	import AppContent from '$components/app-content.svelte';
	import CollectionHeader from '$components/collection-header.svelte';
	import CollectionRow from '$components/collection-row.svelte';
	import CollectionRowPlaceholder from '$components/collection-row-placeholder.svelte';

	import { getContext, onMount } from 'svelte';
	import type { Pagination, RowCell, UserDto } from '$lib/types';
	import {
		delay,
		friendlyDateString,
		fromPaginationToQuery,
		removeItemById
	} from '$lib/utils/functions';

	import { generateMessages } from '$components/toast.svelte';
	import axios from '$lib/axios';
	import Axios from 'axios';
	import { goto } from '$app/navigation';
	import type { PageData } from './$types';
	import type { UserStore } from '$src/lib/store/user';

	// i18n
	import { loadNamespaceAsync } from '$i18n/i18n-util.async';
	import LL, { setLocale } from '$i18n/i18n-svelte';
	$: i18n = $LL.users.list;
	$: sharedI18n = $LL.shared;

	export let data: PageData;

	let userData: UserDto[] = [];
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

		if (userStore.isVolunteer() || userStore.isAdmin()) {
			pagination.searchSpecificField = 'fieldId';
			pagination.searchSpecificValue = userStore.get('user.fieldId');
		}

		await loadNamespaceAsync(data.locale, 'users');
		await loadNamespaceAsync(data.locale, 'shared');
		setLocale(data.locale);
	});

	// Pagination config
	let pagination = {
		itemsPerPage: 20,
		page: 1,
		deleted: false,
		orderKey: 'firstName',
		orderValue: 'asc',
		search: ''
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

			const res = await axios.get(`user?${queryString}`);
			userData = res.data.data;
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
	$: appHeader = {
		name: i18n.appHeader.name(),
		buttonText: i18n.appHeader.buttonText(),
		buttonLink: `/${data.locale}/users/add`
	};

	// Collection Header
	$: collectionHeader = [
		{
			label: i18n.collectionHeader.firstNameLabel(),
			key: 'firstName'
		},
		{
			label: i18n.collectionHeader.lastNameLabel(),
			key: 'lastName'
		},
		{
			label: i18n.collectionHeader.emailLabel(),
			key: 'email'
		},
		{
			label: i18n.collectionHeader.roleLabel(),
			key: 'role'
		},
		{
			label: i18n.collectionHeader.lastAccessLabel(),
			key: 'lastAccess'
		},
		{
			label: sharedI18n.collectionHeader.createdAtLabel(),
			key: 'createdAt'
		},
		{
			label: sharedI18n.collectionHeader.updatedAtLabel(),
			key: 'updatedAt'
		},
		{
			label: sharedI18n.collectionHeader.deletedLabel(),
			key: 'deleted'
		}
	];

	$: collectionData = Object.entries(userData).map(
		([key, item]) =>
			[
				{
					label: 'id',
					key: 'id',
					value: item.id
				},
				{
					label: i18n.collectionHeader.firstNameLabel(),
					key: 'firstName',
					value: item.firstName
				},
				{
					label: i18n.collectionHeader.lastNameLabel(),
					key: 'lastName',
					value: item.lastName ?? i18n.lastNamePlaceholder()
				},
				{
					label: i18n.collectionHeader.emailLabel(),
					key: 'email',
					value: item.email
				},
				{
					label: i18n.collectionHeader.roleLabel(),
					key: 'role',
					value: item.role,
					isTag: true
				},
				{
					label: i18n.collectionHeader.lastAccessLabel(),
					key: 'lastAccess',
					value: item.lastAccess,
					transform: (value: string) => friendlyDateString(value, data.locale)
				},
				{
					label: sharedI18n.collectionHeader.createdAtLabel(),
					key: 'createdAt',
					value: item.createdAt,
					transform: (value: string) => friendlyDateString(value, data.locale)
				},
				{
					label: sharedI18n.collectionHeader.updatedAtLabel(),
					key: 'updatedAt',
					value: item.updatedAt,
					transform: (value: string) => friendlyDateString(value, data.locale)
				},
				{
					label: sharedI18n.collectionHeader.deletedLabel(),
					key: 'deleted',
					value: item.deleted,
					transform: (value: string) => friendlyDateString(value, data.locale)
				}
			] as RowCell[]
	);

	// On Event Functions
	function handleEdit(event: CustomEvent) {
		const id = event.detail;
		goto(`/${data.locale}/users/edit?id=${id}`);
	}

	async function handleRemove(event: CustomEvent) {
		const { id, data } = event.detail;
		const remove = confirm(sharedI18n.remove.user({ email: data.email }));

		if (remove) {
			isLoading = true;
			try {
				await axios.delete(`/user/${id}`);
				userData = removeItemById(id, userData);
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
	}

	function handleSelect(event: CustomEvent) {
		const id = event.detail;
		if (itemsSelected.indexOf(id) > -1) {
			itemsSelected = [...itemsSelected.filter((itemId) => itemId !== id)];
		} else {
			itemsSelected = [...itemsSelected, id];
		}
	}

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
		pagination.search = searchInput;
	}

	function onSearchClear() {
		pagination.search = '';
	}
</script>

<svelte:head>
	<title>{i18n.pageTitle()}</title>
</svelte:head>

<AppContainer {messages} locale={data.locale}>
	<AppContent
		{...appHeader}
		{totalCount}
		showBackButton={false}
		maxPage={totalPages}
		baseRoute={'/user'}
		locale={data.locale}
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
			locale={data.locale}
			on:click={onSort}
			bind:showDeleted={pagination.deleted}
		/>
		{#each collectionData as row, i (row[0].value)}
			<CollectionRow
				rowCells={row}
				locale={data.locale}
				on:edit={handleEdit}
				on:remove={handleRemove}
				on:select={handleSelect}
				bind:showDeleted={pagination.deleted}
			/>
		{/each}
		{#if collectionData.length === 0 && !isLoading}
			<CollectionRowPlaceholder
				locale={data.locale}
				buttonLink={appHeader.buttonLink}
				buttonText={appHeader.buttonText}
			/>
		{/if}
	</AppContent>
</AppContainer>
