<template>
	<div class="text-editor-wrapper" :class="{ loading: loading, 'icon-error': !loading && (!note || note.error), 'sidebar-open': sidebarOpen, 'is-mobile': isMobile }">
		<div v-show="!loading" ref="editor" class="text-editor" />
	</div>
</template>
<script>

import {
	isMobile,
} from '@nextcloud/vue'
import { emit, subscribe, unsubscribe } from '@nextcloud/event-bus'

import { queueCommand } from '../NotesService.js'
import { routeIsNewNote } from '../Util.js'
import store from '../store.js'

export default {
	name: 'NoteRich',

	mixins: [isMobile],

	props: {
		noteId: {
			type: String,
			required: true,
		},
	},

	data() {
		return {
			loading: false,
			editor: null,
			shouldAutotitle: true,
		}
	},

	computed: {
		note() {
			return store.getters.getNote(parseInt(this.noteId))
		},
		isNewNote() {
			return routeIsNewNote(this.$route)
		},
		sidebarOpen() {
			return store.state.app.sidebarOpen
		},
	},

	watch: {
		$route(to, from) {
			if (to.name !== from.name || to.params.noteId !== from.params.noteId) {
				this.onClose(from.params.noteId)
				this.fetchData()
			}
		},
	},

	created() {
		this.fetchData()
		subscribe('files:file:updated', this.fileUpdated)

	},

	destroyed() {
		this?.editor?.destroy()
		unsubscribe('files:file:updated', this.fileUpdated)
	},

	methods: {
		async fetchData() {
			this.etag = null

			if (this.isMobile) {
				emit('toggle-navigation', { open: false })
			}

			this.loading = true

			await this.loadTextEditor()
		},

		async loadTextEditor() {
			if (!this.$refs?.editor) {
				await this.$nextTick()
			}
			this?.editor?.destroy()
			this.loading = true
			this.shouldAutotitle = undefined
			this.editor = (await window.OCA.Text.createEditor({
				el: this.$refs.editor,
				fileId: parseInt(this.noteId),
				readOnly: false,
				onUpdate: ({ markdown }) => {
					if (this.note) {
						const unsaved = !!(this.note?.content && this.note.content !== markdown)
						if (this.shouldAutotitle === undefined) {
							const title = this.getTitle(markdown)
							this.shouldAutotitle = this.isNewNote || (title !== '' && title === this.note.title)
						}
						this.onEdit({ content: markdown, unsaved })
					}
				},
			}))
				.onLoaded(() => {
					this.loading = false
				})
		},

		onEdit(noteData = {}) {
			store.commit('updateNote', {
				...this.note,
				...noteData,
			})
		},

		onClose(noteId) {
			const note = store.getters.getNote(parseInt(noteId))
			store.commit('updateNote', {
				...note,
				unsaved: false,
			})
		},

		fileUpdated({ fileid }) {
			if (this.note.id === fileid) {
				this.onEdit({ unsaved: false })
				if (this.shouldAutotitle) {
					queueCommand(fileid, 'autotitle')
				}
			}
		},

		getTitle(content) {
			const firstLine = content.split('\n')[0] ?? ''
			const title = firstLine
				// See NoteUtil::sanitisePath
				.replaceAll(/^\s*[*+-]\s+/gmu, '')
				.replaceAll(/^[.\s]+/gmu, '')
				.replaceAll(/\*|\||\/|\\|:|"|'|<|>|\?/gmu, '')
				// See NoteUtil::stripMarkdown
				.replaceAll(/^#+\s+(.*?)\s*#*$/gmu, '$1')
				.replaceAll(/^(=+|-+)$/gmu, '')
				.replaceAll(/(\*+|_+)(.*?)\\1/gmu, '$2')
				.replaceAll(/\s/gmu, ' ')
			return title.length > 0 ? title : t('notes', 'New note')
		},
	},
}
</script>
<style lang="scss" scoped>
.text-editor-wrapper {
	height: 100%;
}

.text-editor {
	height: 100%;
}

.note-container {
	min-height: 100%;
	width: 100%;
	background-color: var(--color-main-background);
}

.is-mobile:deep(.text-menubar) {
	// Avoid overlapping the navigation toggle
	margin-left: 44px;
	margin-top: 4px;
}
</style>
