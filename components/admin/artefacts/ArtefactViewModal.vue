<template>
  <UModal :model-value="isOpen" @update:model-value="$emit('update:isOpen', $event)" fullscreen>
    <div class="flex flex-col h-screen bg-white dark:bg-gray-900">
      <!-- Header -->
      <div class="flex-shrink-0 px-6 py-4 border-b border-gray-200 dark:border-gray-700">
        <div class="flex items-center justify-between">
          <div>
            <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
              {{ artefact?.name || 'Document Viewer' }}
            </h3>
            <p class="text-sm text-gray-500 dark:text-gray-400" v-if="artefact">
              {{ artefact.type }} • {{ artefact.category }} • {{ artefact.size }}
            </p>
          </div>
          <UButton
            color="gray"
            variant="ghost"
            icon="i-heroicons-x-mark-20-solid"
            class="-my-1"
            @click="closeModal"
          />
        </div>
      </div>

      <!-- Content Area -->
      <div class="flex-1 overflow-hidden">
        <!-- Loading State -->
        <div v-if="isLoading" class="flex items-center justify-center h-full">
          <div class="text-center">
            <UIcon
              name="i-heroicons-arrow-path"
              class="w-8 h-8 animate-spin text-primary-500 mx-auto mb-4"
            />
            <p class="text-gray-600 dark:text-gray-400">Loading document...</p>
          </div>
        </div>

        <!-- Error State -->
        <div v-else-if="error" class="flex items-center justify-center h-full">
          <div class="text-center">
            <UIcon
              name="i-heroicons-exclamation-triangle"
              class="w-8 h-8 text-red-500 mx-auto mb-4"
            />
            <p class="text-red-600 dark:text-red-400 mb-4">{{ error }}</p>
            <UButton @click="retry" color="primary" variant="outline"> Try Again </UButton>
          </div>
        </div>

        <!-- Document Content -->
        <div v-else-if="viewContent" class="h-full overflow-auto">
          <div v-html="viewContent" class="prose prose-sm max-w-none dark:prose-invert"></div>
        </div>

        <!-- Fallback Download -->
        <div v-else class="flex items-center justify-center h-full">
          <div class="text-center">
            <UIcon name="i-heroicons-document" class="w-8 h-8 text-gray-400 mx-auto mb-4" />
            <p class="text-gray-600 dark:text-gray-400 mb-4">Unable to preview this file type.</p>
            <UButton
              v-if="fileUrl"
              :href="fileUrl"
              target="_blank"
              color="primary"
              icon="i-heroicons-arrow-down-tray"
            >
              Download File
            </UButton>
          </div>
        </div>
      </div>

      <!-- Footer -->
      <div class="flex-shrink-0 px-6 py-4 border-t border-gray-200 dark:border-gray-700">
        <div class="flex justify-between items-center">
          <div class="flex space-x-2">
            <UButton
              v-if="fileUrl"
              :href="fileUrl"
              target="_blank"
              color="gray"
              variant="outline"
              icon="i-heroicons-arrow-down-tray"
              size="sm"
            >
              Download
            </UButton>
          </div>
          <UButton @click="closeModal" color="gray" variant="outline"> Close </UButton>
        </div>
      </div>
    </div>
  </UModal>
</template>

<script setup lang="ts">
import { useArtefactsStore } from '~/stores/artefacts'
import { marked } from 'marked'

interface Props {
  isOpen: boolean
  artefact: any | null
}

interface Emits {
  (e: 'update:isOpen', value: boolean): void
  (e: 'close'): void
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

// Reactive data
const isLoading = ref(false)
const error = ref<string | null>(null)
const viewContent = ref<string | null>(null)
const fileUrl = ref<string | null>(null)
const fileType = ref<string | null>(null)

// Import stores and composables
const artefactsStore = useArtefactsStore()
const { showError } = useNotification()

// Helper function to clear content
const clearContent = () => {
  viewContent.value = null
  fileUrl.value = null
  fileType.value = null
  error.value = null
  isLoading.value = false
}

// Watch for both artefact and modal state changes
watch(
  () => [props.artefact, props.isOpen],
  async ([newArtefact, isOpen]) => {
    if (isOpen && newArtefact) {
      await loadDocument()
    } else if (!isOpen) {
      // Clear content when modal closes
      clearContent()
    }
  },
  { immediate: true },
)

const loadDocument = async () => {
  if (!props.artefact) return

  isLoading.value = true
  error.value = null
  viewContent.value = null
  fileUrl.value = null

  try {
    const result = await artefactsStore.viewArtefact(props.artefact.id)

    if (result.success && result.data) {
      fileUrl.value = result.data.fileUrl
      fileType.value = result.data.fileType

      // Render content based on file type
      await renderContentByFileType(result.data.fileType, result.data.fileUrl)
    } else {
      error.value = result.message || 'Failed to load document'
    }
  } catch (err: any) {
    error.value = err.message || 'Failed to load document'
    showError(error.value)
  } finally {
    isLoading.value = false
  }
}

const renderContentByFileType = async (type: string, url: string) => {
  try {
    if (type === 'url') {
      viewContent.value = `<p>Unable to preview this URL. <a href="${url}" target="_blank" class="text-primary-400 hover:text-primary-300">Open URL in new tab</a>.</p>`
      return
    }

    // For PDFs and images, use the signed URL directly
    if (type === 'pdf') {
      viewContent.value = `<iframe src="${url}" class="w-full h-full border-0" style="height: calc(100vh - 180px);"></iframe>`
      return
    }

    if (['jpg', 'jpeg', 'png', 'gif', 'webp'].includes(type)) {
      viewContent.value = `<div class="flex items-center justify-center h-full" style="height: calc(100vh - 180px);"><img src="${url}" class="max-w-full max-h-full object-contain rounded-md shadow-lg" alt="Document image" /></div>`
      return
    }

    // For other file types, fetch content through proxy
    const response = await fetch(`/api/artefacts/proxy?fileUrl=${encodeURIComponent(url)}`)

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}))
      throw new Error(errorData.error || `HTTP ${response.status}: ${response.statusText}`)
    }

    const responseData = await response.json()

    if (responseData.error) {
      throw new Error(responseData.error)
    }

    const base64Data = responseData.data
    const decodedData = atob(base64Data)

    switch (type) {
      case 'txt':
        viewContent.value = `<div class="h-full overflow-y-auto p-4"><pre style="white-space: pre-wrap; padding: 1rem; background: white; border-radius: 0.375rem; font-family: 'Courier New', monospace; min-height: calc(100vh - 220px); color: #1f2937;">${escapeHtml(decodedData)}</pre></div>`
        break

      case 'md':
        // Configure and render markdown as HTML using marked
        configureMarked()
        const renderedMarkdown = marked.parse(decodedData)
        viewContent.value = `<div class="h-full overflow-y-auto p-4" style="background: white; color: #1f2937; line-height: 1.6; min-height: calc(100vh - 220px);">${renderedMarkdown}</div>`
        break

      case 'csv':
        // Basic CSV display - you can enhance this with proper table rendering
        const csvLines = decodedData.split('\n').slice(0, 100) // Show first 100 lines for fullscreen
        const csvTable = csvLines
          .map(
            (line) =>
              `<tr>${line
                .split(',')
                .map((cell) => `<td class="border px-2 py-1">${escapeHtml(cell.trim())}</td>`)
                .join('')}</tr>`,
          )
          .join('')
        viewContent.value = `
          <div class="h-full overflow-auto p-4" style="height: calc(100vh - 180px); color: #1f2937;">
            <table class="min-w-full border-collapse border border-gray-300" style="color: #1f2937;">
              ${csvTable}
            </table>
            ${csvLines.length === 100 ? '<p class="text-sm mt-2" style="color: #6b7280;">Showing first 100 rows...</p>' : ''}
          </div>`
        break

      case 'json':
        try {
          const jsonData = JSON.parse(decodedData)
          viewContent.value = `<div class="h-full overflow-y-auto p-4"><pre style="white-space: pre-wrap; padding: 1rem; background: white; border-radius: 0.375rem; font-family: 'Courier New', monospace; min-height: calc(100vh - 220px); color: #1f2937;">${escapeHtml(JSON.stringify(jsonData, null, 2))}</pre></div>`
        } catch {
          viewContent.value = `<div class="h-full overflow-y-auto p-4"><pre style="white-space: pre-wrap; padding: 1rem; background: white; border-radius: 0.375rem; font-family: 'Courier New', monospace; min-height: calc(100vh - 220px); color: #1f2937;">${escapeHtml(decodedData)}</pre></div>`
        }
        break

      case 'html':
        // For security, we'll show HTML as text
        viewContent.value = `<div class="h-full overflow-y-auto p-4"><pre style="white-space: pre-wrap; padding: 1rem; background: white; border-radius: 0.375rem; font-family: 'Courier New', monospace; min-height: calc(100vh - 220px); color: #1f2937;">${escapeHtml(decodedData)}</pre></div>`
        break

      default:
        // For other file types, show download option
        viewContent.value = `<div class="flex items-center justify-center h-full"><p>Unable to preview this file type (${type}). <a href="${url}" download class="text-primary-400 hover:text-primary-300">Download the file</a> to view it.</p></div>`
    }
  } catch (err: any) {
    console.error('Error rendering content:', err)
    viewContent.value = `<div class="flex items-center justify-center h-full"><p>Failed to load the file content. <a href="${url}" download class="text-primary-400 hover:text-primary-300">Download the file</a> to view it.</p></div>`
  }
}

const escapeHtml = (text: string): string => {
  const div = document.createElement('div')
  div.textContent = text
  return div.innerHTML
}

// Configure marked with custom renderer for styling
const configureMarked = () => {
  const renderer = new marked.Renderer()

  // Custom heading styles
  renderer.heading = (text: string, level: number) => {
    const sizes = ['1.875rem', '1.5rem', '1.25rem', '1.125rem', '1rem', '0.875rem']
    const margins = ['2rem 0 1rem 0', '1.5rem 0 0.75rem 0', '1rem 0 0.5rem 0', '1rem 0 0.5rem 0', '0.75rem 0 0.5rem 0', '0.75rem 0 0.5rem 0']

    return `<h${level} style="font-size: ${sizes[level-1]}; font-weight: 600; margin: ${margins[level-1]}; color: #1f2937;">${text}</h${level}>`
  }

  // Custom paragraph styles
  renderer.paragraph = (text: string) => {
    return `<p style="margin: 1rem 0; color: #1f2937; line-height: 1.6;">${text}</p>`
  }

  // Custom code block styles
  renderer.code = (code: string, language: string | undefined) => {
    return `<pre style="background: #f3f4f6; padding: 1rem; border-radius: 0.375rem; margin: 1rem 0; overflow-x: auto; color: #1f2937;"><code>${code}</code></pre>`
  }

  // Custom inline code styles
  renderer.codespan = (code: string) => {
    return `<code style="background: #f3f4f6; padding: 0.25rem 0.5rem; border-radius: 0.25rem; font-family: monospace; color: #1f2937;">${code}</code>`
  }

  // Custom blockquote styles
  renderer.blockquote = (quote: string) => {
    return `<blockquote style="border-left: 4px solid #e5e7eb; padding-left: 1rem; margin: 1rem 0; font-style: italic; color: #6b7280;">${quote}</blockquote>`
  }

  // Custom list styles
  renderer.list = (body: string, ordered: boolean) => {
    const type = ordered ? 'ol' : 'ul'
    const style = ordered ? 'list-style-type: decimal;' : 'list-style-type: disc;'
    return `<${type} style="margin: 1rem 0; padding-left: 1.5rem; ${style}">${body}</${type}>`
  }

  renderer.listitem = (text: string) => {
    return `<li style="margin: 0.25rem 0; color: #1f2937;">${text}</li>`
  }

  // Custom link styles
  renderer.link = (href: string, title: string | null, text: string) => {
    return `<a href="${href}" ${title ? `title="${title}"` : ''} style="color: #3b82f6; text-decoration: underline;" target="_blank">${text}</a>`
  }

  // Custom horizontal rule
  renderer.hr = () => {
    return '<hr style="border: none; border-top: 1px solid #e5e7eb; margin: 2rem 0;">'
  }

  marked.setOptions({
    renderer,
    gfm: true,
    breaks: false,
    sanitize: false
  })
}

const retry = async () => {
  await loadDocument()
}

const closeModal = () => {
  emit('update:isOpen', false)
  emit('close')
  clearContent()
}
</script>

<style scoped>
/* Custom styles for document content */
.prose {
  color: #1f2937;
}

.prose pre {
  background-color: white;
  border-radius: 0.375rem;
  padding: 1rem;
  overflow-x: auto;
  color: #1f2937;
}

.prose table {
  font-size: 0.875rem;
}

.prose td {
  padding: 0.25rem 0.5rem;
  border: 1px solid #e5e7eb;
}
</style>
