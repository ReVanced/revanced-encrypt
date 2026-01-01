<script lang="ts">
	import { GPG_KEY_URL } from '$env/static/public';
	import { encrypt, readKey, createMessage, type Key } from 'openpgp';
	import { onMount } from 'svelte';
	import logo from '$lib/assets/logo.svg';

	let fileInput: HTMLInputElement;
	let key: Key | null = null;
	let armoredKey = '';
	let keyUrl = GPG_KEY_URL;
	let keyError = '';
	let disabled = true;
	let inputText = '';
	let copyButtonText = 'Encrypt';

	onMount(() => {
		getKeyFromUrl();
	});

	async function getKeyFromUrl() {
		if (!keyUrl.trim()) return;

		keyError = '';
		try {
			const response = await fetch(keyUrl);
			if (!response.ok) {
				armoredKey = '';
				throw new Error(`HTTP ${response.status}`);
			}
			armoredKey = await response.text();
			await updateKey();
		} catch (e) {
			keyError = 'Failed to get key: ' + (e instanceof Error ? e.message : String(e));
		}
	}

	async function updateKey() {
		if (!armoredKey.trim()) {
			key = null;
			keyError = '';
			disabled = true;
			return;
		}
		try {
			key = await readKey({ armoredKey });
			keyError = '';
		} catch (e) {
			key = null;
			keyError = 'Invalid GPG key: ' + (e instanceof Error ? e.message : String(e));
		}

		keyUrl = '';

		disabled = !key || !!keyError;
	}

	async function handleFiles(files: FileList) {
		if (disabled || !key) return;

		for (const file of files) {
			const msg = await createMessage({ binary: new Uint8Array(await file.arrayBuffer()) });
			const enc = await encrypt({ message: msg, encryptionKeys: key, format: 'binary' });

			const a = document.createElement('a');
			a.href = URL.createObjectURL(new Blob([enc]));
			a.download = file.name + '.asc';
			a.click();
		}
	}

	async function encryptText() {
		if (disabled || !key || !inputText.trim()) return;

		const msg = await createMessage({ text: inputText });
		const enc = await encrypt({ message: msg, encryptionKeys: key, format: 'armored' });
		const encryptedText = enc as string;

		try {
			await navigator.clipboard.writeText(encryptedText);
			copyButtonText = 'Copied!';
			setTimeout(() => {
				copyButtonText = 'Encrypt';
			}, 2000);
		} catch (e) {
			copyButtonText = 'Failed';
			setTimeout(() => {
				copyButtonText = 'Encrypt';
			}, 2000);
		}
	}

	function handleDrop(event: DragEvent) {
		event.preventDefault();
		if (disabled) return;
		const files = event.dataTransfer?.files;
		if (files) handleFiles(files);
	}

	function handleChange(event: Event) {
		if (disabled) return;
		const target = event.target as HTMLInputElement;
		const files = target.files;
		if (files) handleFiles(files);
	}

	function triggerSelect() {
		if (disabled) return;
		fileInput?.click();
	}

	function handleTouchEnd(event: TouchEvent) {
		event.preventDefault();
		triggerSelect();
	}
</script>

<div class="container">
	<header>
		<img src={logo} alt="ReVanced Encrypt Logo" width="100" />
		<h1>ReVanced Encrypt</h1>
	</header>
	<!-- svelte-ignore a11y_click_events_have_key_events -->
	<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
	<main
		class:disabled
		ondragover={(e) => e.preventDefault()}
		ondrop={handleDrop}
		onclick={triggerSelect}
		ontouchend={handleTouchEnd}
	>
		<p>Drag files here or click to select</p>
		<input bind:this={fileInput} type="file" hidden multiple onchange={handleChange} {disabled} />
	</main>

	<div class="text-encrypt-section">
		<textarea bind:value={inputText} placeholder="Enter text to encrypt..." {disabled}></textarea>
		<button onclick={encryptText} disabled={disabled || !inputText.trim()}>{copyButtonText}</button>
	</div>

	<div class="key-section">
		{#if keyError}
			<label for="gpg-key" class="error">{keyError}</label>
		{:else if key}
			<label for="gpg-key" class="success">Key loaded: {key.getUserIDs().join(', ')}</label>
		{/if}
		<div class="key-url-row">
			<input type="url" bind:value={keyUrl} placeholder="Get from URL..." oninput={getKeyFromUrl} />
		</div>
		<textarea
			id="gpg-key"
			bind:value={armoredKey}
			oninput={updateKey}
			placeholder="Paste your GPG key here..."
		></textarea>
	</div>
</div>

<style>
	:global(html),
	:global(body) {
		margin: 0;
		padding: 0;
		height: 100%;
		background-color: rgb(26, 25, 31);
	}

	:global(body) {
		display: flex;
		justify-content: center;
		align-items: center;
		padding: 1rem;
		box-sizing: border-box;
	}

	.container {
		display: flex;
		flex-direction: column;
		gap: 1.5rem;
		font-family: system-ui, sans-serif;
		width: 100%;
		max-width: 600px;
	}

	header {
		text-align: center;
		color: rgb(172, 193, 210);
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 1rem;
	}

	main {
		border-radius: 1rem;
		display: grid;
		place-items: center;
		color: rgb(172, 193, 210);
		border: 2px dashed rgb(172, 193, 210);
		background-color: rgb(30, 31, 36);
		cursor: pointer;
		min-height: 250px;
		padding: 1rem;
		box-sizing: border-box;
		-webkit-tap-highlight-color: transparent;

		&:hover {
			background-color: rgb(40, 41, 46);
		}

		&:active {
			background-color: rgb(50, 51, 56);
		}

		&.disabled {
			opacity: 0.5;
			cursor: not-allowed;
			border-color: rgb(120, 120, 120);
		}
	}

	main p {
		margin: 0;
		text-align: center;
	}

	.text-encrypt-section {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}

	.text-encrypt-section textarea {
		width: 100%;
		height: 100px;
		padding: 0.75rem;
		border-radius: 0.5rem;
		border: 1px solid rgb(172, 193, 210);
		background-color: rgb(30, 31, 36);
		color: rgb(172, 193, 210);
		font-family: monospace;
		font-size: 0.875rem;
		resize: vertical;
		box-sizing: border-box;

		&:focus {
			outline: none;
			border-color: rgb(100, 150, 200);
		}

		&:disabled {
			opacity: 0.5;
			cursor: not-allowed;
		}
	}

	.text-encrypt-section button {
		padding: 0.5rem 1rem;
		border-radius: 0.5rem;
		border: 1px solid rgb(172, 193, 210);
		background-color: rgb(50, 51, 56);
		color: rgb(172, 193, 210);
		cursor: pointer;
		font-size: 0.875rem;
		white-space: nowrap;
		align-self: flex-start;

		&:hover:not(:disabled) {
			background-color: rgb(60, 61, 66);
		}

		&:disabled {
			opacity: 0.5;
			cursor: not-allowed;
		}
	}

	.key-section {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}

	.key-section label {
		color: rgb(172, 193, 210);
		font-size: 0.875rem;
		word-break: break-word;
	}

	.key-url-row {
		display: flex;
		gap: 0.5rem;
	}

	.key-url-row input {
		flex: 1;
		padding: 0.5rem 0.75rem;
		border-radius: 0.5rem;
		border: 1px solid rgb(172, 193, 210);
		background-color: rgb(30, 31, 36);
		color: rgb(172, 193, 210);
		font-size: 0.875rem;
		box-sizing: border-box;

		&:focus {
			outline: none;
			border-color: rgb(100, 150, 200);
		}
	}

	.key-section textarea {
		width: 100%;
		height: 220px;
		padding: 0.75rem;
		border-radius: 0.5rem;
		border: 1px solid rgb(172, 193, 210);
		background-color: rgb(30, 31, 36);
		color: rgb(172, 193, 210);
		font-family: monospace;
		font-size: 0.875rem;
		resize: vertical;
		box-sizing: border-box;

		&:focus {
			outline: none;
			border-color: rgb(100, 150, 200);
		}
	}

	.key-section .error {
		color: rgb(255, 100, 100);
		font-size: 0.875rem;
		margin: 0;
	}

	.key-section .success {
		color: rgb(100, 200, 100);
		font-size: 0.875rem;
		margin: 0;
	}

	@media (max-width: 480px) {
		.container {
			gap: 1rem;
		}

		main {
			min-height: 220px;
		}

		.key-section textarea {
			height: 200px;
			font-size: 0.7rem;
		}
	}
</style>
