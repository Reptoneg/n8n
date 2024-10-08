<script setup lang="ts">
import { computed, ref } from 'vue';
import AssistantIcon from '../AskAssistantIcon/AssistantIcon.vue';
import AssistantText from '../AskAssistantText/AssistantText.vue';
import AssistantAvatar from '../AskAssistantAvatar/AssistantAvatar.vue';
import CodeDiff from '../CodeDiff/CodeDiff.vue';
import type { ChatUI } from '../../types/assistant';
import BlinkingCursor from '../BlinkingCursor/BlinkingCursor.vue';

import Markdown from 'markdown-it';
import InlineAskAssistantButton from '../InlineAskAssistantButton/InlineAskAssistantButton.vue';
import BetaTag from '../BetaTag/BetaTag.vue';
import { useI18n } from '../../composables/useI18n';
import markdownLink from 'markdown-it-link-attributes';

const { t } = useI18n();

const md = new Markdown({
	breaks: true,
}).use(markdownLink, {
	attrs: {
		target: '_blank',
		rel: 'noopener',
	},
});

const MAX_CHAT_INPUT_HEIGHT = 100;

interface Props {
	user?: {
		firstName: string;
		lastName: string;
	};
	messages?: ChatUI.AssistantMessage[];
	streaming?: boolean;
}

const emit = defineEmits<{
	close: [];
	message: [string, string | undefined];
	codeReplace: [number];
	codeUndo: [number];
}>();

const onClose = () => emit('close');

const props = defineProps<Props>();

const textInputValue = ref<string>('');

const chatInput = ref<HTMLTextAreaElement | null>(null);

const sessionEnded = computed(() => {
	return isEndOfSessionEvent(props.messages?.[props.messages.length - 1]);
});

const sendDisabled = computed(() => {
	return !textInputValue.value || props.streaming || sessionEnded.value;
});

function isEndOfSessionEvent(event?: ChatUI.AssistantMessage) {
	return event?.type === 'event' && event?.eventName === 'end-session';
}

function onQuickReply(opt: ChatUI.QuickReply) {
	emit('message', opt.text, opt.type);
}

function onSendMessage() {
	if (sendDisabled.value) return;
	emit('message', textInputValue.value, undefined);
	textInputValue.value = '';
	if (chatInput.value) {
		chatInput.value.style.height = 'auto';
	}
}

function renderMarkdown(content: string) {
	try {
		return md.render(content);
	} catch (e) {
		console.error(`Error parsing markdown content ${content}`);
		return `<p>${t('assistantChat.errorParsingMarkdown')}</p>`;
	}
}

function growInput() {
	if (!chatInput.value) return;
	chatInput.value.style.height = 'auto';
	const scrollHeight = chatInput.value.scrollHeight;
	chatInput.value.style.height = `${Math.min(scrollHeight, MAX_CHAT_INPUT_HEIGHT)}px`;
}
</script>

<template>
	<div :class="$style.container">
		<div :class="$style.header">
			<div :class="$style.chatTitle">
				<div :class="$style.headerText">
					<AssistantIcon size="large" />
					<AssistantText size="large" :text="t('assistantChat.aiAssistantLabel')" />
				</div>
				<BetaTag />
			</div>
			<div :class="$style.back" @click="onClose">
				<n8n-icon icon="arrow-right" color="text-base" />
			</div>
		</div>
		<div :class="$style.body">
			<div v-if="messages?.length" :class="$style.messages">
				<div v-for="(message, i) in messages" :key="i" :class="$style.message">
					<div
						v-if="
							!isEndOfSessionEvent(message) && (i === 0 || message.role !== messages[i - 1].role)
						"
						:class="{ [$style.roleName]: true, [$style.userSection]: i > 0 }"
					>
						<AssistantAvatar v-if="message.role === 'assistant'" />
						<n8n-avatar
							v-else
							:first-name="user?.firstName"
							:last-name="user?.lastName"
							size="xsmall"
						/>

						<span v-if="message.role === 'assistant'">{{
							t('assistantChat.aiAssistantName')
						}}</span>
						<span v-else>{{ t('assistantChat.you') }}</span>
					</div>
					<div v-if="message.type === 'block'">
						<div :class="$style.block">
							<div :class="$style.blockTitle">
								{{ message.title }}
								<BlinkingCursor
									v-if="streaming && i === messages?.length - 1 && !message.content"
								/>
							</div>
							<div :class="$style.blockBody">
								<!-- eslint-disable-next-line vue/no-v-html -->
								<span v-html="renderMarkdown(message.content)"></span>
								<BlinkingCursor
									v-if="streaming && i === messages?.length - 1 && message.title && message.content"
								/>
							</div>
						</div>
					</div>
					<div v-else-if="message.type === 'text'" :class="$style.textMessage">
						<!-- eslint-disable-next-line vue/no-v-html -->
						<span v-if="message.role === 'user'" v-html="renderMarkdown(message.content)"></span>
						<!-- eslint-disable-next-line vue/no-v-html -->
						<span
							v-else
							:class="$style.assistantText"
							v-html="renderMarkdown(message.content)"
						></span>
						<BlinkingCursor
							v-if="streaming && i === messages?.length - 1 && message.role === 'assistant'"
						/>
					</div>
					<div v-else-if="message.type === 'error'" :class="$style.error">
						<span>⚠️ {{ message.content }}</span>
					</div>
					<div v-else-if="message.type === 'code-diff'">
						<CodeDiff
							:title="message.description"
							:content="message.codeDiff"
							:replacing="message.replacing"
							:replaced="message.replaced"
							:error="message.error"
							:streaming="streaming && i === messages?.length - 1"
							@replace="() => emit('codeReplace', i)"
							@undo="() => emit('codeUndo', i)"
						/>
					</div>
					<div v-else-if="isEndOfSessionEvent(message)" :class="$style.endOfSessionText">
						<span>
							{{ t('assistantChat.sessionEndMessage.1') }}
						</span>
						<InlineAskAssistantButton size="small" :static="true" />
						<span>
							{{ t('assistantChat.sessionEndMessage.2') }}
						</span>
					</div>

					<div
						v-if="
							!streaming &&
							'quickReplies' in message &&
							message.quickReplies?.length &&
							i === messages?.length - 1
						"
						:class="$style.quickReplies"
					>
						<div :class="$style.quickRepliesTitle">{{ t('assistantChat.quickRepliesTitle') }}</div>
						<div v-for="opt in message.quickReplies" :key="opt.type">
							<n8n-button
								v-if="opt.text"
								type="secondary"
								size="mini"
								@click="() => onQuickReply(opt)"
							>
								{{ opt.text }}
							</n8n-button>
						</div>
					</div>
				</div>
			</div>

			<div v-else :class="$style.placeholder">
				<div :class="$style.greeting">Hi {{ user?.firstName }} 👋</div>
				<div :class="$style.info">
					<p>
						{{
							t('assistantChat.placeholder.1', [
								`${user?.firstName}`,
								t('assistantChat.aiAssistantName'),
							])
						}}
					</p>
					<p>
						{{ t('assistantChat.placeholder.2') }}
						<InlineAskAssistantButton size="small" :static="true" />
						{{ t('assistantChat.placeholder.3') }}
					</p>
					<p>
						{{ t('assistantChat.placeholder.4') }}
					</p>
				</div>
			</div>
		</div>
		<div
			v-if="messages?.length"
			:class="{ [$style.inputWrapper]: true, [$style.disabledInput]: sessionEnded }"
		>
			<textarea
				ref="chatInput"
				v-model="textInputValue"
				:disabled="sessionEnded"
				:placeholder="t('assistantChat.inputPlaceholder')"
				rows="1"
				wrap="hard"
				@keydown.enter.exact.prevent="onSendMessage"
				@input.prevent="growInput"
				@keydown.stop
			/>
			<n8n-icon-button
				:class="{ [$style.sendButton]: true }"
				icon="paper-plane"
				type="text"
				size="large"
				:disabled="sendDisabled"
				@click="onSendMessage"
			/>
		</div>
	</div>
</template>

<style lang="scss" module>
.container {
	height: 100%;
	position: relative;
}

p {
	line-height: var(--font-line-height-xloose);
	margin: var(--spacing-2xs) 0;
}

.header {
	height: 65px; // same as header height in editor
	padding: 0 var(--spacing-l);
	background-color: var(--color-background-xlight);
	border: var(--border-base);
	border-top: 0;
	display: flex;

	div {
		display: flex;
		align-items: center;
	}

	> div:first-of-type {
		width: 100%;
	}
}

.body {
	background-color: var(--color-background-light);
	border: var(--border-base);
	border-top: 0;
	height: 100%;
	overflow: scroll;
	padding-bottom: 250px; // make scrollable at the end
	position: relative;

	pre {
		text-wrap: stable;
	}
}

.placeholder {
	padding: var(--spacing-s);
}

.messages {
	padding: var(--spacing-xs);
}

.message {
	margin-bottom: var(--spacing-xs);
	font-size: var(--font-size-2xs);
	line-height: var(--font-line-height-xloose);
}

.roleName {
	display: flex;
	align-items: center;
	margin-bottom: var(--spacing-3xs);

	font-weight: var(--font-weight-bold);
	font-size: var(--font-size-2xs);

	> * {
		margin-right: var(--spacing-3xs);
	}
}

.userSection {
	margin-top: var(--spacing-l);
}

.chatTitle {
	display: flex;
	gap: var(--spacing-3xs);
}

.headerText {
	gap: var(--spacing-xs);
}

.greeting {
	color: var(--color-text-dark);
	font-size: var(--font-size-m);
	margin-bottom: var(--spacing-s);
}

.info {
	font-size: var(--font-size-s);
	color: var(--color-text-base);

	button {
		display: inline-flex;
	}
}

.back:hover {
	cursor: pointer;
}

.quickReplies {
	margin-top: var(--spacing-s);
	> * {
		margin-bottom: var(--spacing-3xs);
	}
}

.quickRepliesTitle {
	font-size: var(--font-size-3xs);
	color: var(--color-text-base);
}

.textMessage {
	font-size: var(--font-size-2xs);
}

.block {
	font-size: var(--font-size-2xs);
	background-color: var(--color-foreground-xlight);
	border: var(--border-base);
	border-radius: var(--border-radius-base);
	word-break: break-word;

	li {
		margin-left: var(--spacing-xs);
	}
}

.blockTitle {
	border-bottom: var(--border-base);
	padding: var(--spacing-2xs);
	font-weight: var(--font-weight-bold);
}

.blockBody {
	padding: var(--spacing-xs);
}

.inputWrapper {
	position: absolute;
	display: flex;
	bottom: 0;
	background-color: var(--color-foreground-xlight);
	border: var(--border-base);
	width: 100%;
	padding-top: 1px;

	textarea {
		border: none;
		background-color: transparent;
		width: 100%;
		font-size: var(--spacing-xs);
		padding: var(--spacing-xs);
		outline: none;
		color: var(--color-text-dark);
		resize: none;
		font-family: inherit;
	}
}

.sendButton {
	color: var(--color-text-base) !important;

	&[disabled] {
		color: var(--color-text-light) !important;
	}
}

.error {
	color: var(--color-danger);
}

.assistantText {
	display: inline;

	p {
		display: inline;
		line-height: 1.7;
	}

	ul,
	ol {
		list-style-position: inside;
		margin: var(--spacing-xs) 0 var(--spacing-xs) var(--spacing-xs);
	}
}

.endOfSessionText {
	margin-top: var(--spacing-l);
	padding-top: var(--spacing-3xs);
	border-top: var(--border-base);
	color: var(--color-text-base);

	> button,
	> span {
		margin-right: var(--spacing-3xs);
	}

	button {
		display: inline-flex;
	}
}

.disabledInput {
	cursor: not-allowed;

	* {
		cursor: not-allowed;
	}
}
</style>
