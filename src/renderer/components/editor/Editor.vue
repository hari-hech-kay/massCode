<script setup lang="ts">
import type { Language } from '@/components/editor/types'
import { nextTick } from 'node:process'
import { createCodeHighlight } from '@/components/cm-extensions/codeHighlight'
import { editorScrollbarTheme } from '@/components/cm-extensions/scrollbarTheme'

import {
  useApp,
  useDonations,
  useEditor,
  useResizeHandle,
  useSnippets,
  useSnippetUpdate,
  useTheme,
} from '@/composables'
import { i18n, ipc } from '@/electron'
import {
  mapNormalizedCursorIndex,
  normalizeTerminalText,
} from '@/utils/normalizeTerminalText'
import { closeBrackets, closeBracketsKeymap } from '@codemirror/autocomplete'
import { defaultKeymap, history, historyKeymap } from '@codemirror/commands'
import {
  foldGutter as foldGutterExtension,
  indentUnit,
  LanguageDescription,
  matchBrackets,
} from '@codemirror/language'
import { languages } from '@codemirror/language-data'
import {
  search,
  searchKeymap,
  SearchQuery,
  setSearchQuery,
} from '@codemirror/search'
import { Compartment, EditorState } from '@codemirror/state'
import {
  dropCursor,
  EditorView,
  highlightActiveLine,
  keymap,
  lineNumbers as lineNumbersExtension,
} from '@codemirror/view'
import { useClipboard, useCssVar, useDebounceFn } from '@vueuse/core'
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'

const { settings, cursorPosition } = useEditor()
const {
  selectedSnippetContent,
  selectedSnippet,
  isEmpty,
  selectedSnippetIds,
  isAvailableToCodePreview,
  searchQuery,
} = useSnippets()
const { isShowCodePreview, isShowCodeImage, isShowJsonVisualizer } = useApp()
const { isDark } = useTheme()

const {
  addToUpdateContentQueue,
  getPendingContentUpdate,
  isContentUpdateBusy,
} = useSnippetUpdate()

let view: EditorView | null = null
let lastAppliedContentId: number | undefined

const previewHandleRef = ref<HTMLElement>()
const previewHeight = ref(300)

useResizeHandle(previewHandleRef, {
  direction: 'vertical',
  onMove(dy) {
    previewHeight.value = Math.max(100, previewHeight.value - dy)
    view?.requestMeasure()
  },
})

const isProgrammaticChange = ref(false)

useCssVar('--editor-font-size', document.body, {
  initialValue: `${settings.fontSize}px`,
})

useCssVar('--editor-font-family', document.body, {
  initialValue: settings.fontFamily,
})

const scrollBarOpacity = useCssVar(
  '--editor-scrollbar-opacity',
  document.body,
  {
    initialValue: '1',
  },
)

const isShowHeader = computed(() => {
  if (selectedSnippetIds.value.length > 1)
    return false
  return !isEmpty.value && selectedSnippet.value !== undefined
})
const isShowEditor = computed(() => {
  if (selectedSnippetIds.value.length > 1)
    return false
  return (
    !isShowCodeImage.value
    && !isShowJsonVisualizer.value
    && !isEmpty.value
    && selectedSnippet.value !== undefined
  )
})

watch(selectedSnippetContent, () => {
  if (selectedSnippetContent.value?.language !== 'json') {
    isShowJsonVisualizer.value = false
  }

  if (!isAvailableToCodePreview.value) {
    isShowCodePreview.value = false
  }
})

function getCursorPosition() {
  if (!view)
    return
  const pos = view.state.selection.main.head
  const line = view.state.doc.lineAt(pos)
  cursorPosition.row = line.number - 1
  cursorPosition.column = pos - line.from
}

const hideScrollbar = useDebounceFn(() => {
  scrollBarOpacity.value = '0'
}, 1000)

const languageConf = new Compartment()
const themeConf = new Compartment()
const lineWrappingConf = new Compartment()
const tabSizeConf = new Compartment()
const lineNumbersConf = new Compartment()
const matchBracketsConf = new Compartment()
const highlightLineConf = new Compartment()

async function loadLanguage(lang: string | undefined) {
  if (!lang || lang === 'plain_text')
    return null
  const desc = LanguageDescription.matchLanguageName(languages, lang, true)
  if (desc) {
    return await desc.load()
  }
  return null
}

const baseTheme = EditorView.theme({
  '&': {
    height: '100%',
    backgroundColor: 'var(--background)',
    color: 'var(--foreground)',
  },
  '.cm-content': {
    fontFamily: 'var(--editor-font-family)',
    fontSize: 'var(--editor-font-size)',
    lineHeight: 'calc(var(--editor-font-size) * 1.5)',
  },
  '.cm-gutters': {
    backgroundColor: 'var(--background)',
    color: 'var(--muted-foreground)',
    border: 'none',
  },
  '.cm-activeLine, .cm-activeLineGutter': {
    backgroundColor: 'var(--accent)',
  },
  '.cm-selectionBackground': {
    backgroundColor: 'var(--accent) !important',
  },
  '&.cm-focused .cm-selectionBackground': {
    backgroundColor: 'var(--accent) !important',
  },
  '.cm-cursor': {
    borderLeftColor: 'var(--foreground)',
  },
  '.cm-searchMatch': {
    backgroundColor: 'var(--text-highlight)',
    color: 'black !important',
    borderRadius: '2px',
  },
  '.cm-searchMatch.cm-searchMatch-selected': {
    backgroundColor: 'var(--text-highlight)',
  },
  ...editorScrollbarTheme,
})

async function init() {
  const el = document.getElementById('editor')

  if (!el)
    return

  const initialLang = selectedSnippetContent.value?.language || 'plain_text'
  const langSupport = await loadLanguage(initialLang)

  const extensions = [
    baseTheme,
    history(),
    dropCursor(),
    closeBrackets(),
    search({ top: true }),
    keymap.of([
      ...closeBracketsKeymap,
      ...defaultKeymap,
      ...historyKeymap,
      ...searchKeymap,
    ]),
    themeConf.of(createCodeHighlight(isDark.value)),
    languageConf.of(langSupport ? [langSupport] : []),
    lineWrappingConf.of(settings.wrap ? EditorView.lineWrapping : []),
    tabSizeConf.of(indentUnit.of(' '.repeat(Math.max(1, settings.tabSize)))),
    EditorState.tabSize.of(Math.max(1, settings.tabSize)),
    lineNumbersConf.of(lineNumbersExtension()),
    foldGutterExtension(),
    matchBracketsConf.of(settings.matchBrackets ? matchBrackets() : []),
    highlightLineConf.of(settings.highlightLine ? highlightActiveLine() : []),
    EditorView.updateListener.of((update) => {
      if (
        update.docChanged
        && !isProgrammaticChange.value
        && selectedSnippet.value?.id
      ) {
        const content = selectedSnippetContent.value
        if (
          !content
          || content.value === undefined
          || content.id !== lastAppliedContentId
        ) {
          return
        }

        const updatedValue = update.state.doc.toString()

        if (content.value !== updatedValue) {
          addToUpdateContentQueue(selectedSnippet.value.id, content.id, {
            label: content.label,
            value: updatedValue,
            language: content.language,
          })
        }
      }

      if (update.selectionSet) {
        getCursorPosition()
      }
    }),
    EditorView.domEventHandlers({
      drop: (e, view) => {
        if (selectedSnippetContent.value?.language === 'markdown') {
          const file = e.dataTransfer?.files[0]

          if (!file)
            return false

          if (!file.type.startsWith('image/'))
            return false

          e.preventDefault()

          file.arrayBuffer().then(async (arrayBuffer) => {
            const buffer = Array.from(new Uint8Array(arrayBuffer))

            try {
              const relativePath = await ipc.invoke('fs:assets', {
                buffer,
                fileName: file.name,
              })

              const insertText = `![${file.name}](./${relativePath})`
              const pos = view.posAtCoords({ x: e.clientX, y: e.clientY })

              if (pos !== null) {
                view.dispatch({
                  changes: { from: pos, insert: insertText },
                  selection: { anchor: pos + insertText.length },
                })
                view.focus()
              }
            }
            catch (error) {
              console.error('Ошибка при добавлении изображения:', error)
            }
          })

          return true
        }
        return false
      },
      scroll: () => {
        scrollBarOpacity.value = '1'
        hideScrollbar()
      },
    }),
  ]

  if (selectedSnippetContent.value?.value !== undefined) {
    lastAppliedContentId = selectedSnippetContent.value.id
  }

  view = new EditorView({
    state: EditorState.create({
      doc: selectedSnippetContent.value?.value || ' ',
      extensions,
    }),
    parent: el,
  })

  ipc.on('main-menu:copy-snippet', onCopySnippetMenu)

  watch(selectedSnippetContent, (v) => {
    const scheduledSnippetId = selectedSnippet.value?.id
    const scheduledContentId = v?.id

    nextTick(() => {
      if (
        selectedSnippet.value?.id !== scheduledSnippetId
        || selectedSnippetContent.value?.id !== scheduledContentId
      ) {
        return
      }

      if (selectedSnippet.value && (!v || v.value === undefined)) {
        return
      }

      const isNewValue = v?.id !== lastAppliedContentId
      const isSameContent = v?.id === lastAppliedContentId
      const snippetId = selectedSnippet.value?.id
      const contentId = v?.id
      let nextValue = v?.value || ''

      if (snippetId && contentId) {
        const pendingUpdate = getPendingContentUpdate(snippetId, contentId)
        if (pendingUpdate) {
          nextValue = pendingUpdate.value || ''
        }

        if (
          isSameContent
          && isContentUpdateBusy(snippetId, contentId)
          && view
          && view.state.doc.toString() !== nextValue
        ) {
          return
        }
      }

      setValue(nextValue, true, !isNewValue)
      lastAppliedContentId = contentId

      nextTick(() => {
        if (searchQuery.value) {
          updateSearchOverlay()
        }
      })
    })
  })

  watch(selectedSnippetContent, (v) => {
    const scheduledSnippetId = selectedSnippet.value?.id
    const scheduledContentId = v?.id

    nextTick(() => {
      if (
        !v
        || selectedSnippet.value?.id !== scheduledSnippetId
        || selectedSnippetContent.value?.id !== scheduledContentId
      ) {
        return
      }
      setLanguage(v.language as Language)
    })
  })

  watch(isDark, (dark) => {
    view?.dispatch({
      effects: themeConf.reconfigure(createCodeHighlight(dark)),
    })
  })

  watch(
    () => settings.fontSize,
    () => {
      nextTick(() => {
        view?.requestMeasure()
      })
    },
  )

  watch(
    () => settings.wrap,
    (wrap) => {
      view?.dispatch({
        effects: lineWrappingConf.reconfigure(
          wrap ? EditorView.lineWrapping : [],
        ),
      })
    },
  )

  watch(
    () => settings.highlightLine,
    (highlight) => {
      view?.dispatch({
        effects: highlightLineConf.reconfigure(
          highlight ? highlightActiveLine() : [],
        ),
      })
    },
  )

  watch(
    () => settings.matchBrackets,
    (match) => {
      view?.dispatch({
        effects: matchBracketsConf.reconfigure(match ? matchBrackets() : []),
      })
    },
  )

  watch(
    () => settings.tabSize,
    (tabSize) => {
      const normalizedTabSize = Math.max(1, Number(tabSize) || 1)
      view?.dispatch({
        effects: [
          tabSizeConf.reconfigure(indentUnit.of(' '.repeat(normalizedTabSize))),
        ],
      })
    },
  )

  watch(
    isShowEditor,
    (isVisible, wasVisible) => {
      if (!isVisible || wasVisible !== false)
        return

      nextTick(() => {
        requestAnimationFrame(() => {
          view?.requestMeasure()
        })
      })
    },
    { flush: 'post' },
  )

  watch(searchQuery, () => {
    nextTick(() => {
      updateSearchOverlay()
    })
  })
}

function setValue(value: string, programmatic = true, preserveViewport = true) {
  if (!view)
    return

  const current = view.state.doc.toString()
  if (current === value)
    return

  const { scrollTop, scrollLeft } = view.scrollDOM
  isProgrammaticChange.value = programmatic

  view.dispatch({
    changes: { from: 0, to: view.state.doc.length, insert: value },
  })

  isProgrammaticChange.value = false

  if (preserveViewport) {
    view.scrollDOM.scrollTop = scrollTop
    view.scrollDOM.scrollLeft = scrollLeft
  }
}

async function setLanguage(language: Language) {
  if (!view)
    return
  const langSupport = await loadLanguage(language)
  view.dispatch({
    effects: languageConf.reconfigure(langSupport ? [langSupport] : []),
  })
}

function focusEditor() {
  isShowCodeImage.value = false
  isShowJsonVisualizer.value = false

  nextTick(() => {
    requestAnimationFrame(() => {
      view?.focus()
    })
  })
}

async function format() {
  const availableLang: Language[] = [
    'css',
    'dockerfile',
    'gitignore',
    'graphqlschema',
    'html',
    'ini',
    'jade',
    'java',
    'javascript',
    'json',
    'json5',
    'less',
    'markdown',
    'php',
    'properties',
    'sass',
    'scss',
    'sh',
    'toml',
    'typescript',
    'xml',
    'yaml',
  ]

  if (
    selectedSnippetContent.value?.value
    && !selectedSnippetContent.value?.language
  ) {
    return
  }

  if (
    !availableLang.includes(selectedSnippetContent.value?.language as Language)
  )
    return

  const lang = selectedSnippetContent.value?.language as Language
  const value = view?.state.doc.toString()
  let parser = lang as string

  const shellLike = ['dockerfile', 'gitignore', 'properties', 'ini']

  if (lang === 'javascript')
    parser = 'babel'
  if (lang === 'graphqlschema')
    parser = 'graphql'
  if (shellLike.includes(lang))
    parser = 'sh'

  try {
    const formatted = await ipc.invoke('prettier:format', {
      text: value,
      parser,
    })
    setValue(formatted, false)
  }
  catch (err) {
    console.error(err)
  }
}

function onCopySnippetMenu() {
  const { copy } = useClipboard({ source: view?.state.doc.toString() || '' })
  copy()
  useDonations().incrementCopy('code')
}

function normalizeTerminalOutput() {
  if (!view)
    return

  if (view.state.selection.ranges.some(r => !r.empty)) {
    const changes = view.state.selection.ranges.map((range) => {
      const text = view!.state.sliceDoc(range.from, range.to)
      return {
        from: range.from,
        to: range.to,
        insert: normalizeTerminalText(text),
      }
    })
    view.dispatch({ changes })
    return
  }

  const value = view.state.doc.toString()
  const normalized = normalizeTerminalText(value)

  if (normalized === value)
    return

  const cursorIndex = view.state.selection.main.head
  const mappedIndex = mapNormalizedCursorIndex(value, cursorIndex, normalized)

  view.dispatch({
    changes: { from: 0, to: value.length, insert: normalized },
    selection: { anchor: mappedIndex },
  })
}

ipc.on('main-menu:format', format)
ipc.on('main-menu:normalize-code-line-breaks', normalizeTerminalOutput)

onBeforeUnmount(() => {
  ipc.removeListeners('main-menu:format')
  ipc.removeListeners('main-menu:normalize-code-line-breaks')
  ipc.removeListeners('main-menu:copy-snippet')
  if (view) {
    view.destroy()
    view = null
  }
})

function updateSearchOverlay() {
  if (!view)
    return

  const query = searchQuery.value

  if (query) {
    view.dispatch({
      effects: setSearchQuery.of(
        new SearchQuery({ search: query, caseSensitive: false }),
      ),
    })
  }
  else {
    view.dispatch({
      effects: setSearchQuery.of(new SearchQuery({ search: '' })),
    })
  }
}

onMounted(() => {
  init()
})
</script>

<template>
  <div
    data-editor
    class="relative grid h-full grid-rows-[auto_1fr_auto] overflow-hidden pt-[var(--content-top-offset)]"
  >
    <UiLoadingOverlay
      v-if="selectedSnippet?.pendingCloudDownload"
      :label="i18n.t('cloudDownloads.itemPending')"
    />
    <EditorHeader
      v-if="isShowHeader"
      @focus-editor="focusEditor"
    />
    <div
      v-show="isShowEditor"
      class="flex min-h-0 flex-1 flex-col overflow-auto"
    >
      <div
        id="editor"
        data-editor-mount
        class="min-h-0 flex-1"
      />
      <template v-if="isShowCodePreview">
        <div
          ref="previewHandleRef"
          class="before:bg-border hover:before:bg-primary data-[resizing]:before:bg-primary relative z-10 flex h-px shrink-0 cursor-row-resize items-center justify-center bg-transparent before:absolute before:inset-x-0 before:top-1/2 before:h-px before:-translate-y-1/2 before:transition-[background-color,height] before:duration-150 before:content-[''] after:absolute after:inset-x-0 after:top-1/2 after:h-3 after:-translate-y-1/2 after:content-[''] hover:before:h-0.5 hover:before:delay-200 data-[resizing]:before:h-0.5"
        />
        <div
          :style="{ height: `${previewHeight}px` }"
          class="shrink-0 overflow-hidden"
        >
          <EditorPreview />
        </div>
      </template>
    </div>
    <EditorFooter v-if="isShowEditor" />
    <EditorCodeImage v-if="isShowCodeImage" />
    <EditorJsonVisualizer v-if="isShowJsonVisualizer" />
    <div
      v-if="
        isEmpty
          || selectedSnippetIds.length > 1
          || selectedSnippet === undefined
      "
      class="row-span-full flex items-center justify-center"
    >
      <UiEmptyPlaceholder
        v-if="isEmpty || selectedSnippet === undefined"
        :text="i18n.t('snippet.noSelected')"
      />
      <UiEmptyPlaceholder
        v-if="!isEmpty && selectedSnippetIds.length > 1"
        :text="
          i18n.t('snippet.selectedMultiple', {
            count: selectedSnippetIds.length,
          })
        "
      />
    </div>
  </div>
</template>

<style>
@reference '../../styles.css';

.cm-scroller div {
  opacity: var(--editor-scrollbar-opacity);
  transition: opacity 0.3s;
}

#editor {
  display: flex;
  flex-direction: column;
}

.cm-editor {
  height: 100%;
}
</style>
