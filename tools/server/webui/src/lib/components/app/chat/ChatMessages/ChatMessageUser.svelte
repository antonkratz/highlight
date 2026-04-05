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
	let currentConfig = $derived(config());

	type AttentionHit = ChatAttentionItem & {
		rank: number;
		color: string;
		light: string;
		strong: string;
	};

	type AttentionSegment = {
		text: string;
		hit?: AttentionHit;
	};

	type AttentionPreview = {
		label: string;
		text: string;
		tokenIndex: number;
		head?: number;
		color: string;
		light: string;
		strong: string;
	};

	const HEAD_COLORS = ['#F97316', '#FBBF24', '#34D399', '#0EA5E9', '#A855F7', '#EF4444'];

	function headColor(head?: number): string {
		if (head === undefined || head < 0) {
			return HEAD_COLORS[0];
		}
		return HEAD_COLORS[head % HEAD_COLORS.length];
	}

	function withAlpha(hex: string, alpha: string): string {
		const normalized = hex.replace('#', '');
		return `#${normalized}${alpha}`;
	}

	function isSpecialToken(token: string): boolean {
		return /^<\|.*\|>$/.test(token) || token.startsWith('<|im_') || token.startsWith('<im_');
	}

	function overlaps(a: ChatAttentionItem, b: ChatAttentionItem): boolean {
		return Math.max(a.start, b.start) < Math.min(a.end, b.end);
	}

	function getPromptSlice(
		prompt: string,
		userContent: string
	): { text: string; offset: number; exact: boolean } {
		const trimmedContent = userContent.trim();
		if (!trimmedContent) {
			return { text: prompt, offset: 0, exact: false };
		}

		const start = prompt.lastIndexOf(trimmedContent);
		if (start === -1) {
			return { text: prompt, offset: 0, exact: false };
		}

		return {
			text: prompt.slice(start, start + trimmedContent.length),
			offset: start,
			exact: true
		};
	}

	function getRenderableItems(
		trace: NonNullable<DatabaseMessage['attentionTrace']>,
		promptOffset: number,
		promptLength: number
	): { items: ChatAttentionItem[]; useFullPrompt: boolean } {
		const inRange = trace.items
			.filter((item) => item.end > item.start)
			.filter((item) => item.end > promptOffset && item.start < promptOffset + promptLength);

		const withoutSpecialTokens = inRange.filter((item) => !isSpecialToken(item.token));
		if (withoutSpecialTokens.length > 0) {
			return {
				useFullPrompt: false,
				items: withoutSpecialTokens.map((item) => ({
					...item,
					start: Math.max(item.start, promptOffset) - promptOffset,
					end: Math.min(item.end, promptOffset + promptLength) - promptOffset
				}))
			};
		}

		if (inRange.length > 0) {
			return {
				useFullPrompt: false,
				items: inRange.map((item) => ({
					...item,
					start: Math.max(item.start, promptOffset) - promptOffset,
					end: Math.min(item.end, promptOffset + promptLength) - promptOffset
				}))
			};
		}

		return {
			useFullPrompt: true,
			items: trace.items
				.filter((item) => item.end > item.start)
				.map((item) => ({
					...item,
					start: item.start,
					end: item.end
				}))
		};
	}

	let attentionView = $derived.by(() => {
		const trace = message.attentionTrace;
		if (!trace?.prompt || !trace.items?.length) {
			return {
				layer: trace?.layer,
				tokenIndex: trace?.token_index ?? 0,
				prompt: '',
				segments: [] as AttentionSegment[],
				previews: [] as AttentionPreview[]
			};
		}

		const promptSlice = getPromptSlice(trace.prompt, message.content);
		const renderableResult = getRenderableItems(trace, promptSlice.offset, promptSlice.text.length);
		const prompt = renderableResult.useFullPrompt ? trace.prompt : promptSlice.text;
		const renderableItems = renderableResult.items
			.filter((item) => item.end > item.start)
			.filter((item) => item.end <= prompt.length);
		const candidates = [...renderableItems]
			.sort((a, b) => b.weight - a.weight);

		const selected: AttentionHit[] = [];
		for (const item of candidates) {
			if (selected.some((existing) => overlaps(existing, item))) {
				continue;
			}

			const color = headColor(item.head);
			selected.push({
				...item,
				rank: selected.length + 1,
				color,
				light: withAlpha(color, '33'),
				strong: withAlpha(color, 'bf')
			});
			if (selected.length === 3) {
				break;
			}
		}

		const segments: AttentionSegment[] = [];
		const highlightedItems: AttentionHit[] =
			selected.length > 0
				? [...selected].sort((a, b) => a.start - b.start)
				: [...renderableItems]
						.sort((a, b) => a.start - b.start)
						.map((item, index) => {
							const color = headColor(item.head);
							return {
								...item,
								rank: index + 1,
								color,
								light: withAlpha(color, '33'),
								strong: withAlpha(color, 'bf')
							};
						});
		let cursor = 0;

		for (const item of highlightedItems) {
			if (item.start > cursor) {
				segments.push({ text: prompt.slice(cursor, item.start) });
			}

			segments.push({
				text: prompt.slice(item.start, item.end),
				hit: item
			});

			cursor = item.end;
		}

		if (cursor < prompt.length) {
			segments.push({ text: prompt.slice(cursor) });
		}

		const previews = selected.map((item) => ({
			label: `#${item.rank}`,
			text: prompt.slice(item.start, item.end).trim() || prompt.slice(item.start, item.end),
			tokenIndex: item.token_index,
			head: item.head,
			color: item.color,
			light: item.light,
			strong: item.strong
		}));

		return {
			layer: trace.layer,
			tokenIndex: trace.token_index,
			prompt,
			segments,
			previews
		};
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

		{#if message.attentionTrace && attentionView.segments.length > 0}
			<Card class="max-w-[80%] border-primary/15 bg-background/80 px-3.75 py-3 backdrop-blur-md">
				<div class="mb-2 flex items-center justify-between gap-3 text-[11px] uppercase tracking-[0.18em] text-muted-foreground">
					<span>Live prompt focus</span>
					<span>
						{#if attentionView.layer !== undefined}
							L{attentionView.layer + 1}
						{/if}
						t{attentionView.tokenIndex}
					</span>
				</div>

				<div class="mb-3 grid gap-2 md:grid-cols-3">
					{#each attentionView.previews as preview}
						<div
							class="attention-preview-card"
							style={`--attention-card-color:${preview.color};--attention-card-light:${preview.light};--attention-card-strong:${preview.strong};`}
						>
							<div class="attention-preview-card__label">
								<span>{preview.label}</span>
								<span class="attention-preview-card__head">
									Head {preview.head ?? 0} • t{preview.tokenIndex}
								</span>
							</div>
							<div class="attention-preview-card__text">{preview.text}</div>
						</div>
					{/each}
				</div>

				<div
					class="max-h-52 overflow-y-auto whitespace-pre-wrap break-words font-mono text-xs leading-6 text-foreground/90"
				>
					{#each attentionView.segments as segment}
						{#if segment.hit}
							<span
								class="attention-hit"
							style={`--attention-strength:${Math.max(0.18, segment.hit.weight)}; --attention-color:${segment.hit.color}; --attention-light:${segment.hit.light}; --attention-strong:${segment.hit.strong};`}
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
		background: linear-gradient(
				180deg,
				var(--attention-light, rgba(249, 115, 22, 0.2)) 0%,
				var(--attention-color, rgba(249, 115, 22, 0.35)) 100%
			);
		box-shadow:
			inset 0 2px 0 var(--attention-strong, rgba(31, 99, 212, 0.9)),
			0 0 0 1px var(--attention-strong, rgba(31, 99, 212, 0.55));
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

	.attention-preview-card {
		border: 1px solid var(--attention-card-strong, rgba(249, 115, 22, 0.6));
		border-radius: 0.7rem;
		background: linear-gradient(
			180deg,
			var(--attention-card-light, rgba(249, 115, 22, 0.06)) 0%,
			var(--attention-card-color, rgba(249, 115, 22, 0.12)) 100%
		);
		padding: 0.55rem 0.7rem;
	}

	.attention-preview-card__label {
		display: flex;
		justify-content: space-between;
		gap: 0.75rem;
		margin-bottom: 0.35rem;
		font-family: var(--font-mono, monospace);
		font-size: 10px;
		letter-spacing: 0.12em;
		text-transform: uppercase;
		color: rgb(161 98 7);
	}

	.attention-preview-card__head {
		font-size: 9px;
		letter-spacing: 0.1em;
	}

	.attention-preview-card__text {
		font-family: var(--font-mono, monospace);
		font-size: 12px;
		line-height: 1.5;
		color: inherit;
		opacity: 0.92;
		white-space: pre-wrap;
		word-break: break-word;
	}
</style>
