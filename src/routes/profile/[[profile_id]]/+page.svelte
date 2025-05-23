<script lang="ts">
	import { enhance } from '$app/forms';
	import TreeMap from '$lib/components/TreeMap.svelte';
	import type { ShareFn } from '$lib/share';
	import { onMount } from 'svelte';
	import { flip } from 'svelte/animate';
	import { queryParam, ssp } from 'sveltekit-search-params';
	import Pending from '~icons/eos-icons/loading';
	import TrashCan from '~icons/material-symbols/delete-forever-outline';
	import Fork from '~icons/material-symbols/fork-right-rounded';
	import Share from '~icons/material-symbols/share';
	import type { PageData } from './$types';
	import ProfileHeader from './ProfileHeader.svelte';
	import NoneFound from '~icons/material-symbols/sad-tab-outline-rounded';
	import RelativeTime from '@yaireo/relative-time';
	import DropdownMenu from '$lib/components/DropdownMenu.svelte';
	import MenuItem from '$lib/components/MenuItem.svelte';
	import ArrowUpward from '~icons/material-symbols/arrow-upward';
	import ArrowDownward from '~icons/material-symbols/arrow-downward';

	export let data: PageData;

	type SortByKey = 'created' | 'updated' | 'name';
	type SortOrder = 'asc' | 'desc';
	let share: ShareFn;

	onMount(async () => {
		share = (await import('$lib/share')).share;
	});

	const search = queryParam('s', ssp.string(), {
		pushHistory: false,
	});

	let loading = [] as string[];
	let sort_by: SortByKey = 'updated';
	let sort_order: SortOrder = 'desc';

	const sort_functions: Record<
		SortByKey,
		(sort_order: 'asc' | 'desc') => (a: Record<string, string>, b: Record<string, string>) => number
	> = {
		created: (sort_order) => (a, b) =>
			sort_order === 'asc'
				? new Date(a.created).getTime() - new Date(b.created).getTime()
				: new Date(b.created).getTime() - new Date(a.created).getTime(),

		updated: (sort_order) => (a, b) =>
			sort_order === 'asc'
				? new Date(a.updated).getTime() - new Date(b.updated).getTime()
				: new Date(b.updated).getTime() - new Date(a.updated).getTime(),

		name: (sort_order) => (a, b) =>
			sort_order === 'asc' ? a.name.localeCompare(b.name) : b.name.localeCompare(a.name),
	};

	$: repls = data.repls
		.filter((repl) => repl.name.toLowerCase().includes($search?.toLowerCase() ?? ''))
		.sort(sort_functions[sort_by](sort_order));

	const relative_time = new RelativeTime();
	const sort_options: SortByKey[] = ['created', 'updated', 'name'];
</script>

<svelte:head>
	<title>{data.user?.username || 'Profile'} - SvelteLab</title>
</svelte:head>
<ProfileHeader />
<main>
	<div id="top-bar">
		<input
			bind:value={$search}
			placeholder="🔍 Search..."
			aria-label="Search for a repl"
			type="search"
		/>
		<div class="sort-wrapper">
			<button
				class="sort-order-button"
				on:click={() => (sort_order = sort_order === 'asc' ? 'desc' : 'asc')}
			>
				{#if sort_order === 'asc'}
					<ArrowUpward />
				{:else}
					<ArrowDownward />
				{/if}
			</button>

			<DropdownMenu indicator>
				<div slot="trigger" class="dropdown-trigger">
					Sort by {sort_by}
				</div>

				{#each sort_options as option (option)}
					{#if sort_by !== option}
						<MenuItem on:click={() => (sort_by = option)}>{option}</MenuItem>
					{/if}
				{/each}
			</DropdownMenu>
		</div>
	</div>
	{#each repls as project (project.id)}
		{@const created = relative_time.from(new Date(project.created))}
		{@const updated = relative_time.from(new Date(project.updated))}
		<!-- this will be useful when we will add delete -->
		<article
			animate:flip={{
				duration: 250,
			}}
		>
			<div>
				<a data-sveltekit-preload-data="off" href="/{project.id}">
					<p>
						{project.name}
					</p>
					<small title={project.created}>created {created}</small>
					{#if created != updated}
						<small title={project.updated}>updated {updated}</small>
					{/if}
					<code>/{project.id}</code>
				</a>
				<div class="buttons">
					<button
						on:click={() => {
							share?.({
								text: `Take a look at my REPL`,
								title: `SvelteLab - ${project.name}`,
								url: `/${project.id}`,
							});
						}}
					>
						<Share />
					</button>
					{#if data.user}
						<form
							use:enhance={() => {
								loading.push(project.id);
								loading = loading;
								return ({ update }) => {
									loading = loading.filter((id) => id !== project.id);
									update();
								};
							}}
							action="?/fork"
							method="POST"
						>
							<input name="id" type="hidden" value={project.id} />
							<button
								on:click={(e) => {
									if (!window.confirm(`Are you sure you want to fork "${project.name}"`)) {
										e.stopPropagation();
										e.preventDefault();
									}
								}}
							>
								<Fork />
							</button>
						</form>
					{/if}
					{#if data.user?.id === data.profile.id}
						<form
							use:enhance={() => {
								loading.push(project.id);
								loading = loading;
								return ({ update }) => {
									loading = loading.filter((id) => id !== project.id);
									// optimistic update
									repls = repls.filter((repl) => repl.id !== project.id);
									update();
								};
							}}
							action="?/delete"
							method="POST"
						>
							<input name="id" type="hidden" value={project.id} />
							<button
								on:click={(e) => {
									if (!window.confirm(`Are you sure you want to delete "${project.name}"`)) {
										e.stopPropagation();
										e.preventDefault();
									}
								}}
								style:color="var(--sk-theme-1)"
							>
								<TrashCan />
							</button>
						</form>
					{/if}
				</div>
			</div>
			<a data-sveltekit-preload-data="off" href="/{project.id}" class="tree">
				<TreeMap
					tree={{
						dir: {
							directory: {
								route: {
									directory: {
										'1': {
											file: {
												contents: '',
											},
										},
										'2': {
											file: {
												contents: '',
											},
										},
									},
								},
								'1': {
									file: {
										contents: '',
									},
								},
							},
						},
						'1': {
							file: {
								contents: '',
							},
						},
						'2': {
							file: {
								contents: '',
							},
						},
						'3': {
							file: {
								contents: '',
							},
						},
					}}
				/>
			</a>
			{#if loading.includes(project.id)}
				<div class="loading">
					<Pending />
				</div>
			{/if}
		</article>
	{:else}
		<div class="loader">
			<NoneFound />
			<span>No projects found</span>
		</div>
	{/each}
</main>

<style>
	:global(body) {
		background-color: var(--sk-back-1);
	}
	main {
		padding: 2rem;
		display: grid;
		grid-template-columns: repeat(auto-fill, minmax(30rem, 1fr));
		gap: 2rem;
		max-width: 100%;
		background-color: var(--sk-back-1);
	}
	#top-bar {
		display: flex;
		justify-content: center;
		gap: 1rem;
		grid-column: 1/-1;
		flex-wrap: wrap;
	}
	.sort-wrapper {
		display: flex;
		gap: 0.5rem;
	}
	.dropdown-trigger {
		display: flex;
	}
	input {
		grid-column: 1/-1;
		width: min(50rem, 100%);
		justify-self: center;
		background-color: transparent;
		border: 1px solid var(--sk-back-4);
		padding: 0.5rem;
		border-radius: 0.5rem;
		color: var(--sk-text-1);
		font-size: 2rem;
	}
	input::placeholder {
		color: var(--sk-text-1);
		opacity: 0.6;
	}
	a {
		color: var(--sk-text-1);
		text-decoration: none;
	}
	article {
		box-shadow:
			0 4px 6px -1px rgb(0 0 0 / 30%),
			0 2px 4px -2px rgb(0 0 0 / 70%);
		padding: 2rem;
		position: relative;
		border-radius: 0.5rem;
		overflow: hidden;
		height: 100%;
		background-color: var(--sk-back-2);
		border-left: 1px solid var(--sk-theme-1);
		position: relative;
	}
	article > * {
		max-width: 80%;
	}
	.tree {
		right: 0;
		top: 0;
		width: 20%;
		bottom: 0;
		position: absolute;
		border-left: 1px solid var(--sk-code-bg);
		background: linear-gradient(-45deg, var(--sk-code-bg), transparent);
	}
	.buttons {
		margin-block-start: 2.5rem;
		display: flex;
		font-size: 2rem;
		gap: 0.5em;
	}
	p {
		margin: 0;
		font-size: 2rem;
		word-break: break-all;
	}
	small {
		color: var(--sk-text-2);
		display: block;
		margin-block: 0.5em;
	}

	form {
		line-height: 0;
	}

	.loading {
		position: absolute;
		inset: 0;
		display: grid;
		place-items: center;
		font-size: 3rem;
		background-color: var(--sk-back-5);
		opacity: 0.7;
		max-width: unset;
	}
	.loader {
		grid-column: 1/-1;
	}
</style>
