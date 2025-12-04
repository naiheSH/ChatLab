<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'

// 缓存目录信息类型
interface CacheDirectoryInfo {
  id: string
  name: string
  description: string
  path: string
  icon: string
  canClear: boolean
  size: number
  fileCount: number
  exists: boolean
}

interface CacheInfo {
  baseDir: string
  directories: CacheDirectoryInfo[]
  totalSize: number
}

// 状态
const cacheInfo = ref<CacheInfo | null>(null)
const isLoading = ref(false)
const clearingId = ref<string | null>(null)

// 格式化文件大小
function formatSize(bytes: number): string {
  if (bytes === 0) return '0 B'
  const units = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(1024))
  const size = (bytes / Math.pow(1024, i)).toFixed(i > 0 ? 1 : 0)
  return `${size} ${units[i]}`
}

// 计算总大小
const totalSizeFormatted = computed(() => {
  if (!cacheInfo.value) return '0 B'
  return formatSize(cacheInfo.value.totalSize)
})

// 加载缓存信息
async function loadCacheInfo() {
  isLoading.value = true
  try {
    cacheInfo.value = await window.cacheApi.getInfo()
  } catch (error) {
    console.error('获取缓存信息失败:', error)
  } finally {
    isLoading.value = false
  }
}

// 清理缓存
async function clearCache(cacheId: string) {
  clearingId.value = cacheId
  try {
    const result = await window.cacheApi.clear(cacheId)
    if (result.success) {
      // 刷新缓存信息
      await loadCacheInfo()
    } else {
      console.error('清理缓存失败:', result.error)
    }
  } catch (error) {
    console.error('清理缓存失败:', error)
  } finally {
    clearingId.value = null
  }
}

// 打开目录
async function openDirectory(cacheId: string) {
  try {
    await window.cacheApi.openDir(cacheId)
  } catch (error) {
    console.error('打开目录失败:', error)
  }
}

// 打开根目录
async function openBaseDir() {
  try {
    await window.cacheApi.openDir('base')
  } catch (error) {
    console.error('打开目录失败:', error)
  }
}

// 组件挂载时加载数据
onMounted(() => {
  loadCacheInfo()
})

// 暴露刷新方法
defineExpose({
  refresh: loadCacheInfo,
})
</script>

<template>
  <div class="space-y-6">
    <!-- 标题和总览 -->
    <div class="flex items-center justify-between">
      <div>
        <h3 class="flex items-center gap-2 text-sm font-semibold text-gray-900 dark:text-white">
          <UIcon name="i-heroicons-folder-open" class="h-4 w-4 text-amber-500" />
          本地缓存管理
        </h3>
        <p class="mt-1 text-xs text-gray-500 dark:text-gray-400">管理 ChatLab 在本地存储的数据文件</p>
      </div>
      <div class="flex items-center gap-2">
        <!-- 总大小 -->
        <div class="rounded-lg bg-gray-100 px-3 py-1.5 dark:bg-gray-800">
          <span class="text-xs text-gray-500 dark:text-gray-400">总占用：</span>
          <span class="text-sm font-semibold text-gray-900 dark:text-white">{{ totalSizeFormatted }}</span>
        </div>
        <!-- 刷新按钮 -->
        <UButton icon="i-heroicons-arrow-path" variant="ghost" size="sm" :loading="isLoading" @click="loadCacheInfo" />
      </div>
    </div>

    <!-- 加载状态 -->
    <div v-if="isLoading && !cacheInfo" class="flex items-center justify-center py-8">
      <UIcon name="i-heroicons-arrow-path" class="h-5 w-5 animate-spin text-gray-400" />
      <span class="ml-2 text-sm text-gray-500">加载中...</span>
    </div>

    <!-- 缓存目录列表 -->
    <div v-else-if="cacheInfo" class="space-y-3">
      <div
        v-for="dir in cacheInfo.directories"
        :key="dir.id"
        class="rounded-lg border border-gray-200 bg-gray-50 p-4 transition-colors hover:bg-gray-100 dark:border-gray-700 dark:bg-gray-800/50 dark:hover:bg-gray-800"
      >
        <div class="flex items-start justify-between">
          <!-- 左侧信息 -->
          <div class="flex items-start gap-3">
            <div
              class="flex h-10 w-10 shrink-0 items-center justify-center rounded-lg"
              :class="{
                'bg-green-100 dark:bg-green-900/30': dir.id === 'databases',
                'bg-violet-100 dark:bg-violet-900/30': dir.id === 'ai',
                'bg-amber-100 dark:bg-amber-900/30': dir.id === 'temp',
                'bg-blue-100 dark:bg-blue-900/30': dir.id === 'logs',
              }"
            >
              <UIcon
                :name="dir.icon"
                class="h-5 w-5"
                :class="{
                  'text-green-600 dark:text-green-400': dir.id === 'databases',
                  'text-violet-600 dark:text-violet-400': dir.id === 'ai',
                  'text-amber-600 dark:text-amber-400': dir.id === 'temp',
                  'text-blue-600 dark:text-blue-400': dir.id === 'logs',
                }"
              />
            </div>
            <div>
              <div class="flex items-center gap-2">
                <h4 class="text-sm font-medium text-gray-900 dark:text-white">{{ dir.name }}</h4>
                <UBadge v-if="!dir.exists" variant="soft" color="gray" size="xs">不存在</UBadge>
              </div>
              <p class="mt-0.5 text-xs text-gray-500 dark:text-gray-400">{{ dir.description }}</p>
              <div class="mt-2 flex items-center gap-3 text-xs text-gray-400">
                <span class="flex items-center gap-1">
                  <UIcon name="i-heroicons-document" class="h-3 w-3" />
                  {{ dir.fileCount }} 个文件
                </span>
                <span class="flex items-center gap-1">
                  <UIcon name="i-heroicons-server" class="h-3 w-3" />
                  {{ formatSize(dir.size) }}
                </span>
              </div>
            </div>
          </div>

          <!-- 右侧操作按钮 -->
          <div class="flex items-center gap-2">
            <!-- 清理按钮 -->
            <UButton
              v-if="dir.canClear && dir.size > 0"
              icon="i-heroicons-trash"
              variant="soft"
              color="red"
              size="xs"
              :loading="clearingId === dir.id"
              :disabled="clearingId !== null"
              @click="clearCache(dir.id)"
            >
              清理
            </UButton>
            <!-- 打开目录按钮 -->
            <UButton icon="i-heroicons-folder-open" variant="ghost" size="xs" @click="openDirectory(dir.id)">
              打开
            </UButton>
          </div>
        </div>
      </div>

      <!-- 底部操作栏 -->
      <div
        class="flex items-center justify-between rounded-lg border border-dashed border-gray-300 bg-gray-50/50 px-4 py-3 dark:border-gray-700 dark:bg-gray-800/30"
      >
        <div class="flex items-center gap-2 text-xs text-gray-500 dark:text-gray-400">
          <UIcon name="i-heroicons-folder" class="h-4 w-4" />
          <span class="font-mono">{{ cacheInfo.baseDir }}</span>
        </div>
        <UButton icon="i-heroicons-arrow-top-right-on-square" variant="link" size="xs" @click="openBaseDir">
          打开目录
        </UButton>
      </div>
    </div>

    <!-- 提示信息 -->
    <div class="rounded-lg border border-amber-200 bg-amber-50 p-3 dark:border-amber-800/50 dark:bg-amber-900/20">
      <div class="flex items-start gap-2">
        <UIcon name="i-heroicons-exclamation-triangle" class="h-4 w-4 shrink-0 text-amber-500" />
        <div class="text-xs text-amber-700 dark:text-amber-400">
          <p class="font-medium">注意事项</p>
          <ul class="mt-1 list-inside list-disc space-y-0.5 text-amber-600 dark:text-amber-500">
            <li>聊天记录数据库和 AI 数据不支持一键清理，请在分析页面逐个删除</li>
            <li>临时文件和日志文件可以安全清理</li>
            <li>清理后无法恢复，请谨慎操作</li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>
