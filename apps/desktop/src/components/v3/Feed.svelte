<script lang="ts">
	import { goto } from '$app/navigation';
	import ConfigurableScrollableContainer from '$components/ConfigurableScrollableContainer.svelte';
	import FeedItem from '$components/v3/FeedItem.svelte';
	import CliSymLink from '$components/v3/profileSettings/CliSymLink.svelte';
	import laneNewSvg from '$lib/assets/empty-state/lane-new.svg?raw';
	import { invoke } from '$lib/backend/ipc';
	import { projectAiGenEnabled } from '$lib/config/config';
	import { Feed } from '$lib/feed/feed';
	import { newProjectSettingsPath } from '$lib/routes/routes.svelte';
	import Badge from '@gitbutler/ui/Badge.svelte';
	import Button from '@gitbutler/ui/Button.svelte';
	import Icon from '@gitbutler/ui/Icon.svelte';
	import Spacer from '@gitbutler/ui/Spacer.svelte';
	import Link from '@gitbutler/ui/link/Link.svelte';
	import { onMount, tick } from 'svelte';

	type Props = {
		projectId: string;
		onCloseClick: () => void;
	};

	const { projectId, onCloseClick }: Props = $props();

	const feed = new Feed(projectId);
	const combinedEntries = feed.combined;

	let viewport = $state<HTMLDivElement>();
	let topSentinel = $state<HTMLDivElement>();
	let canLoadMore = $state(false);
	let prevScrollHeight = $state<number>(0);

	const aiGenEnabled = projectAiGenEnabled(projectId);

	async function loadMoreItems() {
		if (!canLoadMore || !viewport) return;
		canLoadMore = false;
		prevScrollHeight = viewport.scrollHeight;
		await feed.fetch();
		await tick();
		const newScrollHeight = viewport.scrollHeight;
		viewport.scrollTop = newScrollHeight - prevScrollHeight - 5;

		await tick();
		canLoadMore = true;
	}

	onMount(() => {
		if (viewport) {
			setTimeout(() => {
				viewport!.scrollTop = viewport!.scrollHeight;
				canLoadMore = true;
			}, 100);

			// Setup observer
			const observer = new IntersectionObserver(
				(entries) => {
					const first = entries[0];
					if (first?.isIntersecting) {
						loadMoreItems();
					}
				},
				{
					root: viewport,
					threshold: 0
				}
			);

			if (topSentinel) {
				observer.observe(topSentinel);
			}

			return () => {
				if (topSentinel) {
					observer.unobserve(topSentinel);
				}
			};
		}
	});

	let showCLISetupSteps = $state(false);
	let showSymlink = $state(false);
</script>

<div class="feed-wrap" class:has-actions={$combinedEntries.length > 0}>
	<div class="feed">
		{#if !aiGenEnabled}
			<div class="eneable-ai-banner">
				<Icon name="warning" color="warning" />
				<p class="text-13 text-bold flex-1">Enable AI generation</p>
				<Button
					kind="outline"
					onclick={() => {
						goto(newProjectSettingsPath(projectId, 'ai'));
					}}
				>
					Enable in settings
				</Button>
			</div>
		{/if}

		<div class="feed__header">
			<h2 class="flex-1 text-14 text-semibold">Butler Actions</h2>
			<Button icon="cross" kind="ghost" onclick={onCloseClick} />
		</div>

		<ConfigurableScrollableContainer childrenWrapHeight="100%" bind:viewport>
			{#if $combinedEntries.length === 0}
				<div class="feed__empty-state">
					<div class="feed__empty-state__content">
						<div class="feed__empty-state__image">
							{@html laneNewSvg}
						</div>

						<h3 class="text-15 text-bold m-bottom-12">Welcome to GitButler Actions!</h3>
						<ul class="feed__empty-state__benefits">
							<li class="text-13 text-body">
								<p class="m-bottom-4">
									<span class="m-right-4">✦</span> Connect your Agentic workflow with GitButler. Editors
									with MCP support (like Cursor, VSCode, Zed) can have changes automatically version
									controlled.
								</p>
								<Link
									href="https://docs.gitbutler.com/agentic-workflows"
									target="_blank"
									class="clr-text-2">Learn more</Link
								>
							</li>
							<li class="text-13 text-body">
								<p class="m-bottom-6">
									<span class="m-right-4">✷</span>
									Perform automated workflows directly from the app — semantic splitting, amending and
									reorganizing of commits.
								</p>
								<Badge kind="soft" style="pop" size="tag">Comming soon!</Badge>
							</li>
						</ul>

						<Spacer margin={28} />

						<div class="cli-setup">
							<p class="text-13 text-body clr-text-2">
								Using Cursor or other agent-based IDEs? Setup GitButler's MCP integration for
								enhanced workflow automation.
							</p>

							<div
								role="presentation"
								class="cli-setup__fold-btn"
								onclick={() => {
									showCLISetupSteps = !showCLISetupSteps;
								}}
							>
								<div class="cli-setup__fold-icon" class:rotate-icon={showCLISetupSteps}>
									<Icon name="chevron-right" />
								</div>
								<p class="text-15 text-bold underline-dotted">Install GitButler CLI</p>
								<Icon name="robot" />
							</div>

							{#if showCLISetupSteps}
								<ul class="cli-setup__steps text-13 text-body">
									<li>
										<Button
											kind="outline"
											icon="play"
											size="tag"
											onclick={async () => await invoke('install_cli')}>Install But CLI</Button
										>
										or
										<span
											role="presentation"
											class="underline-dotted"
											onclick={() => {
												showSymlink = !showSymlink;
											}}>configure manually</span
										>
										<span class="clr-text-2">(requires admin rights)</span>.
									</li>
									{#if showSymlink}
										<CliSymLink class="m-top-2 m-bottom-6" />
									{/if}
									<li>
										Cursor / VSCode setup. <Link
											class="clr-text-2"
											href="https://docs.gitbutler.com/agentic-workflows/cursor">Learn more</Link
										>
									</li>
								</ul>
							{/if}
						</div>
					</div>
				</div>
			{:else}
				<div class="feed-list">
					{#each $combinedEntries as entry (entry.id)}
						<FeedItem {projectId} action={entry} />
					{/each}
					<div bind:this={topSentinel} style="height: 1px"></div>
				</div>
			{/if}
		</ConfigurableScrollableContainer>
	</div>
</div>

<style lang="postcss">
	.feed-wrap {
		display: flex;
		position: relative;
		height: 100%;
		overflow: hidden;
		border-radius: var(--radius-ml);
	}

	.feed {
		display: flex;
		position: relative;
		flex-direction: column;
		width: 100%;
		height: 100%;
		background-color: var(--clr-bg-1);
	}

	.feed__header {
		display: flex;
		align-items: center;
		width: 100%;
		padding: 8px 8px 8px 14px;
		gap: 8px;
		border-bottom: 1px solid var(--clr-border-2);
	}

	.feed-list {
		display: flex;
		flex-direction: column-reverse;
		min-height: 100%;
	}

	.feed__empty-state {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		min-height: 100%;
		padding: 40px;
		background-color: var(--clr-bg-2);
	}

	.feed__empty-state__content {
		display: flex;
		flex-direction: column;
		max-width: 440px;
		margin-bottom: 40px;
	}

	.feed__empty-state__benefits {
		display: flex;
		flex-direction: column;
		gap: 16px;
	}

	.feed__empty-state__image {
		display: flex;
		margin-bottom: 30px;
	}

	.cli-setup {
		display: flex;
		flex-direction: column;
		gap: 14px;
	}

	.cli-setup__fold-btn {
		display: flex;
		align-items: center;
		margin-left: -4px;
		gap: 8px;
		cursor: pointer;

		&:hover {
			& .cli-setup__fold-icon {
				color: var(--clr-text-1);
			}
		}
	}

	.cli-setup__fold-icon {
		display: flex;
		color: var(--clr-text-2);
		transition:
			transform var(--transition-medium),
			color var(--transition-fast);

		&.rotate-icon {
			transform: rotate(90deg);
		}
	}

	.cli-setup__steps {
		display: flex;
		flex-direction: column;
		margin-left: 20px;
		gap: 6px;
		list-style-type: decimal;
	}

	.eneable-ai-banner {
		display: flex;
		z-index: var(--z-ground);
		position: absolute;
		right: 14px;
		bottom: 14px;
		left: 14px;
		align-items: center;
		justify-content: center;
		padding: 12px;
		padding-left: 16px;
		gap: 10px;
		border: 1px solid var(--clr-border-2);
		border-radius: var(--radius-m);
		background-color: var(--clr-bg-1);
		box-shadow: var(--fx-shadow-m);
	}
</style>
