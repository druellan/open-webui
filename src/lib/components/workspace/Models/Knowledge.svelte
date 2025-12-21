<script lang="ts">
	import { getContext, onMount } from 'svelte';
	import { config, knowledge, settings, user } from '$lib/stores';

	import Selector from './Knowledge/Selector.svelte';
	import FileItem from '$lib/components/common/FileItem.svelte';

	import { getKnowledgeBases } from '$lib/apis/knowledge';
	import { uploadFile } from '$lib/apis/files';
	import { createPicker, getAuthToken } from '$lib/utils/google-drive-picker';

	import { toast } from 'svelte-sonner';
	import { v4 as uuidv4 } from 'uuid';
	import { WEBUI_API_BASE_URL } from '$lib/constants';

	export let selectedItems = [];
	const i18n = getContext('i18n');

	let loaded = false;

	let filesInputElement = null;
	let inputFiles = null;

	const uploadFileHandler = async (file, fullContext: boolean = false) => {
		if ($user?.role !== 'admin' && !($user?.permissions?.chat?.file_upload ?? true)) {
			toast.error($i18n.t('You do not have permission to upload files.'));
			return null;
		}

		const tempItemId = uuidv4();
		const fileItem = {
			type: 'file',
			file: '',
			id: null,
			url: '',
			name: file.name,
			collection_name: '',
			status: 'uploading',
			size: file.size,
			error: '',
			itemId: tempItemId,
			...(fullContext ? { context: 'full' } : {})
		};

		if (fileItem.size == 0) {
			toast.error($i18n.t('You cannot upload an empty file.'));
			return null;
		}

		selectedItems = [...selectedItems, fileItem];

		try {
			// If the file is an audio file, provide the language for STT.
			let metadata = null;
			if (
				(file.type.startsWith('audio/') || file.type.startsWith('video/')) &&
				$settings?.audio?.stt?.language
			) {
				metadata = {
					language: $settings?.audio?.stt?.language
				};
			}

			// During the file upload, file content is automatically extracted.
			const uploadedFile = await uploadFile(localStorage.token, file, metadata);

			if (uploadedFile) {
				console.log('File upload completed:', {
					id: uploadedFile.id,
					name: fileItem.name,
					collection: uploadedFile?.meta?.collection_name
				});

				if (uploadedFile.error) {
					console.warn('File upload warning:', uploadedFile.error);
					toast.warning(uploadedFile.error);
				}

				fileItem.status = 'uploaded';
				fileItem.file = uploadedFile;
				fileItem.id = uploadedFile.id;
				fileItem.collection_name =
					uploadedFile?.meta?.collection_name || uploadedFile?.collection_name;
				fileItem.url = `${WEBUI_API_BASE_URL}/files/${uploadedFile.id}`;

				selectedItems = selectedItems;
			} else {
				selectedItems = selectedItems.filter((item) => item?.itemId !== tempItemId);
			}
		} catch (e) {
			toast.error(`${e}`);
			selectedItems = selectedItems.filter((item) => item?.itemId !== tempItemId);
		}
	};

	const inputFilesHandler = async (inputFiles) => {
		console.log('Input files handler called with:', inputFiles);

		inputFiles.forEach(async (file) => {
			console.log('Processing file:', {
				name: file.name,
				type: file.type,
				size: file.size,
				extension: file.name.split('.').at(-1)
			});

			if (
				($config?.file?.max_size ?? null) !== null &&
				file.size > ($config?.file?.max_size ?? 0) * 1024 * 1024
			) {
				console.log('File exceeds max size limit:', {
					fileSize: file.size,
					maxSize: ($config?.file?.max_size ?? 0) * 1024 * 1024
				});
				toast.error(
					$i18n.t(`File size should not exceed {{maxSize}} MB.`, {
						maxSize: $config?.file?.max_size
					})
				);
				return;
			}

			if (!file['type'].startsWith('image/')) {
				uploadFileHandler(file);
			} else {
				toast.error($i18n.t(`Unsupported file type.`));
			}
		});
	};

	const uploadGoogleDriveHandler = async () => {
		try {
			try {
				const fileData = await createPicker();
				if (fileData) {
					const file = new File([fileData.blob], fileData.name, {
						type: fileData.blob.type
					});
					await uploadFileHandler(file);
				} else {
					console.log('No file was selected from Google Drive');
				}
			} catch (error) {
				console.error('Google Drive Error:', error);
				toast.error(
					$i18n.t('Error accessing Google Drive: {{error}}', {
						error: error.message
					})
				);
			}
		} catch (error) {
			console.error('Google Drive initialization error:', error);
			toast.error($i18n.t('Failed to initialize Google Drive'));
		}
	};

	onMount(async () => {
		if (!$knowledge) {
			knowledge.set(await getKnowledgeBases(localStorage.token));
		}
		loaded = true;
	});
</script>

<input
	bind:this={filesInputElement}
	bind:files={inputFiles}
	type="file"
	hidden
	multiple
	on:change={async () => {
		if (inputFiles && inputFiles.length > 0) {
			const _inputFiles = Array.from(inputFiles);
			inputFilesHandler(_inputFiles);
		} else {
			toast.error($i18n.t(`File not found.`));
		}

		filesInputElement.value = '';
	}}
/>

<div>
	<slot name="label">
		<div class="mb-2">
			<div class="flex w-full justify-between mb-1">
				<div class=" self-center text-xs font-medium text-gray-500">
					{$i18n.t('Knowledge')}
				</div>
			</div>
		</div>
	</slot>

	<div class="flex flex-col mb-1">
		{#if selectedItems?.length > 0}
			<div class=" flex flex-wrap items-center gap-2 mb-2.5">
				{#each selectedItems as file, fileIdx}
					<FileItem
						{file}
						small={true}
						item={file}
						name={file.name}
						modal={true}
						edit={true}
						loading={file.status === 'uploading'}
						type={file?.legacy
							? `Legacy${file.type ? ` ${file.type}` : ''}`
							: (file?.type ?? 'collection')}
						dismissible
						on:dismiss={(e) => {
							selectedItems = selectedItems.filter((_, idx) => idx !== fileIdx);
						}}
					/>
				{/each}
			</div>
		{/if}

		{#if loaded}
			<div class="flex flex-wrap flex-row text-sm gap-1">
				<Selector
					knowledgeItems={$knowledge || []}
					on:select={(e) => {
						const item = e.detail;

						if (!selectedItems.find((k) => k.id === item.id)) {
							selectedItems = [
								...selectedItems,
								{
									...item
								}
							];
						}
					}}
				>
					<div
						class=" px-3.5 py-1.5 font-medium hover:bg-black/5 dark:hover:bg-white/5 outline outline-1 outline-gray-100 dark:outline-gray-850 rounded-3xl"
					>
						{$i18n.t('Select Knowledge')}
					</div>
				</Selector>

				{#if $user?.role === 'admin' || $user?.permissions?.chat?.file_upload}
					<button
						class=" px-3.5 py-1.5 font-medium hover:bg-black/5 dark:hover:bg-white/5 outline outline-1 outline-gray-100 dark:outline-gray-850 rounded-3xl"
						type="button"
						on:click={() => {
							filesInputElement.click();
						}}>{$i18n.t('Upload Files')}</button
					>
				{/if}

				{#if $config?.features?.enable_google_drive_integration && ($user?.role === 'admin' || $user?.permissions?.chat?.file_upload)}
					<button
						class=" px-3.5 py-1.5 font-medium hover:bg-black/5 dark:hover:bg-white/5 outline outline-1 outline-gray-100 dark:outline-gray-850 rounded-3xl flex items-center gap-2"
						type="button"
						on:click={uploadGoogleDriveHandler}
					>
						<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 87.3 78" class="w-4">
							<path
								d="m6.6 66.85 3.85 6.65c.8 1.4 1.95 2.5 3.3 3.3l13.75-23.8h-27.5c0 1.55.4 3.1 1.2 4.5z"
								fill="#0066da"
							/>
							<path
								d="m43.65 25-13.75-23.8c-1.35.8-2.5 1.9-3.3 3.3l-25.4 44a9.06 9.06 0 0 0 -1.2 4.5h27.5z"
								fill="#00ac47"
							/>
							<path
								d="m73.55 76.8c1.35-.8 2.5-1.9 3.3-3.3l1.6-2.75 7.65-13.25c.8-1.4 1.2-2.95 1.2-4.5h-27.502l5.852 11.5z"
								fill="#ea4335"
							/>
							<path
								d="m43.65 25 13.75-23.8c-1.35-.8-2.9-1.2-4.5-1.2h-18.5c-1.6 0-3.15.45-4.5 1.2z"
								fill="#00832d"
							/>
							<path
								d="m59.8 53h-32.3l-13.75 23.8c1.35.8 2.9 1.2 4.5 1.2h50.8c1.6 0 3.15-.45 4.5-1.2z"
								fill="#2684fc"
							/>
							<path
								d="m73.4 26.5-12.7-22c-.8-1.4-1.95-2.5-3.3-3.3l-13.75 23.8 16.15 28h27.45c0-1.55-.4-3.1-1.2-4.5z"
								fill="#ffba00"
							/>
						</svg>
						<span>{$i18n.t('Google Drive')}</span>
					</button>
				{/if}
			</div>
		{/if}
		<!-- {knowledge} -->
	</div>

	<div class=" text-xs dark:text-gray-700">
		{$i18n.t('To attach knowledge base here, add them to the "Knowledge" workspace first.')}
	</div>
</div>
