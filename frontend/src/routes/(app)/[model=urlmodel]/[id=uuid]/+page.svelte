<script lang="ts">
	import { page } from '$app/stores';
	import ConfirmModal from '$lib/components/Modals/ConfirmModal.svelte';
	import CreateModal from '$lib/components/Modals/CreateModal.svelte';
	import ModelTable from '$lib/components/ModelTable/ModelTable.svelte';
	import type {
		ModalComponent,
		ModalSettings,
		ModalStore,
		ToastStore
	} from '@skeletonlabs/skeleton';
	import { TabGroup, Tab, getModalStore, getToastStore } from '@skeletonlabs/skeleton';
	import { breadcrumbObject } from '$lib/utils/stores';
	import { superForm } from 'sveltekit-superforms/client';
	import { getModelInfo } from '$lib/utils/crud.js';
	import { URL_MODEL_MAP } from '$lib/utils/crud';
	import { isURL } from '$lib/utils/helpers';

	const modalStore: ModalStore = getModalStore();
	const toastStore: ToastStore = getToastStore();

	export let data;

	$: breadcrumbObject.set(data.data);

	let tabSet = 0;

	function handleFormUpdated({
		form,
		pageStatus,
		closeModal
	}: {
		form: any;
		pageStatus: number;
		closeModal: boolean;
	}) {
		if (closeModal && form.valid) {
			$modalStore[0] ? modalStore.close() : null;
		}
		if (form.message) {
			const toast: { message: string; background: string } = {
				message: form.message,
				background: pageStatus === 200 ? 'variant-filled-success' : 'variant-filled-error'
			};
			toastStore.trigger(toast);
		}
	}

	function modalCreateForm(model: Record<string, any>): void {
		const modalComponent: ModalComponent = {
			ref: CreateModal,
			props: {
				form: model.createForm,
				model: model,
				debug: false
			}
		};
		const modal: ModalSettings = {
			type: 'component',
			component: modalComponent,
			// Data
			title: `New ${model.info.verboseName.toLowerCase()}`
		};
		modalStore.trigger(modal);
	}

	function modalConfirm(id: string, name: string, action: string): void {
		const modalComponent: ModalComponent = {
			ref: ConfirmModal,
			props: {
				_form: data.data.form,
				id: id,
				debug: false,
				URLModel: getModelInfo('risk-acceptances').urlModel,
				formAction: action
			}
		};
		const modal: ModalSettings = {
			type: 'component',
			component: modalComponent,
			// Data
			title: 'Confirm',
			body: `Are you sure? This action will permanently affect the following object: ${name}?`
		};
		modalStore.trigger(modal);
	}

	function getForms(model: Record<string, any>) {
		let { form: createForm, message: createMessage } = superForm(model.createForm, {
			onUpdated: ({ form }) =>
				handleFormUpdated({ form, pageStatus: $page.status, closeModal: true })
		});
		let { form: deleteForm, message: deleteMessage } = superForm(model.deleteForm, {
			onUpdated: ({ form }) =>
				handleFormUpdated({ form, pageStatus: $page.status, closeModal: true })
		});
		return { createForm, createMessage, deleteForm, deleteMessage };
	}

	let forms: Record<string, any> = {};

	const user = $page.data.user;
	const canEditObject: boolean = Object.hasOwn(user.permissions, `change_${data.model.name}`);

	$: displayEditButton = function () {
		return (
			canEditObject &&
			!['Accepted', 'Rejected', 'Revoked'].includes(data.data.state) &&
			!data.data.urn
		);
	};
	$: Object.entries(data.relatedModels).forEach(([key, value]) => {
		forms[key] = getForms(value);
	});
</script>

<div class="flex flex-col space-y-2">
	{#if data.data.state === 'Submitted' && $page.data.user.id === data.data.approver.id}
		<div
			class="flex flex-row space-x-4 items-center bg-yellow-100 rounded-container-token shadow px-6 py-2 mb-2 justify-between"
		>
			<div class="text-yellow-900">
				This risk acceptance is awaiting processing. Remember to <strong>review</strong> it before accepting
				or rejecting it, you will not be able to go back.
			</div>
			<div class="flex space-x-2">
				<button
					on:click={(_) => {
						modalConfirm(data.data.id, data.data.name, '?/accept');
					}}
					on:keydown={(_) => modalConfirm(data.data.id, data.data.name, '?/accept')}
					class="btn variant-filled-success"
				>
					<i class="fas fa-check mr-2" /> Accept</button
				>
				<button
					on:click={(_) => {
						modalConfirm(data.data.id, data.data.name, '?/reject');
					}}
					on:keydown={(_) => modalConfirm(data.data.id, data.data.name, '?/reject')}
					class="btn variant-filled-error"
				>
					<i class="fas fa-xmark mr-2" /> Reject</button
				>
			</div>
		</div>
	{:else if data.data.state === 'Accepted'}
		<div
			class="flex flex-row items-center space-x-4 bg-green-100 rounded-container-token shadow-lg px-6 py-2 mt-2 justify-between"
		>
			<div class="text-green-900">
				This risk acceptance is currently accepted. It can be revoked at any time, but this will be <strong
					>irrevocable</strong
				>. You will need to duplicate it with a different verison if necessary.
			</div>
			{#if $page.data.user.id === data.data.approver.id}
				<div class="ml-auto whitespace-nowrap">
					<button
						on:click={(_) => {
							modalConfirm(data.data.id, data.data.name, '?/revoke');
						}}
						on:keydown={(_) => modalConfirm(data.data.id, data.data.name, '?/revoke')}
						class="btn variant-filled-error"
					>
						<i class="fas fa-xmark mr-2" /> Revoke</button
					>
				</div>
			{/if}
		</div>
	{/if}
	<div class="card px-6 py-4 bg-white flex flex-row space-y-2 justify-between shadow-lg">
		<div class="flex flex-col space-y-2">
			{#each Object.entries(data.data).filter(([key, _]) => !['id', 'is_published'].includes(key)) as [key, value]}
				<div class="flex flex-col">
					<div class="text-sm font-medium text-gray-800 capitalize-first">
						{key.replace('_', ' ')}
					</div>
					<ul class="text-sm">
						<li class="text-gray-600 list-none">
							{#if value}
								{#if Array.isArray(value)}
									<ul>
										{#each value as val}
											<li>
												{#if val.str && val.id}
													{@const itemHref = `/${
														URL_MODEL_MAP[data.urlModel]['foreignKeyFields']?.find(
															(item) => item.field === key
														)?.urlModel
													}/${val.id}`}
													<a href={itemHref} class="anchor">{val.str}</a>
												{:else}
													{value}
												{/if}
											</li>
										{/each}
									</ul>
								{:else if value.id}
									{@const itemHref = `/${
										URL_MODEL_MAP[data.urlModel]['foreignKeyFields']?.find(
											(item) => item.field === key
										)?.urlModel
									}/${value.id}`}
									<a href={itemHref} class="anchor">{value.str}</a>
								{:else if isURL(value)}
									<a href={value} target="_blank" class="anchor">{value}</a>
								{:else}
									{(value.str || value.name) ?? value}
								{/if}
							{:else}
								--
							{/if}
						</li>
					</ul>
				</div>
			{/each}
		</div>
		{#if displayEditButton()}
			<a
				href={`${$page.url.pathname}/edit?next=${$page.url.pathname}`}
				class="btn variant-filled-primary h-fit"
				><i class="fa-solid fa-pen-to-square mr-2" /> Edit</a
			>
		{/if}
	</div>
</div>

{#if Object.keys(data.relatedModels).length > 0}
	<div class="card shadow-lg mt-8 bg-white">
		<TabGroup justify="justify-center">
			{#each Object.entries(data.relatedModels) as [urlmodel, model], index}
				<Tab bind:group={tabSet} value={index} name={`${urlmodel}_tab`}>
					{model.info.verboseNamePlural}
					{#if model.table.body.length > 0}
						<span class="badge variant-soft-secondary">{model.table.body.length}</span>
					{/if}
				</Tab>
			{/each}
			<svelte:fragment slot="panel">
				{#each Object.entries(data.relatedModels) as [urlmodel, model], index}
					{#if tabSet === index}
						<div class="flex flex-row justify-between px-4 py-2">
							<h4 class="font-semibold lowercase capitalize-first my-auto">
								Associated {model.info.verboseNamePlural}
							</h4>
							<button
								class="btn variant-filled-primary self-end my-auto"
								on:click={(_) => modalCreateForm(model)}
								><i class="fa-solid fa-plus mr-2 lowercase" />New {model.info.verboseName}</button
							>
						</div>
						{#if model.table}
							<ModelTable source={model.table} deleteForm={model.deleteForm} URLModel={urlmodel} />
						{/if}
					{/if}
				{/each}
			</svelte:fragment>
		</TabGroup>
	</div>
{/if}