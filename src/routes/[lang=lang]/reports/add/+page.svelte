<script lang="ts">
	import '$lib/scss/dashboard.scss';
	import '$lib/scss/dashboard-form.scss';
	import AppContainer from '$components/app-container.svelte';
	import AppContent from '$components/app-content.svelte';
	import { generateMessages } from '$components/toast.svelte';

	import Icon from 'svelte-icons-pack/Icon.svelte';
	import HiOutlinePencil from 'svelte-icons-pack/hi/HiOutlinePencil';
	import HiOutlineMenuAlt2 from 'svelte-icons-pack/hi/HiOutlineMenuAlt2';
	import HiOutlinePaperClip from 'svelte-icons-pack/hi/HiOutlinePaperClip';
	import HiOutlineCalendar from 'svelte-icons-pack/hi/HiOutlineCalendar';
	import HiOutlineTag from 'svelte-icons-pack/hi/HiOutlineTag';
	import HiOutlineGlobe from 'svelte-icons-pack/hi/HiOutlineGlobe';

	import { getContext, onMount } from 'svelte';

	import { TEMPLATES } from '$src/lib/constants';
	import * as yup from 'yup';
	import axios from '$lib/axios';
	import Axios from 'axios';
	import type { PageData } from '../../users/add/$types';
	import type { FieldDto, FileDto, Pagination } from '$src/lib/types';
	import { fromPaginationToQuery } from '$src/lib/utils/functions';
	import type { UserStore } from '$src/lib/store/user';
	import { ReportType } from '$src/lib/enums';

	export let data: PageData;
	let userStore = getContext<UserStore>('userStore');
	let isAdminOrVolunteer = true;

	// App Content Options
	const showActions = false;
	const showRefreshButton = true;

	// App Header
	const appHeader = {
		name: 'Adicionar Relatório',
		buttonText: 'Salvar'
	};

	const query = {
		itemsPerPage: 100,
		page: 1,
		deleted: false,
		orderKey: 'designation',
		orderValue: 'asc'
	} as Pagination;
	$: queryString = fromPaginationToQuery(query);
	$: {
		month = type === ReportType.ANNUAL ? null : month;
	}

	// Form
	let formRef: HTMLFormElement;
	let isLoading = false;
	let fields: FieldDto[] = [];

	const reportTypes = [
		{ value: 'ORDINARY', text: 'Comum' },
		{ value: 'SEMESTER', text: 'Semenstral' },
		{ value: 'ANNUAL', text: 'Anual' }
	];

	const schema = yup.object().shape({
		title: yup.string().required(TEMPLATES.YUP.REQUIRED('Título')),
		text: yup.string().optional(),
		shortDescription: yup.string().required(TEMPLATES.YUP.REQUIRED('Descrição')),
		attachments: yup
			.array(yup.string())
			.min(1, TEMPLATES.YUP.MIN_NUMBER('Anexos', 1))
			.required(TEMPLATES.YUP.REQUIRED('Anexos')),
		month: yup.number().when('type', {
			is: (val: ReportType) => val !== ReportType.ANNUAL && val !== null,
			then: (schema) =>
				schema
					.integer()
					.min(1, TEMPLATES.YUP.MIN_NUMBER('Mês', 1))
					.max(12, TEMPLATES.YUP.MAX_NUMBER('Mês', 12))
					.required(TEMPLATES.YUP.REQUIRED('Mês')),
			otherwise: (schema) => schema.nullable().optional()
		}),
		year: yup
			.number()
			.integer()
			.min(2000, TEMPLATES.YUP.MIN_NUMBER('Ano', 2000))
			.max(2100, TEMPLATES.YUP.MAX_NUMBER('Ano', 2100))
			.required(TEMPLATES.YUP.REQUIRED('Ano')),
		type: yup
			.string()
			.oneOf(Object.values(ReportType), TEMPLATES.YUP.ONE_OF(reportTypes.map((r) => r.text)))
			.required(TEMPLATES.YUP.REQUIRED('Tipo')),
		field: yup.string().nullable().optional()
	});

	let title: string;
	let text: string;
	let shortDescription: string;
	let attachments: string[];
	let month: number | null;
	let year: number;
	let type: ReportType | null = null;
	let field: string | null;

	let filesToUpload: File[] | null;
	let messages: any[] = [];

	onMount(async () => {
		await loadData();
		isAdminOrVolunteer = userStore.isVolunteer() || userStore.isAdmin();
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
			const isValid = schema.validateSync(
				{ title, text, shortDescription, attachments, month, year, type, field },
				{ abortEarly: false }
			);

			if (isValid) {
				if (filesToUpload) {
					await uploadFiles(filesToUpload);
				}

				const postData = {
					title,
					text,
					shortDescription,
					attachments,
					month,
					year,
					type,
					field
				};
				if (isAdminOrVolunteer) {
					// @ts-ignore
					delete postData.field;
				}
				if (!month) {
					// @ts-ignore
					delete postData.month;
				}
				const res = await axios.post('/report', postData);

				isLoading = false;
				messages = generateMessages([{ message: res.data.message, variant: 'success' }]);
				eraseForm();
			}
		} catch (error) {
			isLoading = false;

			if (error instanceof Axios.AxiosError) {
				console.log(error.response);
				messages = generateMessages(
					(error.response?.data.data as string[]).map((err) => ({ message: err }))
				);
			} else if (error instanceof yup.ValidationError) {
				messages = generateMessages(error.inner.map((err) => ({ message: err.message })));
			} else {
				console.warn(error);
			}
		}
	}

	function eraseForm() {
		formRef.reset();
		field = null;
		type = null;
	}

	function onFileInputChange(event: Event) {
		const files = (event.target as any).files as File[];
		if (files.length > 0) {
			filesToUpload = files;
			attachments = ['yupbypass'];
		} else {
			filesToUpload = null;
			attachments = [];
		}
	}

	async function uploadFiles(files: File[]) {
		try {
			let formData = new FormData();
			for (let i = 0; i < files.length; i++) {
				formData.append('files', files[i]);
			}

			const res = await axios.post('/file/bulk', formData, {
				headers: {
					'Content-Type': 'multipart/form-data'
				}
			});

			messages = generateMessages([{ message: res.data.message, variant: 'success' }]);
			attachments = (res.data.data as FileDto[]).map((file) => file.name);
		} catch (error) {
			if (error instanceof Axios.AxiosError) {
				messages = generateMessages([{ message: error.response?.data.message }]);
			} else {
				console.warn(error);
			}
		}
	}
</script>

<svelte:head>
	<title>Report</title>
</svelte:head>

<AppContainer {messages} locale={data.locale}>
	<AppContent {...appHeader} {isLoading} on:click={onSubmit} {showActions} {showRefreshButton}>
		<form bind:this={formRef} on:submit|preventDefault|stopPropagation={onSubmit} class="app-form">
			<div class="input">
				<Icon src={HiOutlinePencil} />
				<input bind:value={title} name="title" placeholder="Título" autocomplete="off" required />
			</div>
			<div class="input">
				<Icon src={HiOutlineMenuAlt2} />
				<textarea
					bind:value={shortDescription}
					name="shortDescription"
					placeholder="Descrição"
					autocomplete="off"
					rows="5"
					required
				/>
			</div>
			<div class="input">
				<Icon src={HiOutlineMenuAlt2} />
				<textarea
					bind:value={text}
					name="text"
					placeholder="Texto"
					autocomplete="off"
					rows="5"
					required
				/>
			</div>
			<div class="input">
				<Icon src={HiOutlinePaperClip} />
				<input
					on:change={onFileInputChange}
					name="attachments"
					multiple
					type="file"
					accept="application/pdf"
				/>
			</div>
			<div class="input">
				<Icon src={HiOutlineCalendar} />
				<div class="number">
					<input
						bind:value={month}
						name="month"
						placeholder="Mês"
						type="number"
						min="1"
						max="12"
						disabled={type == ReportType.ANNUAL}
					/>
				</div>
				<div class="number">
					<input
						bind:value={year}
						name="year"
						placeholder="Ano"
						type="number"
						min="2000"
						max="2100"
						required
					/>
				</div>
			</div>
			<div class="input">
				<Icon src={HiOutlineTag} />
				<select bind:value={type} name="type" required>
					<option value={null} disabled selected>Tipo</option>

					{#each reportTypes as reportType}
						<option value={reportType.value}>{reportType.text}</option>
					{/each}
				</select>
			</div>
			{#if !isAdminOrVolunteer}
				<div class="input">
					<Icon src={HiOutlineGlobe} />
					<select bind:value={field} name="field" required>
						<option value={null} disabled selected>Campo Missionário</option>

						{#each fields as field}
							<option value={field.id}>
								{field.abbreviation} - {field.designation}
							</option>
						{/each}
					</select>
				</div>
			{/if}
		</form>
	</AppContent>
</AppContainer>