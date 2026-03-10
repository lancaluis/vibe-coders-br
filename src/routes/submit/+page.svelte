<script lang="ts">
	import { goto } from '$app/navigation';
	import { supabase } from '$lib/supabase';
	import { onMount, onDestroy } from 'svelte';
	import type { User } from '@supabase/supabase-js';

	// Estado de Autenticação
	let currentUser = $state<User | null>(null);
	let isLoadingSession = $state<boolean>(true);

	// Estado do Formulário
	let title = $state<string>('');
	let tagline = $state<string>('');
	let projectUrl = $state<string>('');

	// Manipulação de Arquivo (com tipagem estrita)
	let imageFile = $state<File | null>(null);
	let imagePreviewUrl = $state<string | null>(null);

	// Estado da Interface
	let isSubmitting = $state<boolean>(false);
	let errorMessage = $state<string | null>(null);

	onMount(async () => {
		const {
			data: { session }
		} = await supabase.auth.getSession();
		currentUser = session?.user ?? null;
		isLoadingSession = false;

		// Proteção de Rota: Se não estiver logado, chuta para a home
		if (!currentUser) {
			goto('/');
		}
	});

	// Prevenção de Memory Leak: Limpa a URL do preview quando o componente for destruído
	onDestroy(() => {
		if (imagePreviewUrl) URL.revokeObjectURL(imagePreviewUrl);
	});

	// Função tipada para o evento de input de arquivo
	function handleFileChange(event: Event) {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];

		if (file) {
			if (!file.type.startsWith('image/')) {
				errorMessage = 'Por favor, selecione apenas arquivos de imagem.';
				return;
			}
			errorMessage = null;
			imageFile = file;

			// Atualiza o preview de forma otimizada
			if (imagePreviewUrl) URL.revokeObjectURL(imagePreviewUrl);
			imagePreviewUrl = URL.createObjectURL(file);
		}
	}

	async function handleSubmit(event: Event) {
		event.preventDefault();

		if (!currentUser) return;
		if (!title || !tagline || !projectUrl || !imageFile) {
			errorMessage = 'Por favor, preencha todos os campos obrigatórios.';
			return;
		}

		isSubmitting = true;
		errorMessage = null;

		try {
			// 1. Upload da Imagem no Supabase Storage
			// Usamos o crypto.randomUUID() para evitar colisão de nomes de arquivos
			const fileExt = imageFile.name.split('.').pop();
			const filePath = `${currentUser.id}/${crypto.randomUUID()}.${fileExt}`;

			const { error: uploadError } = await supabase.storage
				.from('project-covers')
				.upload(filePath, imageFile, {
					cacheControl: '3600',
					upsert: false
				});

			if (uploadError) throw new Error(`Erro no upload da imagem: ${uploadError.message}`);

			// 2. Resgata a URL Pública da imagem recém-upada
			const {
				data: { publicUrl }
			} = supabase.storage.from('project-covers').getPublicUrl(filePath);

			// 3. Salva os dados no Banco de Dados Relacional
			const { error: dbError } = await supabase.from('projects').insert({
				user_id: currentUser.id,
				title: title,
				tagline: tagline,
				project_url: projectUrl,
				cover_image_url: publicUrl
			});

			if (dbError) throw new Error(`Erro ao salvar no banco: ${dbError.message}`);

			// Sucesso! Redireciona o usuário de volta para a Home
			goto('/');
		} catch (error) {
			if (error instanceof Error) {
				errorMessage = error.message;
			} else {
				errorMessage = 'Ocorreu um erro inesperado ao publicar o projeto.';
			}
		} finally {
			isSubmitting = false;
		}
	}
</script>

{#if isLoadingSession}
	<div class="flex min-h-screen items-center justify-center bg-zinc-950 text-zinc-400">
		<p>Autenticando...</p>
	</div>
{:else if currentUser}
	<main class="min-h-screen bg-zinc-950 px-4 py-12 text-zinc-50 sm:px-6 lg:px-8">
		<div class="mx-auto max-w-2xl space-y-8">
			<div>
				<h1 class="text-3xl font-bold tracking-tight text-white">Lançar Projeto</h1>
				<p class="mt-2 text-zinc-400">Mostre para a comunidade o que você construiu.</p>
			</div>

			<form
				onsubmit={handleSubmit}
				class="space-y-6 rounded-xl border border-zinc-800 bg-zinc-900 p-6 sm:p-8"
			>
				<div>
					<label for="title" class="block text-sm font-medium text-zinc-300">Nome do Projeto</label>
					<input
						type="text"
						id="title"
						bind:value={title}
						disabled={isSubmitting}
						placeholder="Ex: SupaBase Helper"
						class="mt-2 block w-full rounded-md border-zinc-700 bg-zinc-950 text-white shadow-sm focus:border-emerald-500 focus:ring-emerald-500 disabled:opacity-50 sm:text-sm"
						required
					/>
				</div>

				<div>
					<label for="tagline" class="block text-sm font-medium text-zinc-300"
						>Frase de Efeito (Tagline)</label
					>
					<input
						type="text"
						id="tagline"
						bind:value={tagline}
						disabled={isSubmitting}
						maxlength="60"
						placeholder="Ex: A forma mais rápida de gerenciar seu banco (Max 60 chars)"
						class="mt-2 block w-full rounded-md border-zinc-700 bg-zinc-950 text-white shadow-sm focus:border-emerald-500 focus:ring-emerald-500 disabled:opacity-50 sm:text-sm"
						required
					/>
				</div>

				<div>
					<label for="url" class="block text-sm font-medium text-zinc-300">Link do Projeto</label>
					<input
						type="url"
						id="url"
						bind:value={projectUrl}
						disabled={isSubmitting}
						placeholder="https://meuprojeto.com.br"
						class="mt-2 block w-full rounded-md border-zinc-700 bg-zinc-950 text-white shadow-sm focus:border-emerald-500 focus:ring-emerald-500 disabled:opacity-50 sm:text-sm"
						required
					/>
				</div>

				<div>
					<label for="cover" class="block text-sm font-medium text-zinc-300">Imagem de Capa</label>
					<input
						type="file"
						id="cover"
						accept="image/*"
						onchange={handleFileChange}
						disabled={isSubmitting}
						class="mt-2 block w-full cursor-pointer text-sm text-zinc-400 file:mr-4 file:rounded-md file:border-0 file:bg-zinc-800 file:px-4 file:py-2 file:text-sm file:font-semibold file:text-zinc-300 hover:file:bg-zinc-700 disabled:opacity-50"
						required
					/>

					{#if imagePreviewUrl}
						<div
							class="relative mt-4 aspect-video overflow-hidden rounded-lg border border-zinc-800"
						>
							<img src={imagePreviewUrl} alt="Preview da capa" class="h-full w-full object-cover" />
						</div>
					{/if}
				</div>

				{#if errorMessage}
					<div class="rounded-md border border-red-500/50 bg-red-900/50 p-4">
						<p class="text-sm text-red-400">{errorMessage}</p>
					</div>
				{/if}

				<button
					type="submit"
					disabled={isSubmitting}
					class="flex w-full justify-center rounded-md border border-transparent bg-emerald-500 px-4 py-3 text-sm font-medium text-zinc-950 shadow-sm transition-colors hover:bg-emerald-600 focus:ring-2 focus:ring-emerald-500 focus:ring-offset-2 focus:ring-offset-zinc-900 focus:outline-none disabled:cursor-not-allowed disabled:opacity-50"
				>
					{isSubmitting ? 'Publicando...' : 'Lançar Projeto'}
				</button>
			</form>
		</div>
	</main>
{/if}
