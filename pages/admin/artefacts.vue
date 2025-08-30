<template>
  <div class="space-y-6">
    <!-- Header -->
    <ArtefactsHeader @upload="showUploadModal = true" />

    <!-- Stats Cards -->
    <ArtefactsStats 
      :total-artefacts="totalArtefacts"
      :processed-artefacts="processedArtefacts"
      :total-categories="totalCategories"
      :total-size="totalSize"
    />

    <!-- Search and Filters -->
    <ArtefactsFilters
      v-model:search-query="searchQuery"
      v-model:selected-category="selectedCategory"
      v-model:selected-type="selectedType"
      v-model:selected-status="selectedStatus"
      :available-categories="availableCategories"
    />

    <!-- Artefacts Table -->
    <ArtefactsTable
      :artefacts="filteredArtefacts"
      @view-artefact="viewArtefact"
      @download-artefact="downloadArtefact"
      @delete-artefact="deleteArtefact"
      @view-summary="viewSummary"
    />

    <!-- Upload Modal -->
    <ArtefactUploadModal
      v-model:is-open="showUploadModal"
      :available-categories="availableCategories"
      @close="showUploadModal = false"
      @file-uploaded="handleFileUploaded"
      @google-drive-uploaded="handleGoogleDriveUploaded"
      @category-added="addCategory"
      @category-deleted="deleteCategory"
    />

    <!-- Summary Modal -->
    <ArtefactSummaryModal
      v-model:is-open="showSummaryModal"
      :artefact="selectedArtefact"
      @close="showSummaryModal = false"
      @download="downloadArtefact"
    />
  </div>
</template>

<script setup lang="ts">
import { formatDateTime } from '~/utils'

// Using admin layout
definePageMeta({
  layout: 'admin',
  middleware: 'auth',
})

// Import components
import ArtefactsHeader from '~/components/admin/artefacts/ArtefactsHeader.vue'
import ArtefactsStats from '~/components/admin/artefacts/ArtefactsStats.vue'
import ArtefactsFilters from '~/components/admin/artefacts/ArtefactsFilters.vue'
import ArtefactsTable from '~/components/admin/artefacts/ArtefactsTable.vue'
import ArtefactUploadModal from '~/components/admin/artefacts/ArtefactUploadModal.vue'
import ArtefactSummaryModal from '~/components/admin/artefacts/ArtefactSummaryModal.vue'

// Reactive data
const searchQuery = ref('')
const selectedCategory = ref('')
const selectedType = ref('')
const selectedStatus = ref('')
const showUploadModal = ref(false)
const showSummaryModal = ref(false)
const selectedArtefact = ref(null)

// Categories management
const availableCategories = ref([
  'HR Policy',
  'Financial',
  'Technical',
  'Analytics',
  'Legal & Compliance',
  'Policies & Procedures',
  'Product / Service Information',
  'Technical / Operational Documentation',
  'Training & Onboarding'
])

// Sample artefacts data
const artefacts = ref([
  {
    id: 1,
    name: 'Employee Handbook 2024.pdf',
    description: 'Comprehensive guide to company policies',
    category: 'HR Policy',
    type: 'PDF',
    size: '2.3 MB',
    status: 'processed',
    uploadedBy: 'Sarah Johnson',
    lastUpdated: formatDateTime(new Date('2024-01-15T14:30:00')),
    artefact: 'Employee Handbook 2024.pdf',
  },
  {
    id: 2,
    name: 'Q4 Financial Report.docx',
    description: 'Quarterly financial reports including revenue, expenses',
    category: 'Financial',
    type: 'Word',
    size: '1.6 MB',
    status: 'processing',
    uploadedBy: 'Mike Chen',
    lastUpdated: formatDateTime(new Date('2024-01-10T09:15:00')),
    artefact: 'Q4 Financial Report.docx',
  },
  {
    id: 3,
    name: 'Product Specifications.md',
    description: 'Detailed technical specifications for the new product',
    category: 'Technical',
    type: 'Markdown',
    size: '512.0 kB',
    status: 'processed',
    uploadedBy: 'Emily Davis',
    lastUpdated: formatDateTime(new Date('2024-01-08T16:45:00')),
    artefact: 'Product Specifications.md',
  },
  {
    id: 4,
    name: 'Customer Data.csv',
    description: 'Customer demographics and behavior analysis data',
    category: 'Analytics',
    type: 'CSV',
    size: '3.1 MB',
    status: 'processed',
    uploadedBy: 'Alex Rodriguez',
    lastUpdated: formatDateTime(new Date('2024-01-20T11:20:00')),
    artefact: 'Customer Data.csv',
  },
])

// Computed properties
const totalArtefacts = computed(() => artefacts.value.length)
const processedArtefacts = computed(
  () => artefacts.value.filter((doc) => doc.status === 'processed').length,
)
const totalCategories = computed(() => {
  const categories = new Set(artefacts.value.map((doc) => doc.category))
  return categories.size
})
const totalSize = computed(() => '7.8 MB') // This would be calculated from actual file sizes

const filteredArtefacts = computed(() => {
  return artefacts.value.filter((artefact) => {
    const matchesSearch =
      !searchQuery.value ||
      artefact.name.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      artefact.description.toLowerCase().includes(searchQuery.value.toLowerCase())

    const matchesCategory = !selectedCategory.value || artefact.category === selectedCategory.value
    const matchesType = !selectedType.value || artefact.type === selectedType.value
    const matchesStatus = !selectedStatus.value || artefact.status === selectedStatus.value

    return matchesSearch && matchesCategory && matchesType && matchesStatus
  })
})

// Methods
const viewArtefact = (artefact: any) => {
  console.log('View artefact:', artefact)
}

const downloadArtefact = (artefact: any) => {
  console.log('Download artefact:', artefact)
}

const deleteArtefact = (artefact: any) => {
  if (confirm(`Are you sure you want to delete ${artefact.name}?`)) {
    const index = artefacts.value.findIndex((d) => d.id === artefact.id)
    if (index > -1) {
      artefacts.value.splice(index, 1)
    }
  }
}

const viewSummary = (artefact: any) => {
  selectedArtefact.value = artefact
  showSummaryModal.value = true
}

// Upload handlers
const handleFileUploaded = (artefact: any) => {
  artefacts.value.unshift(artefact)
  
  // After 2 seconds, mark as processed
  setTimeout(() => {
    const uploadedArtefact = artefacts.value.find(a => a.id === artefact.id)
    if (uploadedArtefact) {
      uploadedArtefact.status = 'processed'
    }
  }, 2000)
}

const handleGoogleDriveUploaded = (newArtefacts: any[]) => {
  newArtefacts.forEach(artefact => {
    artefacts.value.unshift(artefact)
    
    // After 2 seconds, mark as processed
    setTimeout(() => {
      const uploadedArtefact = artefacts.value.find(a => a.id === artefact.id)
      if (uploadedArtefact) {
        uploadedArtefact.status = 'processed'
      }
    }, 2000)
  })
}

// Category management methods
const addCategory = (category: string) => {
  const trimmedCategory = category.trim()
  if (trimmedCategory && !availableCategories.value.includes(trimmedCategory)) {
    availableCategories.value.push(trimmedCategory)
  }
}

const deleteCategory = (category: string) => {
  const index = availableCategories.value.indexOf(category)
  if (index > -1) {
    availableCategories.value.splice(index, 1)

    // Update any existing artefacts that use this category
    artefacts.value.forEach(artefact => {
      if (artefact.category === category) {
        artefact.category = 'Uncategorized'
      }
    })
  }
}

useHead({
  title: 'Artefact Management - Admin Dashboard',
})
</script>
