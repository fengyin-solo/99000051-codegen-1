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

      <div v-else-if="loadError" class="loading-state">
        <el-result icon="error" title="Failed to load boards" sub-title="Please check your connection and try again.">
          <template #extra>
            <el-button type="primary" @click="loadBoards">Retry</el-button>
          </template>
        </el-result>
      </div>

      <div v-else-if="boardStore.boards.length === 0" class="empty-state">
        <el-empty description="No boards yet. Create your first board!">
          <el-button type="primary" @click="showCreateDialog = true">Create Board</el-button>
        </el-empty>
      </div>

      <template v-else>
        <!-- Aggregate stats across every board -->
        <div class="stats-row">
          <el-statistic title="Boards" :value="totalBoards" />
          <el-statistic title="Columns" :value="totalColumns" />
          <el-statistic title="Cards" :value="totalCards" />
        </div>

        <!-- Search / filter / sort toolbar -->
        <div class="toolbar">
          <el-input
            v-model="keyword"
            class="search-input"
            placeholder="Search boards by name or description"
            clearable
            :prefix-icon="Search"
          />
          <el-select v-model="cardFilter" class="filter-select">
            <el-option label="All boards" value="all" />
            <el-option label="With cards" value="hasCards" />
            <el-option label="No cards" value="noCards" />
          </el-select>
          <el-select v-model="sortBy" class="sort-select">
            <el-option label="Newest created" value="created_desc" />
            <el-option label="Oldest created" value="created_asc" />
            <el-option label="Name A–Z" value="name_asc" />
            <el-option label="Name Z–A" value="name_desc" />
            <el-option label="Most cards" value="cards_desc" />
            <el-option label="Fewest cards" value="cards_asc" />
          </el-select>
        </div>

        <div class="result-meta">
          <el-text type="info" size="small">
            Showing {{ filteredBoards.length }} of {{ boardStore.boards.length }} board(s)
          </el-text>
        </div>

        <div v-if="filteredBoards.length === 0" class="empty-state">
          <el-empty description="No boards match your search or filters.">
            <el-button @click="resetControls">Clear search & filters</el-button>
          </el-empty>
        </div>

        <div v-else class="boards-grid">
          <BoardCard
            v-for="board in filteredBoards"
            :key="board.id"
            :board="board"
            @open="openBoard"
            @delete="confirmDeleteBoard"
          />
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
import { Plus, Loading, Search } from '@element-plus/icons-vue'
import { useBoardStore } from '../stores/board.js'
import BoardCard from '../components/BoardCard.vue'

const router = useRouter()
const boardStore = useBoardStore()

const showCreateDialog = ref(false)
const creating = ref(false)
const createFormRef = ref(null)
const loadError = ref(false)

const createForm = ref({ name: '', description: '' })
const createRules = {
  name: [{ required: true, message: 'Board name is required', trigger: 'blur' }]
}

// Overview controls
const keyword = ref('')
const cardFilter = ref('all')
const sortBy = ref('created_desc')

onMounted(loadBoards)

async function loadBoards() {
  loadError.value = false
  try {
    await boardStore.fetchBoards()
  } catch (err) {
    loadError.value = true
  }
}

// Aggregate counts over every board the user owns
const totalBoards = computed(() => boardStore.boards.length)
const totalColumns = computed(() =>
  boardStore.boards.reduce((sum, b) => sum + (Number(b.column_count) || 0), 0)
)
const totalCards = computed(() =>
  boardStore.boards.reduce((sum, b) => sum + (Number(b.card_count) || 0), 0)
)

function createdTime(board) {
  const t = new Date(board.created_at).getTime()
  return Number.isNaN(t) ? 0 : t
}

const filteredBoards = computed(() => {
  const kw = keyword.value.trim().toLowerCase()
  let list = boardStore.boards.filter(board => {
    if (kw) {
      const name = (board.name || '').toLowerCase()
      const desc = (board.description || '').toLowerCase()
      if (!name.includes(kw) && !desc.includes(kw)) return false
    }
    const cardCount = Number(board.card_count) || 0
    if (cardFilter.value === 'hasCards' && cardCount === 0) return false
    if (cardFilter.value === 'noCards' && cardCount > 0) return false
    return true
  })

  const byName = (a, b) => (a.name || '').localeCompare(b.name || '')
  const sorters = {
    created_desc: (a, b) => createdTime(b) - createdTime(a) || b.id - a.id,
    created_asc: (a, b) => createdTime(a) - createdTime(b) || a.id - b.id,
    name_asc: byName,
    name_desc: (a, b) => byName(b, a),
    cards_desc: (a, b) => (Number(b.card_count) || 0) - (Number(a.card_count) || 0) || createdTime(b) - createdTime(a),
    cards_asc: (a, b) => (Number(a.card_count) || 0) - (Number(b.card_count) || 0) || createdTime(b) - createdTime(a)
  }
  return [...list].sort(sorters[sortBy.value] || sorters.created_desc)
})

function resetControls() {
  keyword.value = ''
  cardFilter.value = 'all'
  sortBy.value = 'created_desc'
}

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

.stats-row {
  display: flex;
  gap: 16px;
  margin-bottom: 20px;
}

.stats-row :deep(.el-statistic) {
  flex: 1;
  background: #fff;
  border-radius: 8px;
  padding: 16px 20px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.06);
}

.toolbar {
  display: flex;
  gap: 12px;
  margin-bottom: 10px;
  flex-wrap: wrap;
}

.search-input {
  flex: 1;
  min-width: 240px;
}

.filter-select {
  width: 160px;
}

.sort-select {
  width: 180px;
}

.result-meta {
  margin-bottom: 16px;
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
