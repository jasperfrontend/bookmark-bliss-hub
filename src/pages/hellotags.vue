<template>
  <div>
    <v-card class="pa-6 mb-6 bg-transparent" flat>
      <v-card-title class="text-h4 mb-4 d-flex align-center">
        <v-icon icon="mdi-tag-multiple" class="mr-3" />
        Your saved tags
        <v-spacer />
        <v-chip 
          color="primary" 
          variant="tonal" 
          size="small"
        >
          {{ filteredTags.length }} tag{{ filteredTags.length === 1 ? '' : 's' }}
        </v-chip>
      </v-card-title>
      
      <!-- Search and Filter Controls -->
      <v-card-text class="pb-2">
        <v-row class="mb-4">
          <v-col cols="12" md="8">
            <v-text-field
              v-model="searchQuery"
              label="Search tags..."
              prepend-inner-icon="mdi-magnify"
              variant="outlined"
              density="comfortable"
              clearable
              @keydown.enter="focusFirstTag"
              ref="searchInput"
            />
          </v-col>
          <v-col cols="12" md="4">
            <v-select
              v-model="sortOption"
              :items="sortOptions"
              label="Sort by"
              variant="outlined"
              density="comfortable"
            />
          </v-col>
        </v-row>

        <!-- Quick Filters -->
        <div class="d-flex flex-wrap gap-2 mb-4">
          <v-chip
            v-for="letter in availableLetters"
            :key="letter"
            :variant="selectedLetter === letter ? 'flat' : 'outlined'"
            :color="selectedLetter === letter ? 'primary' : 'default'"
            size="small"
            @click="filterByLetter(letter)"
            class="cursor-pointer"
          >
            {{ letter }}
          </v-chip>
          <v-chip
            :variant="selectedLetter === null ? 'flat' : 'outlined'"
            :color="selectedLetter === null ? 'primary' : 'default'"
            size="small"
            @click="clearLetterFilter"
            class="cursor-pointer"
          >
            All
          </v-chip>
        </div>
      </v-card-text>

      <!-- Tags Display -->
      <v-card-text>
        <div v-if="filteredTags.length === 0 && searchQuery" class="text-center text-grey-darken-1 py-8">
          <v-icon icon="mdi-tag-off-outline" size="48" class="mb-2" />
          <p class="text-h6">No tags found</p>
          <p class="text-caption">Try adjusting your search or filters</p>
        </div>

        <div v-else-if="filteredTags.length === 0" class="text-center text-grey-darken-1 py-8">
          <v-icon icon="mdi-tag-outline" size="48" class="mb-2" />
          <p class="text-h6">No tags yet</p>
          <p class="text-caption">Tags will appear here as you create bookmarks with tags</p>
        </div>

        <!-- Tags Grid -->
        <div v-else class="tags-grid">
          <v-card
            v-for="(tag, index) in paginatedTags"
            :key="tag.id"
            :ref="el => tagRefs[index] = el"
            class="tag-card cursor-pointer"
            variant="outlined"
            hover
            @click="handleSearchTag(tag.title)"
            :tabindex="0"
            @keydown.enter="handleSearchTag(tag.title)"
            @keydown.delete="handleDeleteTag(tag)"
          >
            <v-card-text class="pa-3">
              <div class="d-flex align-center justify-space-between">
                <div class="flex-grow-1 mr-2">
                  <div class="text-subtitle-1 font-weight-medium mb-1">
                    {{ tag.title }}
                  </div>
                  <div class="text-caption text-grey-darken-1">
                    {{ tag.usage_count || 0 }} bookmark{{ (tag.usage_count || 0) === 1 ? '' : 's' }}
                  </div>
                </div>
                <div class="d-flex align-center">
                  <v-btn
                    @click.stop="handleDeleteTag(tag)"
                    icon="mdi-delete"
                    variant="text"
                    size="small"
                    color="error"
                    :title="`Delete tag: ${tag.title}`"
                  />
                </div>
              </div>
            </v-card-text>
          </v-card>
        </div>

        <!-- Pagination -->
        <div v-if="totalPages > 1" class="d-flex justify-center mt-6">
          <v-pagination
            v-model="currentPage"
            :length="totalPages"
            :total-visible="7"
            color="primary"
          />
        </div>
      </v-card-text>
    </v-card>

    
    <!-- Clean Unused Tags Section -->
    <v-card class="pa-6 mb-6" flat>
      <v-card-title class="text-h5 mb-4 d-flex align-center">
        <v-icon icon="mdi-broom" class="mr-3" />
        Maintenance
      </v-card-title>
      <v-card-text>
        <p class="mb-4">
          Due to their shared nature, tags do not get removed when you delete a bookmark. 
          You can clean up unused tags that are no longer associated with any bookmarks.
        </p>
        <v-btn 
          @click="cleanupDialog = true" 
          color="warning"
          variant="flat"
          prepend-icon="mdi-delete-sweep"
        >
          Clean unused tags
        </v-btn>
      </v-card-text>
    </v-card>

    <!-- Confirmation Dialogs -->
    <v-dialog v-model="cleanupDialog" max-width="400" persistent>
      <v-card
        prepend-icon="mdi-alert"
        title="Confirm removal of unused tags"
        text="Are you sure you want to clean unused tags? This will permanently remove tags that are not associated with any bookmarks."
      >
        <template v-slot:actions>
          <v-spacer />
          <v-btn 
            @click="cleanupDialog = false"
            variant="text"
            :disabled="deleting"
          >
            Cancel
          </v-btn>
          <v-btn 
            @click="deleteUnusedTags(); cleanupDialog = false" 
            color="warning"
            variant="flat"
            :loading="deleting"
          >
            Remove unused tags
          </v-btn>
        </template>
      </v-card>
    </v-dialog>

    <v-dialog v-model="confirmDialog.show" max-width="400" persistent>
      <v-card>
        <v-card-title class="d-flex align-center">
          <v-icon icon="mdi-delete" class="mr-2" color="error" />
          Confirm Deletion
        </v-card-title>
        
        <v-card-text>
          Are you sure you want to delete this tag?
          <v-code class="d-block mt-2 pa-2">{{ confirmDialog.tagTitle }}</v-code>
          <v-alert type="warning" variant="tonal" class="mt-3">
            This will remove the tag from all your bookmarks that use it.
          </v-alert>
        </v-card-text>
        
        <v-card-actions>
          <v-spacer />
          <v-btn 
            variant="text" 
            @click="confirmDialog.show = false"
            :disabled="deleting"
          >
            Cancel
          </v-btn>
          <v-btn 
            color="error" 
            variant="flat"
            @click="confirmDeleteTag"
            :loading="deleting"
          >
            Delete
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Notification Snackbar -->
    <v-snackbar
      v-model="notification.show"
      :color="notification.type"
      :timeout="3000"
      location="bottom right"
    >
      <div class="d-flex align-center">
        <v-icon 
          :icon="notification.type === 'success' ? 'mdi-check-circle' : 'mdi-alert-circle'" 
          class="me-2" 
        />
        {{ notification.message }}
      </div>
    </v-snackbar>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import { useRouter } from 'vue-router'
import supabase from '@/lib/supabaseClient'

const router = useRouter()

const tagsData = ref([])
const searchQuery = ref('')
const selectedLetter = ref(null)
const sortOption = ref('alphabetical')
const currentPage = ref(1)
const itemsPerPage = 24
const deleting = ref(false)
const cleanupDialog = ref(false)
const searchInput = ref(null)
const tagRefs = ref([])

// Sort options
const sortOptions = [
  { title: 'Alphabetical (A-Z)', value: 'alphabetical' },
  { title: 'Alphabetical (Z-A)', value: 'alphabetical-desc' },
  { title: 'Most Used', value: 'usage-desc' },
  { title: 'Least Used', value: 'usage-asc' },
  { title: 'Recently Created', value: 'created-desc' },
  { title: 'Oldest First', value: 'created-asc' }
]

// Dialog states
const confirmDialog = ref({
  show: false,
  tag: null,
  tagTitle: ''
})

// Notification state
const notification = ref({
  show: false,
  type: 'success',
  message: ''
})

// Computed properties
const filteredTags = computed(() => {
  let tags = [...tagsData.value]

  // Apply search filter
  if (searchQuery.value) {
    const query = searchQuery.value.toLowerCase().trim()
    tags = tags.filter(tag => 
      tag.title.toLowerCase().includes(query)
    )
  }

  // Apply letter filter
  if (selectedLetter.value) {
    tags = tags.filter(tag => 
      tag.title.toLowerCase().startsWith(selectedLetter.value.toLowerCase())
    )
  }

  // Apply sorting
  switch (sortOption.value) {
    case 'alphabetical':
      tags.sort((a, b) => a.title.localeCompare(b.title))
      break
    case 'alphabetical-desc':
      tags.sort((a, b) => b.title.localeCompare(a.title))
      break
    case 'usage-desc':
      tags.sort((a, b) => (b.usage_count || 0) - (a.usage_count || 0))
      break
    case 'usage-asc':
      tags.sort((a, b) => (a.usage_count || 0) - (b.usage_count || 0))
      break
    case 'created-desc':
      tags.sort((a, b) => new Date(b.created_at || 0) - new Date(a.created_at || 0))
      break
    case 'created-asc':
      tags.sort((a, b) => new Date(a.created_at || 0) - new Date(b.created_at || 0))
      break
  }

  return tags
})

const paginatedTags = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage
  const end = start + itemsPerPage
  return filteredTags.value.slice(start, end)
})

const totalPages = computed(() => {
  return Math.ceil(filteredTags.value.length / itemsPerPage)
})

const availableLetters = computed(() => {
  const letters = new Set()
  tagsData.value.forEach(tag => {
    const firstLetter = tag.title.charAt(0).toUpperCase()
    if (firstLetter.match(/[A-Z]/)) {
      letters.add(firstLetter)
    }
  })
  return Array.from(letters).sort()
})

// Methods
function showNotification(type, message) {
  notification.value = {
    show: true,
    type,
    message
  }
}

function handleSearchTag(tag) {
  router.push(`/tag/${encodeURIComponent(tag)}`)
}

function handleDeleteTag(tag) {
  confirmDialog.value = {
    show: true,
    tag: tag,
    tagTitle: tag.title
  }
}

function filterByLetter(letter) {
  selectedLetter.value = selectedLetter.value === letter ? null : letter
  currentPage.value = 1
}

function clearLetterFilter() {
  selectedLetter.value = null
  currentPage.value = 1
}

function focusFirstTag() {
  nextTick(() => {
    if (tagRefs.value[0]) {
      tagRefs.value[0].$el.focus()
    }
  })
}

async function getTags() {
  try {
    const { data: { session } } = await supabase.auth.getSession()
    
    if (!session?.user) {
      showNotification('error', 'You must be logged in to view tags')
      return
    }
    
    const userId = session.user.id
    
    // Get tags with usage count by joining with bookmark_tags
    const { data, error } = await supabase
      .from('tags')
      .select(`
        id,
        title,
        created_at,
        bookmark_tags (
          bookmark_id,
          bookmarks!inner (
            user_id
          )
        )
      `)
      .eq('user_id', userId)
      .eq('bookmark_tags.bookmarks.user_id', userId)
      .order('title', { ascending: true })

    if (error) {
      console.error('Error retrieving tags:', error)
      showNotification('error', 'Error retrieving tags!')
      return
    }

    // Calculate usage count for each tag
    tagsData.value = (data || []).map(tag => ({
      ...tag,
      usage_count: tag.bookmark_tags?.length || 0
    }))

  } catch (error) {
    console.error('Error in getTags:', error)
    showNotification('error', 'Error retrieving tags!')
  }
}

async function confirmDeleteTag() {
  if (!confirmDialog.value.tag) return
  
  deleting.value = true
  
  try {
    const { data: { session } } = await supabase.auth.getSession()
    
    if (!session?.user) {
      showNotification('error', 'You must be logged in to delete tags')
      return
    }
    
    const userId = session.user.id
    const tagToDelete = confirmDialog.value.tag
    
    // First, get all bookmark IDs for the current user
    const { data: userBookmarks, error: bookmarkFetchError } = await supabase
      .from('bookmarks')
      .select('id')
      .eq('user_id', userId)
    
    if (bookmarkFetchError) {
      console.error('Error fetching user bookmarks:', bookmarkFetchError)
      showNotification('error', 'Failed to fetch user bookmarks')
      return
    }
    
    const bookmarkIds = userBookmarks.map(bookmark => bookmark.id)
    
    // Delete bookmark_tags relationships for this tag and user's bookmarks
    if (bookmarkIds.length > 0) {
      const { error: relationshipError } = await supabase
        .from('bookmark_tags')
        .delete()
        .eq('tag_id', tagToDelete.id)
        .in('bookmark_id', bookmarkIds)
      
      if (relationshipError) {
        console.error('Error deleting tag relationships:', relationshipError)
        showNotification('error', 'Failed to delete tag relationships')
        return
      }
    }
    
    // Then delete the tag itself
    const { error: tagError } = await supabase
      .from('tags')
      .delete()
      .eq('id', tagToDelete.id)
      .eq('user_id', userId)
    
    if (tagError) {
      console.error('Error deleting tag:', tagError)
      showNotification('error', 'Failed to delete tag')
      return
    }
    
    // Remove from local state
    tagsData.value = tagsData.value.filter(tag => tag.id !== tagToDelete.id)
    
    showNotification('success', `Deleted tag: ${tagToDelete.title}`)
    
  } catch (error) {
    console.error('Error deleting tag:', error)
    showNotification('error', 'Failed to delete tag')
  } finally {
    deleting.value = false
    confirmDialog.value.show = false
    confirmDialog.value.tag = null
    confirmDialog.value.tagTitle = ''
  }
}

async function deleteUnusedTags() {
  deleting.value = true
  
  try {
    const { data, error } = await supabase.rpc('delete_unused_tags')
    const count = data ?? 0

    if (error) {
      console.error('Could not clean orphaned tags:', error.message)
      showNotification('error', 'Error deleting unused tags.')
    } else {
      showNotification('success', `${count} unused tag${count !== 1 ? 's' : ''} removed.`)
      
      // Refresh tags after cleanup
      setTimeout(() => {
        getTags()
      }, 1000)
    }
  } catch (error) {
    console.error('Error in deleteUnusedTags:', error)
    showNotification('error', 'Error deleting unused tags.')
  } finally {
    deleting.value = false
  }
}

// Watch for filter changes to reset pagination
watch([searchQuery, selectedLetter, sortOption], () => {
  currentPage.value = 1
})

// Keyboard shortcuts
function handleKeydown(event) {
  // Ctrl/Cmd + F to focus search
  if ((event.ctrlKey || event.metaKey) && event.key === 'f') {
    event.preventDefault()
    searchInput.value?.focus()
  }
}

onMounted(() => {
  getTags()
  document.addEventListener('keydown', handleKeydown)
})

// Cleanup event listener
onUnmounted(() => {
  document.removeEventListener('keydown', handleKeydown)
})
</script>

<style scoped>
.tags-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 16px;
}

.tag-card {
  transition: transform 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
}

.tag-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.tag-card:focus {
  outline: 2px solid rgb(var(--v-theme-primary));
  outline-offset: 2px;
}

.cursor-pointer {
  cursor: pointer;
}

.gap-2 {
  gap: 8px;
}

/* Responsive adjustments */
@media (max-width: 960px) {
  .tags-grid {
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
    gap: 12px;
  }
}

@media (max-width: 600px) {
  .tags-grid {
    grid-template-columns: 1fr;
    gap: 8px;
  }
}
</style>

<route lang="yaml">
meta:
  layout: contentpage
</route>