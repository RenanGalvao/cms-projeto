<script lang="ts">
	import '$lib/scss/dashboard.scss';
	import '$lib/scss/dashboard-form.scss';
	import AppContainer from '$components/app-container.svelte';
	import AppContent from '$components/app-content.svelte';
	import { generateMessages } from '$components/toast.svelte';

	import Icon from 'svelte-icons-pack/Icon.svelte';
	import HiOutlineMail from "svelte-icons-pack/hi/HiOutlineMail";
	import HiOutlineUser from "svelte-icons-pack/hi/HiOutlineUser";
	import HiOutlineGlobe from "svelte-icons-pack/hi/HiOutlineGlobe";
	import HiOutlineIdentification from "svelte-icons-pack/hi/HiOutlineIdentification";

	import { onMount } from 'svelte';
	import { Role } from '$lib/enums';

	import { MESSAGES, TEMPLATES } from '$src/lib/constants';
	import * as yup from 'yup';
	import axios from '$lib/axios';
	import Axios from 'axios';
	import type { PageData } from './$types';
	import type { CreateMailPayloadDto, FieldDto, Pagination } from '$src/lib/types';
	import { fromPaginationToQuery } from '$src/lib/utils/functions';

	export let data: PageData;

	// App Content Options
	const showActions = false;
	const showRefreshButton = false;

	// App Header
	const appHeader = {
		name: 'Adicionar Usuário',
		buttonText: 'Enviar Email'
	};

	const query = {
		itemsPerPage: 100,
		page: 1,
		deleted: false,
		orderKey: 'designation',
		orderValue: 'asc'
	} as Pagination;
	$: queryString = fromPaginationToQuery(query);

	// Form
	let formRef: HTMLFormElement;
	let isLoading = false;
	let fields: FieldDto[] = [];
	const roles = [
		{ value: 'VOLUNTEER', text: 'Voluntário' },
		{ value: 'ADMIN', text: 'Admin' },
		{ value: 'WEB_MASTER', text: 'Web Master' }
	];
	const schema = yup.object().shape({
		email: yup.string().required(TEMPLATES.YUP.REQUIRED('Email')).email(MESSAGES.YUP.EMAIL),
		name: yup.string().required(TEMPLATES.YUP.REQUIRED('Nome')),
		payload: yup.object().shape({
			field: yup.string().optional(),
			role: yup
				.string()
				.oneOf(Object.values(Role), TEMPLATES.YUP.ONE_OF(roles.map((r) => r.text)))
				.required(TEMPLATES.YUP.REQUIRED('Acesso'))
		})
	});

	let email: string;
	let name: string;
	let payload: CreateMailPayloadDto = {
		role: null,
		field: null
	};

	let messages: any[] = [];

	onMount(async () => {
		await loadData();
	});

	async function loadData() {
		try {
			isLoading = true;
			fields = (await axios.get(`/field?${queryString}`)).data.data;
			isLoading = false;
		} catch (error) {
			isLoading = false;

			if (error instanceof Axios.AxiosError) {
				messages = generateMessages([{ message: error.response?.data.message }]);
			} else if (error instanceof yup.ValidationError) {
				messages = generateMessages(error.inner.map((err) => ({ message: err.message })));
			} else {
				console.warn(error);
			}
		}
	}

	async function onSubmit(event: Event) {
		event.preventDefault();
		event.stopPropagation();
		isLoading = true;

		try {
			const isValid = schema.validateSync({ name, email, payload }, { abortEarly: false });

			if (isValid) {
				const res = await axios.post('/auth/create-email', {
					name,
					email,
					payload
				});

				isLoading = false;
				messages = generateMessages([{ message: res.data.message, variant: 'success' }]);
				eraseForm();
			}
		} catch (error) {
			isLoading = false;

			if (error instanceof Axios.AxiosError) {
				messages = generateMessages([{ message: error.response?.data.message }]);
			} else if (error instanceof yup.ValidationError) {
				messages = generateMessages(error.inner.map((err) => ({ message: err.message })));
			} else {
				console.warn(error);
			}
		}
	}

	function eraseForm() {
		formRef.reset();
		payload.field = null;
		payload.role = null;
	}
</script>

<svelte:head>
	<title>Dashboard | Usuários</title>
</svelte:head>

<AppContainer {messages} locale={data.locale}>
	<AppContent {...appHeader} {isLoading} on:click={onSubmit} {showActions} {showRefreshButton}>
		<form bind:this={formRef} on:submit|preventDefault|stopPropagation={onSubmit} class="app-form">
			<div class="input">
				<Icon src={HiOutlineUser} />
				<input
					bind:value={name}
					name="name"
					type="text"
					placeholder="Nome"
					autocomplete="off"
					required
				/>
			</div>
			<div class="input">
				<Icon src={HiOutlineMail} />
				<input
					bind:value={email}
					name="email"
					type="email"
					placeholder="Email"
					autocomplete="off"
					required
				/>
			</div>
			<div class="input">
				<Icon src={HiOutlineGlobe} />
				<select bind:value={payload.field} name="field" required>
					<option value={null} disabled selected>Campo Missionário</option>

					{#each fields as field}
						<option value={field.id}>
							{field.abbreviation} - {field.designation}
						</option>
					{/each}
				</select>
			</div>
			<div class="input">
				<Icon src={HiOutlineIdentification} />
				<select bind:value={payload.role} name="role" required>
					<option value={null} disabled selected>Acesso</option>
					{#each roles as role}
						<option value={role.value}>
							{role.text}
						</option>
					{/each}
				</select>
			</div>
		</form>
	</AppContent>
</AppContainer>