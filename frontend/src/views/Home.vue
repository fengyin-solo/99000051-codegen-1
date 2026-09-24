<template>
  <div class="home-page">
    <div class="home-container">
      <div class="home-header">
        <h1>My Boards</h1>
        <el-button type="primary" :icon="Plus" @click="showCreateDialog = true">
          New Board
        </el-button>
      </div>

      <div v-if="boardStore.loading" class="loading-state">
        <el-icon class="is-loading" :size="32"><Loading /></el-icon>
        <p>Loading boards...</p>
      </div>

      <div v-else-if="boardStore.boards.length === 0" class="empty-state">
        <el-empty description="No boards yet. Create your first board!">
          <el-button type="primary" @click="showCreateDialog = true">Create Board</el-button>
        </el-empty>
      </div>

      <template v-else>
        <!-- Overview: board count and total column/card scale across all boards -->
        <div class="overview-stats">
          <div class="stat-item">
            <span class="stat-value">{{ boardStore.boards.length }}</span>
            <span class="stat-label">{{ boardStore.boards.length === 1 ? 'Board' : 'Boards' }}</span>
          </div>
          <el-divider direction="vertical" />
          <div class="stat-item">
            <span class="stat-value">{{ totalColumns }}</span>
            <span class="stat-label">Columns</span>
          </div>
          <el-divider direction="vertical" />
          <div class="stat-item">
            <span class="stat-value">{{ totalCards }}</span>
            <span class="stat-label">Cards</span>
          </div>
        </div>

        <!-- Search / sort / filter toolbar -->
        <div class="boards-toolbar">
          <el-input
            v-model="searchQuery"
            class="toolbar-search"
            placeholder="Search boards by name or description"
            clearable
            :prefix-icon="Search"
          />
          <el-select v-model="sortKey" class="toolbar-select">
            <template #prefix>
              <el-icon><SortIcon /></el-icon>
            </template>
            <el-option
              v-for="option in sortOptions"
              :key="option.value"
              :label="option.label"
              :value="option.value"
            />
          </el-select>
          <el-select v-model="filterKey" class="toolbar-select">
            <template #prefix>
              <el-icon><Filter /></el-icon>
            </template>
            <el-option
              v-for="option in filterOptions"
              :key="option.value"
              :label="option.label"
              :value="option.value"
            />
          </el-select>
        </div>

        <p v-if="isFiltering" class="result-count">
          Showing {{ filteredBoards.length }} of {{ boardStore.boards.length }} boards
        </p>

        <div v-if="filteredBoards.length > 0" class="boards-grid">
          <BoardCard
            v-for="board in filteredBoards"
            :key="board.id"
            :board="board"
            @open="openBoard"
            @delete="confirmDeleteBoard"
          />
        </div>

        <div v-else class="empty-state">
          <el-empty description="No boards match your search or filter.">
            <el-button type="primary" @click="resetView">Clear search &amp; filters</el-button>
          </el-empty>
        </div>
      </template>
    </div>

    <!-- Create Board Dialog -->
    <el-dialog v-model="showCreateDialog" title="Create New Board" width="460px" :close-on-click-modal="false">
      <el-form ref="createFormRef" :model="createForm" :rules="createRules" label-position="top">
        <el-form-item label="Board Name" prop="name">
          <el-input v-model="createForm.name" placeholder="Enter board name" maxlength="50" show-word-limit />
        </el-form-item>
        <el-form-item label="Description (optional)" prop="description">
          <el-input v-model="createForm.description" type="textarea" :rows="3" placeholder="Enter board description" maxlength="200" show-word-limit />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showCreateDialog = false">Cancel</el-button>
        <el-button type="primary" :loading="creating" @click="handleCreateBoard">Create</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Plus, Loading, Search, Filter, Sort as SortIcon } from '@element-plus/icons-vue'
import { useBoardStore } from '../stores/board.js'
import BoardCard from '../components/BoardCard.vue'

const router = useRouter()
const boardStore = useBoardStore()

const showCreateDialog = ref(false)
const creating = ref(false)
const createFormRef = ref(null)

const createForm = ref({ name: '', description: '' })
const createRules = {
  name: [{ required: true, message: 'Board name is required', trigger: 'blur' }]
}

// --- Search / sort / filter state ------------------------------------------------
// Newly created boards may not yet carry count fields; always normalise to 0 so
// sorting, filtering and stats stay correct before the next refetch.
function cardCountOf(board) {
  return board.card_count ?? 0
}
function columnCountOf(board) {
  return board.column_count ?? 0
}

const searchQuery = ref('')
const sortKey = ref('newest')
const filterKey = ref('all')

const sortOptions = [
  { value: 'newest', label: 'Recently created' },
  { value: 'oldest', label: 'Oldest first' },
  { value: 'name', label: 'Name (A–Z)' },
  { value: 'cards', label: 'Most cards' },
  { value: 'columns', label: 'Most columns' }
]

const filterOptions = [
  { value: 'all', label: 'All boards' },
  { value: 'with-cards', label: 'With cards' },
  { value: 'empty', label: 'No cards' }
]

const isFiltering = computed(
  () => searchQuery.value.trim() !== '' || filterKey.value !== 'all'
)

const totalColumns = computed(() =>
  boardStore.boards.reduce((sum, board) => sum + columnCountOf(board), 0)
)
const totalCards = computed(() =>
  boardStore.boards.reduce((sum, board) => sum + cardCountOf(board), 0)
)

// created_at is stored as 'YYYY-MM-DD HH:MM:SS', which compares correctly as a
// string; id is used as a tiebreaker (later insert -> newer board).
function compareCreatedAsc(a, b) {
  if (a.created_at === b.created_at) return a.id - b.id
  return (a.created_at || '') < (b.created_at || '') ? -1 : 1
}

const filteredBoards = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()

  const result = boardStore.boards.filter((board) => {
    if (filterKey.value === 'with-cards' && cardCountOf(board) === 0) return false
    if (filterKey.value === 'empty' && cardCountOf(board) > 0) return false
    if (query) {
      const name = (board.name || '').toLowerCase()
      const description = (board.description || '').toLowerCase()
      if (!name.includes(query) && !description.includes(query)) return false
    }
    return true
  })

  switch (sortKey.value) {
    case 'oldest':
      result.sort(compareCreatedAsc)
      break
    case 'name':
      result.sort(
        (a, b) => (a.name || '').localeCompare(b.name || '') || a.id - b.id
      )
      break
    case 'cards':
      result.sort(
        (a, b) => cardCountOf(b) - cardCountOf(a) || b.id - a.id
      )
      break
    case 'columns':
      result.sort(
        (a, b) => columnCountOf(b) - columnCountOf(a) || b.id - a.id
      )
      break
    case 'newest':
    default:
      result.sort((a, b) => compareCreatedAsc(b, a))
  }

  return result
})

function resetView() {
  searchQuery.value = ''
  sortKey.value = 'newest'
  filterKey.value = 'all'
}
// --------------------------------------------------------------------------------

onMounted(() => {
  boardStore.fetchBoards()
})

function openBoard(board) {
  router.push(`/board/${board.id}`)
}

async function handleCreateBoard() {
  if (!createFormRef.value) return
  const valid = await createFormRef.value.validate().catch(() => false)
  if (!valid) return

  creating.value = true
  try {
    const board = await boardStore.createBoard(createForm.value.name, createForm.value.description)
    showCreateDialog.value = false
    createForm.value = { name: '', description: '' }
    ElMessage.success('Board created!')
    router.push(`/board/${board.id}`)
  } catch (err) {
    ElMessage.error(err.response?.data?.error || 'Failed to create board')
  } finally {
    creating.value = false
  }
}

async function confirmDeleteBoard(board) {
  try {
    await ElMessageBox.confirm(
      `Are you sure you want to delete "${board.name}"? All columns and cards will be permanently removed.`,
      'Delete Board',
      { type: 'warning', confirmButtonText: 'Delete', cancelButtonText: 'Cancel' }
    )
    await boardStore.deleteBoard(board.id)
    ElMessage.success('Board deleted')
  } catch (err) {
    if (err !== 'cancel') {
      ElMessage.error('Failed to delete board')
    }
  }
}
</script>

<style scoped>
.home-page {
  padding: 30px;
}

.home-container {
  max-width: 1200px;
  margin: 0 auto;
}

.home-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.home-header h1 {
  font-size: 28px;
  color: #303133;
}

.overview-stats {
  display: flex;
  align-items: center;
  gap: 12px;
  background: #fff;
  border: 1px solid #ebeef5;
  border-radius: 8px;
  padding: 14px 20px;
  margin-bottom: 16px;
}

.stat-item {
  display: flex;
  align-items: baseline;
  gap: 8px;
}

.stat-value {
  font-size: 22px;
  font-weight: 600;
  color: #303133;
}

.stat-label {
  font-size: 13px;
  color: #909399;
}

.overview-stats :deep(.el-divider--vertical) {
  height: 24px;
}

.boards-toolbar {
  display: flex;
  gap: 12px;
  margin-bottom: 12px;
}

.toolbar-search {
  flex: 1;
  max-width: 360px;
}

.toolbar-select {
  width: 170px;
}

.result-count {
  margin: 0 0 16px;
  font-size: 13px;
  color: #909399;
}

.boards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.loading-state {
  text-align: center;
  padding: 60px;
  color: #909399;
}

.loading-state p {
  margin-top: 12px;
}

.empty-state {
  padding: 60px 0;
}
</style>
