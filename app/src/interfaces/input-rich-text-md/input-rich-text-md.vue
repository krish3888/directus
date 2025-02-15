<template>
	<div class="interface-input-rich-text-md" :class="view[0]" ref="markdownInterface">
		<div class="toolbar">
			<v-menu show-arrow placement="bottom-start">
				<template #activator="{ toggle }">
					<v-button small icon @click="toggle" v-tooltip="t('wysiwyg_options.heading')">
						<v-icon name="title" />
					</v-button>
				</template>
				<v-list>
					<v-list-item v-for="n in 6" :key="n" clickable @click="edit('heading', { level: n })">
						<v-list-item-content><v-text-overflow :text="t(`wysiwyg_options.h${n}`)" /></v-list-item-content>
						<v-list-item-hint>{{ translateShortcut(['meta', 'alt']) }} {{ n }}</v-list-item-hint>
					</v-list-item>
				</v-list>
			</v-menu>

			<v-button
				small
				icon
				@click="edit('bold')"
				v-tooltip="t('wysiwyg_options.bold') + ' - ' + translateShortcut(['meta', 'b'])"
			>
				<v-icon name="format_bold" />
			</v-button>
			<v-button
				small
				icon
				@click="edit('italic')"
				v-tooltip="t('wysiwyg_options.italic') + ' - ' + translateShortcut(['meta', 'i'])"
			>
				<v-icon name="format_italic" />
			</v-button>
			<v-button
				small
				icon
				@click="edit('strikethrough')"
				v-tooltip="t('wysiwyg_options.strikethrough') + ' - ' + translateShortcut(['meta', 'alt', 'd'])"
			>
				<v-icon name="format_strikethrough" />
			</v-button>
			<v-button small icon @click="edit('listBulleted')" v-tooltip="t('wysiwyg_options.bullist')">
				<v-icon name="format_list_bulleted" />
			</v-button>
			<v-button small icon @click="edit('listNumbered')" v-tooltip="t('wysiwyg_options.numlist')">
				<v-icon name="format_list_numbered" />
			</v-button>
			<v-button
				small
				icon
				@click="edit('blockquote')"
				v-tooltip="t('wysiwyg_options.blockquote') + ' - ' + translateShortcut(['meta', 'alt', 'q'])"
			>
				<v-icon name="format_quote" />
			</v-button>
			<v-button
				small
				icon
				@click="edit('code')"
				v-tooltip="t('wysiwyg_options.codeblock') + ' - ' + translateShortcut(['meta', 'alt', 'c'])"
			>
				<v-icon name="code" />
			</v-button>
			<v-button
				small
				icon
				@click="edit('link')"
				v-tooltip="t('wysiwyg_options.link') + ' - ' + translateShortcut(['meta', 'k'])"
			>
				<v-icon name="insert_link" />
			</v-button>

			<v-menu show-arrow :close-on-content-click="false">
				<template #activator="{ toggle }">
					<v-button small icon @click="toggle" v-tooltip="t('wysiwyg_options.table')">
						<v-icon name="table_chart" />
					</v-button>
				</template>

				<template #default="{ deactivate }">
					<div class="table-options">
						<div class="field half">
							<p class="type-label">{{ t('rows') }}</p>
							<v-input :min="1" type="number" v-model="table.rows" />
						</div>
						<div class="field half">
							<p class="type-label">{{ t('columns') }}</p>
							<v-input :min="1" type="number" v-model="table.columns" />
						</div>
						<div class="field full">
							<v-button
								full-width
								@click="
									() => {
										edit('table', table);
										deactivate();
									}
								"
							>
								Create
							</v-button>
						</div>
					</div>
				</template>
			</v-menu>

			<v-button @click="imageDialogOpen = true" small icon v-tooltip="t('wysiwyg_options.image')">
				<v-icon name="insert_photo" />
			</v-button>

			<v-button
				v-for="custom in customSyntax"
				small
				icon
				:key="custom.name"
				@click="edit('custom', custom)"
				v-tooltip="custom.name"
			>
				<v-icon :name="custom.icon" />
			</v-button>

			<div class="spacer"></div>

			<v-item-group class="view" mandatory v-model="view" rounded>
				<v-button x-small value="editor">Editor</v-button>
				<v-button x-small value="preview">Preview</v-button>
			</v-item-group>
		</div>

		<div ref="codemirrorEl"></div>

		<div v-if="view[0] === 'preview'" class="preview-box" v-html="html"></div>

		<v-dialog :model-value="imageDialogOpen" @esc="imageDialogOpen = null" @update:model-value="imageDialogOpen = null">
			<v-card>
				<v-card-title>{{ t('upload_from_device') }}</v-card-title>
				<v-card-text>
					<v-upload @input="onImageUpload" from-url from-library />
				</v-card-text>
				<v-card-actions>
					<v-button @click="imageDialogOpen = null" secondary>{{ t('cancel') }}</v-button>
				</v-card-actions>
			</v-card>
		</v-dialog>
	</div>
</template>

<script lang="ts">
import { useI18n } from 'vue-i18n';
import { defineComponent, computed, ref, onMounted, watch, reactive, PropType } from 'vue';

import CodeMirror from 'codemirror';
import 'codemirror/mode/markdown/markdown';
import 'codemirror/addon/display/placeholder.js';

import { edit, CustomSyntax } from './edits';
import { getPublicURL } from '@/utils/get-root-path';
import { md } from '@/utils/md';
import { addTokenToURL } from '@/api';
import escapeStringRegexp from 'escape-string-regexp';
import useShortcut from '@/composables/use-shortcut';
import translateShortcut from '@/utils/translate-shortcut';

export default defineComponent({
	emits: ['input'],
	props: {
		value: {
			type: String,
			default: null,
		},
		disabled: {
			type: Boolean,
			default: false,
		},
		placeholder: {
			type: String,
			default: null,
		},
		customSyntax: {
			type: Array as PropType<CustomSyntax[]>,
			default: () => [],
		},
		imageToken: {
			type: String,
			default: undefined,
		},
	},
	setup(props, { emit }) {
		const { t } = useI18n();

		const markdownInterface = ref<HTMLElement>();
		const codemirrorEl = ref<HTMLTextAreaElement>();
		let codemirror: CodeMirror.Editor | null = null;

		const view = ref(['editor']);

		const imageDialogOpen = ref(false);

		onMounted(async () => {
			if (codemirrorEl.value) {
				codemirror = CodeMirror(codemirrorEl.value, {
					mode: 'markdown',
					configureMouse: () => ({ addNew: false }),
					lineWrapping: true,
					placeholder: props.placeholder,
					value: props.value || '',
				});

				codemirror.on('change', (cm) => {
					const content = cm.getValue();
					emit('input', content);
				});
			}
		});

		watch(
			() => props.value,
			(newValue) => {
				if (!codemirror) return;

				const existingValue = codemirror.getValue();

				if (existingValue !== newValue) {
					codemirror.setValue('');
					codemirror.clearHistory();
					codemirror.setValue(newValue);
					codemirror.refresh();
				}
			}
		);

		const html = computed(() => {
			let mdString = props.value || '';

			if (!props.imageToken) {
				const baseUrl = getPublicURL() + 'assets/';
				const regex = new RegExp(`\\]\\((${escapeStringRegexp(baseUrl)}[^\\s\\)]*)`, 'gm');

				const images = Array.from(mdString.matchAll(regex));

				for (const image of images) {
					mdString = mdString.replace(image[1], addTokenToURL(image[1]));
				}
			}

			const html = md(mdString);

			return html;
		});

		const table = reactive({
			rows: 4,
			columns: 4,
		});

		useShortcut('meta+b', () => edit(codemirror, 'bold'), markdownInterface);
		useShortcut('meta+i', () => edit(codemirror, 'italic'), markdownInterface);
		useShortcut('meta+k', () => edit(codemirror, 'link'), markdownInterface);
		useShortcut('meta+alt+d', () => edit(codemirror, 'strikethrough'), markdownInterface);
		useShortcut('meta+alt+q', () => edit(codemirror, 'blockquote'), markdownInterface);
		useShortcut('meta+alt+c', () => edit(codemirror, 'code'), markdownInterface);
		useShortcut('meta+alt+1', () => edit(codemirror, 'heading', { level: 1 }), markdownInterface);
		useShortcut('meta+alt+2', () => edit(codemirror, 'heading', { level: 2 }), markdownInterface);
		useShortcut('meta+alt+3', () => edit(codemirror, 'heading', { level: 3 }), markdownInterface);
		useShortcut('meta+alt+4', () => edit(codemirror, 'heading', { level: 4 }), markdownInterface);
		useShortcut('meta+alt+5', () => edit(codemirror, 'heading', { level: 5 }), markdownInterface);
		useShortcut('meta+alt+6', () => edit(codemirror, 'heading', { level: 6 }), markdownInterface);

		return {
			t,
			codemirrorEl,
			edit,
			view,
			html,
			table,
			onImageUpload,
			imageDialogOpen,
			useShortcut,
			translateShortcut,
			markdownInterface,
		};

		function onImageUpload(image: any) {
			if (!codemirror) return;

			let url = getPublicURL() + `assets/` + image.id;

			if (props.imageToken) {
				url += '?access_token=' + props.imageToken;
			}

			codemirror.replaceSelection(`![](${url})`);

			imageDialogOpen.value = false;
		}
	},
});
</script>

<style lang="scss" scoped>
@import '@/styles/mixins/form-grid';

.interface-input-rich-text-md {
	--v-button-background-color: transparent;
	--v-button-color: var(--foreground-normal);
	--v-button-background-color-hover: var(--border-normal);
	--v-button-color-hover: var(--foreground-normal);

	min-height: 300px;
	overflow: hidden;
	border: 2px solid var(--border-normal);
	border-radius: var(--border-radius);
}

textarea {
	display: none;
}

.preview-box {
	padding: 20px;
}

.preview-box :deep(h1) {
	margin-bottom: 0;
	font-weight: 300;
	font-size: 44px;
	font-family: var(--font-serif), serif;
	line-height: 52px;
}

.preview-box :deep(h2) {
	margin-top: 40px;
	margin-bottom: 0;
	font-weight: 600;
	font-size: 34px;
	line-height: 38px;
}

.preview-box :deep(h3) {
	margin-top: 40px;
	margin-bottom: 0;
	font-weight: 600;
	font-size: 26px;
	line-height: 31px;
}

.preview-box :deep(h4) {
	margin-top: 40px;
	margin-bottom: 0;
	font-weight: 600;
	font-size: 22px;
	line-height: 28px;
}

.preview-box :deep(h5) {
	margin-top: 40px;
	margin-bottom: 0;
	font-weight: 600;
	font-size: 18px;
	line-height: 26px;
}

.preview-box :deep(h6) {
	margin-top: 40px;
	margin-bottom: 0;
	font-weight: 600;
	font-size: 16px;
	line-height: 24px;
}

.preview-box :deep(p) {
	margin-top: 20px;
	margin-bottom: 20px;
	font-size: 16px;
	font-family: var(--font-serif), serif;
	line-height: 32px;
}

.preview-box :deep(a) {
	color: #546e7a;
}

.preview-box :deep(ul),
.preview-box :deep(ol) {
	margin: 24px 0;
	font-size: 18px;
	font-family: var(--font-serif), serif;
	line-height: 34px;
}

.preview-box :deep(ul ul),
.preview-box :deep(ol ol),
.preview-box :deep(ul ol),
.preview-box :deep(ol ul) {
	margin: 0;
}

.preview-box :deep(b),
.preview-box :deep(strong) {
	font-weight: 600;
}

.preview-box :deep(code) {
	padding: 2px 4px;
	font-size: 18px;
	font-family: var(--family-monospace), monospace;
	line-height: 34px;
	overflow-wrap: break-word;
	background-color: #eceff1;
	border-radius: 4px;
}

.preview-box :deep(pre) {
	padding: 20px;
	overflow: auto;
	font-size: 18px;
	font-family: var(--family-monospace), monospace;
	line-height: 24px;
	background-color: #eceff1;
	border-radius: 4px;
}

.preview-box :deep(blockquote) {
	margin-left: -10px;
	padding-left: 10px;
	font-size: 18px;
	font-family: var(--font-serif), serif;
	font-style: italic;
	line-height: 34px;
	border-left: 2px solid #546e7a;
}

.preview-box :deep(blockquote blockquote) {
	margin-left: 10px;
}

.preview-box :deep(video),
.preview-box :deep(iframe),
.preview-box :deep(img) {
	max-width: 100%;
	height: auto;
	border-radius: 4px;
}

.preview-box :deep(hr) {
	margin-top: 52px;
	margin-bottom: 56px;
	text-align: center;
	border: 0;
	border-top: 2px solid #cfd8dc;
}

.preview-box :deep(table) {
	border-collapse: collapse;
}

.preview-box :deep(table th),
.preview-box :deep(table td) {
	padding: 0.4rem;
	border: 1px solid #cfd8dc;
}

.preview-box :deep(figure) {
	display: table;
	margin: 1rem auto;
}

.preview-box :deep(figure figcaption) {
	display: block;
	margin-top: 0.25rem;
	color: #999;
	text-align: center;
}

.interface-input-rich-text-md :deep(.CodeMirror) {
	border: none;
	border-radius: 0;
}

.interface-input-rich-text-md :deep(.CodeMirror .CodeMirror-lines) {
	padding: 0 20px;
}

.interface-input-rich-text-md :deep(.CodeMirror .CodeMirror-lines:first-of-type) {
	margin-top: 20px;
}

.interface-input-rich-text-md :deep(.CodeMirror .CodeMirror-lines:last-of-type) {
	margin-bottom: 20px;
}

.interface-input-rich-text-md :deep(.CodeMirror .CodeMirror-scroll) {
	min-height: 260px;
}

.interface-input-rich-text-md.preview :deep(.CodeMirror) {
	display: none;
}

.toolbar {
	display: flex;
	flex-wrap: wrap;
	align-items: center;
	min-height: 40px;
	padding: 0 4px;
	background-color: var(--background-subdued);
	border-bottom: 2px solid var(--border-normal);

	.v-button + .v-button {
		margin-left: 2px;
	}

	.spacer {
		flex-grow: 1;
	}

	.view {
		--v-button-background-color: var(--border-subdued);
		--v-button-color: var(--foreground-subdued);
		--v-button-background-color-hover: var(--border-normal);
		--v-button-color-hover: var(--foreground-normal);
		--v-button-background-color-activated: var(--border-normal);
		--v-button-color-activated: var(--foreground-normal);
	}
}

.table-options {
	@include form-grid;

	--form-vertical-gap: 12px;
	--form-horizontal-gap: 12px;

	padding: 12px;

	.v-input {
		min-width: 100px;
	}
}
</style>
