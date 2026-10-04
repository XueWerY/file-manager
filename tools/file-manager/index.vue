<template>
  <div class="file-manager">
    <div class="fm-browser">
      <div class="fm-header">
        <div class="fm-breadcrumb">
          <span class="fm-breadcrumb-item" @click="navigateTo('root')">根目录</span>
          <template v-for="(seg, idx) in breadcrumbs" :key="idx">
            <span class="fm-sep">/</span>
            <span class="fm-breadcrumb-item" @click="seg.isFile ? null : navigateTo(seg.path)">{{ seg.name }}</span>
          </template>
        </div>
      </div>

      <div v-if="loading" class="fm-loading">加载中...</div>
      <div v-else-if="error" class="fm-error">{{ error }}</div>
      <div v-else class="fm-body">
        <div v-if="entries.length === 0" class="fm-empty">此目录为空</div>
        <div v-else class="fm-list">
          <div
            v-for="entry in sortedEntries"
            :key="entry.path"
            class="fm-item"
            :class="{ 'fm-item-dir': entry.isDirectory }"
            @click="entry.isDirectory ? navigateTo(entry.path) : openFile(entry)"
          >
            <span class="fm-icon">{{ entry.isDirectory ? '📁' : getFileIcon(entry.name) }}</span>
            <span class="fm-name">{{ entry.name }}</span>
            <span class="fm-size">云端数据</span>
          </div>
        </div>
      </div>

      <BaseDialog :visible="previewFile !== null" :title="previewFile?.name || ''" :width="680" teleport @update:visible="previewFile = null">
        <pre class="fm-preview-content">{{ previewContent || '加载中...' }}</pre>
      </BaseDialog>
    </div>
  </div>
</template>
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import BaseDialog from '../../../../components/ui/BaseDialog.vue'
import { logger } from '../../../../lib/logger'
import { getData } from '../../../../lib/api'

// ====== 云端数据目录定义 ======
interface ModuleChildDef {
  label: string
  dataKeys: string[]
}

interface ModuleGroupDef {
  key: string
  label: string
  children: ModuleChildDef[]
}

const MODULE_DEFS: ModuleGroupDef[] = [
  {
    key: 'footprint',
    label: '足迹',
    children: [
      { label: '记录', dataKeys: ['footprint/footprint'] },
      { label: '日记', dataKeys: ['footprint/diary'] }
    ]
  },
  {
    key: 'focus',
    label: '专注',
    children: [
      { label: '常用专注', dataKeys: ['focus/favorites'] },
      { label: '专注记录', dataKeys: ['focus/records'] }
    ]
  },
  {
    key: 'list',
    label: '清单',
    children: [
      { label: '清单列表', dataKeys: ['list/lists'] },
      { label: '清单任务', dataKeys: ['list/tasks'] },
      { label: '文件夹', dataKeys: ['list/folders'] },
      { label: '已完成', dataKeys: ['list/completed'] },
      { label: '收藏', dataKeys: ['list/favorites'] }
    ]
  },
  {
    key: 'countdown',
    label: '倒数日',
    children: [
      { label: '分类', dataKeys: ['countdown/categories'] },
      { label: '倒数日', dataKeys: ['countdown/countdowns'] }
    ]
  },
  {
    key: 'notes',
    label: '笔记',
    children: [
      { label: '笔记', dataKeys: ['notes/notes'] },
      { label: '分类', dataKeys: ['notes/categories'] },
      { label: '置顶', dataKeys: ['notes/favorites'] }
    ]
  },
  {
    key: 'course',
    label: '课程表',
    children: [
      { label: '课程', dataKeys: ['course/courses'] }
    ]
  },
  {
    key: 'settings',
    label: '设置',
    children: [
      { label: '设置', dataKeys: ['settings/settings'] }
    ]
  },
  {
    key: 'profile',
    label: '个人资料',
    children: [
      { label: '个人信息', dataKeys: ['profile/profile'] }
    ]
  }
]

interface FsEntry {
  name: string
  path: string
  isDirectory: boolean
  dataKey?: string
}

interface Breadcrumb {
  name: string
  path: string
  isFile: boolean
}

const loading = ref(false)
const error = ref('')
const currentDir = ref<string>('root')
const entries = ref<FsEntry[]>([])
const breadcrumbs = ref<Breadcrumb[]>([])
const previewFile = ref<FsEntry | null>(null)
const previewContent = ref('')

// 文件浏览仅展示云端业务数据，不访问本机文件系统。
const CLOUD_DATA_ROOT = 'cloud-data'
const CLOUD_DATA_GROUP_PREFIX = `${CLOUD_DATA_ROOT}/group/`
const CLOUD_DATA_FILE_PREFIX = `${CLOUD_DATA_ROOT}/file/`
const sortedEntries = computed(() => {
  const dirs = entries.value.filter(e => e.isDirectory)
  const files = entries.value.filter(e => !e.isDirectory)
  dirs.sort((a, b) => a.name.localeCompare(b.name))
  files.sort((a, b) => a.name.localeCompare(b.name))
  return [...dirs, ...files]
})

function getFileIcon(name: string): string {
  return name.toLowerCase().endsWith('.json') ? '📋' : '📁'
}

async function loadDir(dirPath: string) {
  loading.value = true
  error.value = ''
  try {
    if (dirPath === CLOUD_DATA_ROOT) {
      entries.value = MODULE_DEFS.map(group => ({
        name: group.label,
        path: `${CLOUD_DATA_GROUP_PREFIX}${group.key}`,
        isDirectory: true
      }))
    } else if (dirPath.startsWith(CLOUD_DATA_GROUP_PREFIX)) {
      const groupKey = dirPath.slice(CLOUD_DATA_GROUP_PREFIX.length)
      const group = MODULE_DEFS.find(item => item.key === groupKey)
      if (!group) throw new Error('找不到云端数据分类')
      entries.value = group.children.flatMap(child => child.dataKeys.map((dataKey, index) => ({
        name: `${child.label}${child.dataKeys.length > 1 ? ` ${index + 1}` : ''}.json`,
        path: `${CLOUD_DATA_FILE_PREFIX}${encodeURIComponent(dataKey)}`,
        isDirectory: false,
        dataKey
      })))
    } else {
      throw new Error('无效的云端数据目录')
    }
  } catch (e: any) {
    entries.value = []
    error.value = '读取失败: ' + (e.message || String(e))
    logger.error('[文件管理器] 读取目录失败', { path: dirPath, error: e })
  } finally {
    loading.value = false
  }
}

async function navigateTo(target: string) {
  if (target === 'root') {
    currentDir.value = 'root'
    breadcrumbs.value = []
    await showRoots()
    return
  }
  currentDir.value = target
  await loadDir(target)
  updateBreadcrumbs()
}

function updateBreadcrumbs() {
  if (currentDir.value === 'root') {
    breadcrumbs.value = []
    return
  }

  const groupKey = currentDir.value.startsWith(CLOUD_DATA_GROUP_PREFIX)
    ? currentDir.value.slice(CLOUD_DATA_GROUP_PREFIX.length)
    : ''
  const group = MODULE_DEFS.find(item => item.key === groupKey)
  breadcrumbs.value = [
    { name: '云端数据', path: CLOUD_DATA_ROOT, isFile: false },
    ...(group ? [{ name: group.label, path: currentDir.value, isFile: false }] : [])
  ]
}

async function showRoots() {
  loading.value = true
  error.value = ''
  currentDir.value = 'root'
  breadcrumbs.value = []
  try {
    entries.value = [{ name: '云端数据', path: CLOUD_DATA_ROOT, isDirectory: true }]
  } catch (e: any) {
    entries.value = []
    error.value = '加载失败: ' + (e.message || String(e))
    logger.error('[文件管理器] 加载云端数据入口失败', { error: e })
  } finally {
    loading.value = false
  }
}

async function openFile(entry: FsEntry) {
  if (entry.isDirectory) return
  if (!entry.dataKey) {
    previewFile.value = entry
    previewContent.value = '云端数据路径无效'
    return
  }

  previewFile.value = entry
  previewContent.value = '加载中...'
  try {
    const result = await getData(entry.dataKey)
    if (!result.success) throw new Error('云端数据读取失败')
    previewContent.value = result.data === null
      ? '云端暂无此项数据'
      : JSON.stringify(result.data, null, 2)
  } catch (e: any) {
    previewContent.value = '读取云端数据失败: ' + (e.message || String(e))
    logger.error('[文件管理器] 读取云端数据失败', { dataKey: entry.dataKey, error: e })
  }
}

onMounted(async () => {
  logger.info('[文件管理器] 已打开')
  await showRoots()
})

</script>

<style scoped>
.file-manager {
  height: 100%;
  display: flex;
  flex-direction: column;
  font-size: 13px;
  color: #ddd;
}

.fm-browser {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
}

.fm-header {
  padding: 8px 0 12px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  flex-shrink: 0;
}

.fm-breadcrumb {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 4px;
  font-size: 13px;
}

.fm-breadcrumb-item {
  color: var(--chalk-primary);
  cursor: pointer;
  padding: 2px 6px;
  border-radius: 4px;
  transition: background 0.15s;
}

.fm-breadcrumb-item:hover {
  background: rgba(102, 126, 234, 0.15);
}

.fm-sep {
  color: var(--chalk-dim);
  font-size: 12px;
}

.fm-loading,
.fm-error,
.fm-empty {
  text-align: center;
  padding: 32px 0;
  color: var(--chalk-dim);
}

.fm-error {
  color: var(--chalk-red);
}

.fm-body {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding-top: 8px;
}

.fm-list {
  display: flex;
  flex-direction: column;
}

.fm-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 12px;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.15s;
}

.fm-item:hover {
  background: rgba(255, 255, 255, 0.06);
}

.fm-item-dir {
  font-weight: 500;
}

.fm-icon {
  flex-shrink: 0;
  width: 24px;
  text-align: center;
  font-size: 18px;
}

.fm-name {
  flex: 1;
  color: var(--chalk-white);
  word-break: break-all;
}

.fm-size {
  min-width: 60px;
  flex-shrink: 0;
  color: var(--chalk-dim);
  font-size: 12px;
  text-align: right;
}

.fm-preview-content {
  white-space: pre-wrap;
  word-break: break-all;
  font-family: 'Consolas', 'Monaco', monospace;
  font-size: 12px;
  line-height: 1.6;
  color: #ccc;
  margin: 0;
}

.fm-body::-webkit-scrollbar {
  display: none;
}

.fm-body {
  scrollbar-width: none;
}
</style>
