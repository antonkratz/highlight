<script lang="ts">
	import { Card } from '$lib/components/ui/card';
	import { ChatAttachmentsList, MarkdownContent } from '$lib/components/app';
	import { getMessageEditContext } from '$lib/contexts';
	import { config } from '$lib/stores/settings.svelte';
	import ChatMessageActions from './ChatMessageActions.svelte';
	import ChatMessageEditForm from './ChatMessageEditForm.svelte';
	import { MessageRole } from '$lib/enums';
	import type { ChatAttentionItem } from '$lib/types/chat';

	interface Props {
		class?: string;
		message: DatabaseMessage;
		siblingInfo?: ChatMessageSiblingInfo | null;
		deletionInfo: {
			totalCount: number;
			userMessages: number;
			assistantMessages: number;
			messageTypes: string[];
		} | null;
		showDeleteDialog: boolean;
		onEdit: () => void;
		onDelete: () => void;
		onConfirmDelete: () => void;
		onForkConversation?: (options: { name: string; includeAttachments: boolean }) => void;
		onShowDeleteDialogChange: (show: boolean) => void;
		onNavigateToSibling?: (siblingId: string) => void;
		onCopy: () => void;
	}

	let {
		class: className = '',
		message,
		siblingInfo = null,
		deletionInfo,
		showDeleteDialog,
		onEdit,
		onDelete,
		onConfirmDelete,
		onForkConversation,
		onShowDeleteDialogChange,
		onNavigateToSibling,
		onCopy
	}: Props = $props();

	// Get contexts
	const editCtx = getMessageEditContext();

	let isMultiline = $state(false);
	let messageElement: HTMLElement | undefined = $state();
	const currentConfig = config();

	type AttentionSegment = {
		text: string;
		hit?: ChatAttentionItem & { rank: number };
	};

	let attentionSegments = $derived.by(() => {
		const trace = message.attentionTrace;
		if (!trace?.prompt || !trace.items?.length) {
			return [] as AttentionSegment[];
		}

		const segments: AttentionSegment[] = [];
		const items = [...trace.items]
			.sort((a, b) => a.start - b.start)
			.map((item, index) => ({ ...item, rank: index + 1 }));
		let cursor = 0;

		for (const item of items) {
			if (item.start > cursor) {
				segments.push({ text: trace.prompt.slice(cursor, item.start) });
			}

			segments.push({
				text: trace.prompt.slice(item.start, item.end),
				hit: item
			});

			cursor = item.end;
		}

		if (cursor < trace.prompt.length) {
			segments.push({ text: trace.prompt.slice(cursor) });
		}

		return segments;
	});

	$effect(() => {
		if (!messageElement || !message.content.trim()) return;

		if (message.content.includes('\n')) {
			isMultiline = true;
			return;
		}

		const resizeObserver = new ResizeObserver((entries) => {
			for (const entry of entries) {
				const element = entry.target as HTMLElement;
				const estimatedSingleLineHeight = 24; // Typical line height for text-md

				isMultiline = element.offsetHeight > estimatedSingleLineHeight * 1.5;
			}
		});

		resizeObserver.observe(messageElement);

		return () => {
			resizeObserver.disconnect();
		};
	});
</script>

<div
	aria-label="User message with actions"
	class="group flex flex-col items-end gap-3 md:gap-2 {className}"
	role="group"
>
	{#if editCtx.isEditing}
		<ChatMessageEditForm />
	{:else}
		{#if message.extra && message.extra.length > 0}
			<div class="mb-2 max-w-[80%]">
				<ChatAttachmentsList attachments={message.extra} readonly imageHeight="h-80" />
			</div>
		{/if}

		{#if message.content.trim()}
			<Card
				class="max-w-[80%] overflow-y-auto rounded-[1.125rem] border-none bg-primary/5 px-3.75 py-1.5 text-foreground backdrop-blur-md data-[multiline]:py-2.5 dark:bg-primary/15"
				data-multiline={isMultiline ? '' : undefined}
				style="max-height: var(--max-message-height); overflow-wrap: anywhere; word-break: break-word;"
			>
				{#if currentConfig.renderUserContentAsMarkdown}
					<div bind:this={messageElement}>
						<MarkdownContent class="markdown-user-content -my-4" content={message.content} />
					</div>
				{:else}
					<span bind:this={messageElement} class="text-md whitespace-pre-wrap">
						{message.content}
					</span>
				{/if}
			</Card>
		{/if}

		{#if message.attentionTrace && attentionSegments.length > 0}
			<Card class="max-w-[80%] border-primary/15 bg-background/80 px-3.75 py-3 backdrop-blur-md">
				<div class="mb-2 flex items-center justify-between gap-3 text-[11px] uppercase tracking-[0.18em] text-muted-foreground">
					<span>Live prompt focus</span>
					<span>
						{#if message.attentionTrace.layer !== undefined}
							L{message.attentionTrace.layer + 1}
						{/if}
						t{message.attentionTrace.token_index}
					</span>
				</div>

				<div
					class="max-h-52 overflow-y-auto whitespace-pre-wrap break-words font-mono text-xs leading-6 text-foreground/90"
				>
					{#each attentionSegments as segment}
						{#if segment.hit}
							<span
								class="attention-hit"
								style={`--attention-strength:${Math.max(0.18, segment.hit.weight)};`}
								title={`prompt token ${segment.hit.token_index} • weight ${segment.hit.weight.toFixed(3)}`}
							>
								<span class="attention-hit__marker">{segment.hit.rank}</span>{segment.text}
							</span>
						{:else}
							<span>{segment.text}</span>
						{/if}
					{/each}
				</div>
			</Card>
		{/if}

		{#if message.timestamp}
			<div class="max-w-[80%]">
				<ChatMessageActions
					actionsPosition="right"
					{deletionInfo}
					justify="end"
					{onConfirmDelete}
					{onCopy}
					{onDelete}
					{onEdit}
					{onForkConversation}
					{onNavigateToSibling}
					{onShowDeleteDialogChange}
					{siblingInfo}
					{showDeleteDialog}
					role={MessageRole.USER}
				/>
			</div>
		{/if}
	{/if}
</div>

<style>
	.attention-hit {
		position: relative;
		border-radius: 0.35rem;
		background:
			linear-gradient(
				180deg,
				rgba(251, 191, 36, 0.18) 0%,
				rgba(251, 191, 36, calc(var(--attention-strength) * 0.7)) 100%
			);
		box-shadow: inset 0 2px 0 rgba(251, 191, 36, calc(var(--attention-strength) + 0.15));
		padding: 0.15rem 0.05rem 0.05rem;
	}

	.attention-hit__marker {
		position: absolute;
		top: -0.6rem;
		left: 0.15rem;
		font-size: 9px;
		font-weight: 700;
		line-height: 1;
		color: rgb(180 83 9);
	}
</style>
