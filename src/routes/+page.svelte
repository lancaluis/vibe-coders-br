<script lang="ts">
	import { Dialog } from 'bits-ui';
	import { supabase } from '$lib/supabase';
	import { onMount } from 'svelte';
	import type { User } from '@supabase/supabase-js';

	// Estado (Runes do Svelte 5)
	let user = $state<User | null>(null);
	let projects = $state<any[]>([]);
	let isLoadingProjects = $state(true);

	onMount(() => {
		// Verifica a sessão inicial
		supabase.auth.getSession().then(({ data: { session } }) => {
			user = session?.user ?? null;
		});

		// Escuta mudanças (login/logout)
		const {
			data: { subscription }
		} = supabase.auth.onAuthStateChange((_event, session) => {
			user = session?.user ?? null;
		});

		// Busca a lista de projetos e seus votos
		fetchProjects();

		return () => subscription.unsubscribe();
	});

	async function fetchProjects() {
		isLoadingProjects = true;
		const { data, error } = await supabase
			.from('projects')
			.select(
				`
				*,
				profiles!projects_user_id_fkey ( username, avatar_url ),
				votes ( user_id )
			`
			)
			.order('created_at', { ascending: false }); // Desempate: mais recentes primeiro

		if (error) {
			console.error('Erro ao buscar projetos:', error);
		} else {
			// Ordenação Javascript (Client-side)
			// Coloca quem tem o array de 'votes' maior no topo da lista
			const sortedProjects = (data ?? []).sort((a, b) => {
				const votesA = a.votes?.length || 0;
				const votesB = b.votes?.length || 0;
				return votesB - votesA;
			});

			projects = sortedProjects;
		}
		isLoadingProjects = false;
	}

	async function toggleVote(project: any) {
		if (!user) {
			alert('Por favor, faça login com o GitHub para votar!');
			return;
		}

		// Verifica se o usuário atual já votou
		const hasVoted = project.votes?.some((v: any) => v.user_id === user?.id);

		if (hasVoted) {
			// UI Otimista: Remove o voto da tela na hora
			project.votes = project.votes.filter((v: any) => v.user_id !== user?.id);

			// Deleta do banco (Background)
			const { error } = await supabase
				.from('votes')
				.delete()
				.match({ project_id: project.id, user_id: user.id });

			if (error) console.error('Erro ao remover voto:', error);
		} else {
			// UI Otimista: Adiciona o voto na tela na hora
			project.votes = [...(project.votes || []), { user_id: user.id }];

			// Insere no banco (Background)
			const { error } = await supabase
				.from('votes')
				.insert({ project_id: project.id, user_id: user.id });

			if (error) console.error('Erro ao registrar voto:', error);
		}
	}

	async function loginWithGithub() {
		await supabase.auth.signInWithOAuth({
			provider: 'github',
			options: { redirectTo: window.location.origin }
		});
	}

	async function logout() {
		await supabase.auth.signOut();
	}
</script>

<main class="min-h-screen bg-zinc-950 pb-20 text-zinc-50">
	<header class="space-y-6 px-4 pt-20 pb-16 text-center">
		<h1 class="text-5xl font-extrabold tracking-tight md:text-6xl">
			Vibe <span class="text-emerald-400">Coders</span> BR
		</h1>
		<p class="mx-auto max-w-2xl text-lg text-zinc-400 md:text-xl">
			Descubra, vote e compartilhe os projetos indie mais incríveis da comunidade brasileira.
		</p>

		<div class="flex items-center justify-center gap-4 pt-4">
			{#if user}
				<div
					class="flex items-center gap-4 rounded-full border border-zinc-800 bg-zinc-900 py-1.5 pr-4 pl-2"
				>
					<img
						src={user.user_metadata.avatar_url}
						alt="Avatar"
						class="h-8 w-8 rounded-full border border-zinc-700"
					/>
					<span class="text-sm font-medium text-zinc-300">{user.user_metadata.user_name}</span>
					<button
						onclick={logout}
						class="ml-2 text-xs font-bold text-red-400 transition-colors hover:text-red-300"
						>Sair</button
					>
				</div>
				<a
					href="/submit"
					class="rounded-full bg-emerald-500 px-5 py-2.5 text-sm font-bold text-zinc-950 shadow-[0_0_15px_rgba(16,185,129,0.3)] transition-colors hover:bg-emerald-600"
				>
					+ Lançar Projeto
				</a>
			{:else}
				<Dialog.Root>
					<Dialog.Trigger
						class="rounded-full bg-emerald-500 px-6 py-3 font-bold text-zinc-950 transition-colors hover:bg-emerald-600"
					>
						Entrar com GitHub
					</Dialog.Trigger>
					<Dialog.Portal>
						<Dialog.Overlay class="fixed inset-0 z-40 bg-black/80 backdrop-blur-sm" />
						<Dialog.Content
							class="fixed top-[50%] left-[50%] z-50 w-full max-w-sm translate-x-[-50%] translate-y-[-50%] rounded-2xl border border-zinc-800 bg-zinc-900 p-6 shadow-2xl"
						>
							<Dialog.Title class="mb-2 text-2xl font-bold text-white">Acessar conta</Dialog.Title>
							<Dialog.Description class="mb-6 text-zinc-400"
								>Faça login com seu GitHub para postar e votar.</Dialog.Description
							>
							<button
								onclick={loginWithGithub}
								class="flex w-full items-center justify-center gap-2 rounded-lg bg-white px-4 py-3 font-bold text-black transition-colors hover:bg-zinc-200"
							>
								<svg viewBox="0 0 24 24" class="h-5 w-5" fill="currentColor"
									><path
										d="M12 0c-6.626 0-12 5.373-12 12 0 5.302 3.438 9.8 8.207 11.387.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23.957-.266 1.983-.399 3.003-.404 1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.235 3.221 0 4.609-2.807 5.624-5.479 5.921.43.372.823 1.102.823 2.222v3.293c0 .319.192.694.801.576 4.765-1.589 8.199-6.086 8.199-11.386 0-6.627-5.373-12-12-12z"
									/></svg
								>
								Entrar com GitHub
							</button>
							<Dialog.Close class="absolute top-4 right-4 text-zinc-500 hover:text-white"
								>✕</Dialog.Close
							>
						</Dialog.Content>
					</Dialog.Portal>
				</Dialog.Root>
			{/if}
		</div>
	</header>

	<section class="mx-auto max-w-5xl px-4">
		{#if isLoadingProjects}
			<div class="flex justify-center py-12 text-zinc-500">
				<p>Carregando projetos incríveis...</p>
			</div>
		{:else if projects.length === 0}
			<div
				class="rounded-2xl border border-dashed border-zinc-800 bg-zinc-900/30 py-20 text-center"
			>
				<p class="mb-4 text-zinc-400">Ainda não há projetos por aqui.</p>
				{#if user}
					<a href="/submit" class="text-emerald-400 hover:underline">Seja o primeiro a lançar!</a>
				{/if}
			</div>
		{:else}
			<div class="grid grid-cols-1 gap-6 md:grid-cols-2 lg:grid-cols-3">
				{#each projects as project}
					<div
						class="group overflow-hidden rounded-2xl border border-zinc-800 bg-zinc-900 transition-all duration-300 hover:border-zinc-700 hover:shadow-[0_8px_30px_rgb(0,0,0,0.5)]"
					>
						<a
							href={project.project_url}
							target="_blank"
							rel="noopener noreferrer"
							class="relative block aspect-video overflow-hidden"
						>
							<img
								src={project.cover_image_url}
								alt={project.title}
								class="h-full w-full object-cover transition-transform duration-500 group-hover:scale-105"
							/>
						</a>

						<div class="space-y-4 p-5">
							<div>
								<h3 class="mb-1 line-clamp-1 text-xl font-bold text-white">{project.title}</h3>
								<p class="line-clamp-2 text-sm text-zinc-400">{project.tagline}</p>
							</div>

							<div class="flex items-center justify-between border-t border-zinc-800/50 pt-2">
								<div class="flex items-center gap-2">
									<img
										src={project.profiles.avatar_url}
										alt="Autor"
										class="h-6 w-6 rounded-full bg-zinc-800"
									/>
									<span class="max-w-[100px] truncate text-xs text-zinc-400"
										>{project.profiles.username}</span
									>
								</div>

								<button
									onclick={() => toggleVote(project)}
									class="group/btn flex items-center gap-1.5 rounded-lg border px-3 py-1.5 transition-colors
									{project.votes?.some((v: any) => user && v.user_id === user.id)
										? 'border-emerald-500/50 bg-emerald-500/10 text-emerald-400'
										: 'border-transparent bg-zinc-800 text-zinc-300 hover:bg-zinc-700 hover:text-emerald-400'}"
								>
									<svg
										viewBox="0 0 24 24"
										class="h-4 w-4 transition-transform group-hover/btn:-translate-y-0.5"
										fill="currentColor"
									>
										<path d="M12 4l8 10h-16l8-10z" />
									</svg>
									<span class="text-sm font-bold">
										{project.votes?.length || 0}
									</span>
								</button>
							</div>
						</div>
					</div>
				{/each}
			</div>
		{/if}
	</section>
</main>
