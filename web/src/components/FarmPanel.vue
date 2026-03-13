<script setup lang="ts">
import { onClickOutside, useIntervalFn, useWindowSize } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import ConfirmModal from '@/components/ConfirmModal.vue'
import LandCard from '@/components/LandCard.vue'
import { useAccountStore } from '@/stores/account'
import { useFarmStore } from '@/stores/farm'
import { useStatusStore } from '@/stores/status'

const farmStore = useFarmStore()
const accountStore = useAccountStore()
const statusStore = useStatusStore()
const { lands, summary, loading } = storeToRefs(farmStore)
const { currentAccountId, currentAccount } = storeToRefs(accountStore)
const { status, loading: statusLoading, realtimeConnected } = storeToRefs(statusStore)
const { width } = useWindowSize()

const operating = ref(false)
const confirmVisible = ref(false)
const confirmConfig = ref({
  title: '',
  message: '',
  opType: '',
  options: null as any,
})

// 右键菜单相关
const contextMenu = ref<HTMLElement | null>(null)
const contextMenuVisible = ref(false)
const contextMenuPosition = ref({ x: 0, y: 0 })
const selectedLand = ref<any>(null)

// 种子选择相关
const seedModalVisible = ref(false)
const bagSeeds = ref<any[]>([])
const seedLoading = ref(false)
const seedSearch = ref('')

onClickOutside(contextMenu, () => {
  contextMenuVisible.value = false
})

function showContextMenu(e: MouseEvent, land: any) {
  e.preventDefault()
  selectedLand.value = land
  contextMenuPosition.value = { x: e.clientX, y: e.clientY }
  contextMenuVisible.value = true
}

async function handleMenuAction(action: string) {
  contextMenuVisible.value = false
  if (!selectedLand.value || !currentAccountId.value)
    return

  const landId = selectedLand.value.id

  if (action === 'remove') {
    confirmConfig.value = {
      title: '确认铲除',
      message: `确定要铲除土地 #${landId} 上的作物吗？`,
      opType: 'remove_one',
      options: { landId },
    }
    confirmVisible.value = true
  }
  else if (action === 'fertilize_organic') {
    await farmStore.operate(currentAccountId.value, 'fertilize_one', {
      landId,
      fertilizerId: 1012, // 有机肥 ID
    })
  }
  else if (action === 'plant') {
    openSeedModal()
  }
}

async function openSeedModal() {
  if (!currentAccountId.value)
    return
  seedModalVisible.value = true
  seedLoading.value = true
  try {
    bagSeeds.value = await farmStore.fetchBagSeeds(currentAccountId.value)
  }
  finally {
    seedLoading.value = false
  }
}

async function selectSeed(seed: any) {
  if (!selectedLand.value || !currentAccountId.value)
    return
  seedModalVisible.value = false
  await farmStore.operate(currentAccountId.value, 'plant_one', {
    landId: selectedLand.value.id,
    seedId: seed.id,
  })
}

// 过滤种子
const filteredSeeds = computed(() => {
  if (!seedSearch.value)
    return bagSeeds.value
  const keyword = seedSearch.value.toLowerCase()
  return bagSeeds.value.filter(s => (s.name || '').toLowerCase().includes(keyword))
})

async function executeOperate() {
  if (!currentAccountId.value || !confirmConfig.value.opType)
    return
  confirmVisible.value = false
  operating.value = true
  try {
    await farmStore.operate(currentAccountId.value, confirmConfig.value.opType, confirmConfig.value.options)
  }
  finally {
    operating.value = false
  }
}

function handleOperate(opType: string) {
  if (!currentAccountId.value)
    return

  const confirmMap: Record<string, string> = {
    harvest: '确定要收获所有成熟作物吗？',
    clear: '确定要一键除草/除虫吗？',
    plant: '确定要一键种植吗？(根据策略配置)',
    upgrade: '确定要升级所有可升级的土地吗？(消耗金币)',
    all: '确定要一键全收吗？(包含收获、除草、种植等)',
    ripen: '确定要一键催熟吗？(优先普通肥，催熟后自动收获并按策略种植)',
  }

  confirmConfig.value = {
    title: '确认操作',
    message: confirmMap[opType] || '确定执行此操作吗？',
    opType,
    options: null,
  }
  confirmVisible.value = true
}

const operations = [
  { type: 'harvest', label: '收获', icon: 'i-carbon-wheat', color: 'bg-blue-600 hover:bg-blue-700' },
  { type: 'ripen', label: '催熟', icon: 'i-carbon-chemistry', color: 'bg-amber-600 hover:bg-amber-700' },
  { type: 'clear', label: '除草/虫', icon: 'i-carbon-clean', color: 'bg-teal-600 hover:bg-teal-700' },
  { type: 'plant', label: '种植', icon: 'i-carbon-sprout', color: 'bg-green-600 hover:bg-green-700' },
  { type: 'upgrade', label: '升级土地', icon: 'i-carbon-upgrade', color: 'bg-purple-600 hover:bg-purple-700' },
  { type: 'all', label: '一键全收', icon: 'i-carbon-flash', color: 'bg-orange-600 hover:bg-orange-700' },
]

const displayLands = computed(() => {
  const list = Array.isArray(lands.value)
    ? [...lands.value].sort((a: any, b: any) => Number(a?.id || 0) - Number(b?.id || 0))
    : []

  if (width.value < 768)
    return list

  const columns = width.value >= 1024 ? 8 : 4
  const ordered: any[] = list

  if (width.value < 1024)
    return ordered

  const groupMap = new Map<number, {
    plantSize: number
    members: Array<{ index: number, row: number, col: number, land: any }>
  }>()

  for (let i = 0; i < ordered.length; i++) {
    const land = ordered[i]
    const plantSize = Math.max(1, Number(land?.plantSize) || 1)
    const masterLandId = Number(land?.masterLandId || 0)
    if (plantSize <= 1 || !masterLandId)
      continue

    const row = Math.floor(i / columns)
    const col = i % columns
    if (!groupMap.has(masterLandId)) {
      groupMap.set(masterLandId, {
        plantSize,
        members: [],
      })
    }

    groupMap.get(masterLandId)!.members.push({ index: i, row, col, land })
  }

  const mergedAnchors = new Map<number, any>()
  const hiddenIndexes = new Set<number>()

  for (const [, group] of groupMap) {
    const size = Math.max(1, Number(group.plantSize) || 1)
    if (group.members.length < size * size)
      continue

    const rowsInGroup = group.members.map(member => member.row)
    const colsInGroup = group.members.map(member => member.col)
    const minRow = Math.min(...rowsInGroup)
    const maxRow = Math.max(...rowsInGroup)
    const minCol = Math.min(...colsInGroup)
    const maxCol = Math.max(...colsInGroup)

    if ((maxRow - minRow + 1) !== size || (maxCol - minCol + 1) !== size)
      continue

    const occupiedCells = new Set(group.members.map(member => `${member.row}:${member.col}`))
    let isFullRectangle = true
    for (let row = minRow; row <= maxRow; row++) {
      for (let col = minCol; col <= maxCol; col++) {
        if (!occupiedCells.has(`${row}:${col}`)) {
          isFullRectangle = false
          break
        }
      }
      if (!isFullRectangle)
        break
    }

    if (!isFullRectangle)
      continue

    const anchor = group.members.find(member => member.row === minRow && member.col === minCol)
    if (!anchor)
      continue

    const mergedLandIds: number[][] = []
    for (let row = minRow; row <= maxRow; row++) {
      const rowIds: number[] = []
      for (let col = minCol; col <= maxCol; col++) {
        const member = group.members.find(item => item.row === row && item.col === col)
        if (member)
          rowIds.push(Number(member.land?.id || 0))
      }
      if (rowIds.length > 0)
        mergedLandIds.push(rowIds)
    }

    mergedAnchors.set(anchor.index, {
      ...anchor.land,
      mergedCard: true,
      mergedLandIds,
    })

    for (const member of group.members) {
      if (member.index !== anchor.index)
        hiddenIndexes.add(member.index)
    }
  }

  const merged: any[] = []
  for (let i = 0; i < ordered.length; i++) {
    if (hiddenIndexes.has(i))
      continue
    merged.push(mergedAnchors.get(i) || ordered[i])
  }

  return merged
})

function getLandWrapperClass(land: any) {
  if (land?.mergedCard)
    return 'lg:col-span-2 lg:row-span-2'
  return ''
}

async function refresh() {
  if (currentAccountId.value) {
    const acc = currentAccount.value
    if (!acc)
      return

    if (!realtimeConnected.value)
      await statusStore.fetchStatus(currentAccountId.value)

    if (acc.running && status.value?.connection?.connected)
      farmStore.fetchLands(currentAccountId.value)
  }
}

watch(currentAccountId, () => {
  refresh()
})

const { pause, resume } = useIntervalFn(() => {
  if (lands.value) {
    lands.value = lands.value.map((l: any) =>
      l.matureInSec > 0 ? { ...l, matureInSec: l.matureInSec - 1 } : l,
    )
  }
}, 1000)

const { pause: pauseRefresh, resume: resumeRefresh } = useIntervalFn(refresh, 60000)

onMounted(() => {
  refresh()
  resume()
  resumeRefresh()
})

onUnmounted(() => {
  pause()
  pauseRefresh()
})
</script>

<template>
  <div class="space-y-4">
    <div class="rounded-lg bg-white shadow dark:bg-gray-800">
      <div class="flex flex-col items-center justify-between gap-4 border-b border-gray-100 p-4 sm:flex-row dark:border-gray-700">
        <h3 class="flex items-center gap-2 text-lg font-bold">
          <div class="i-carbon-grid text-xl" />
          土地详情
        </h3>
        <div class="grid grid-cols-2 gap-2 sm:flex sm:flex-wrap">
          <button
            v-for="op in operations"
            :key="op.type"
            class="flex items-center justify-center gap-1.5 rounded px-3 py-2 text-sm text-white transition disabled:cursor-not-allowed disabled:opacity-50"
            :class="op.color"
            :disabled="operating"
            @click="handleOperate(op.type)"
          >
            <div :class="op.icon" />
            {{ op.label }}
          </button>
        </div>
      </div>

      <div class="flex flex-wrap gap-4 border-b border-gray-100 bg-gray-50 p-4 text-sm dark:border-gray-700 dark:bg-gray-900/50">
        <div class="flex items-center gap-1.5 rounded-full bg-orange-100 px-3 py-1 text-orange-700 dark:bg-orange-900/30 dark:text-orange-400">
          <div class="i-carbon-clean" />
          <span class="font-medium">可收: {{ summary?.harvestable || 0 }}</span>
        </div>
        <div class="flex items-center gap-1.5 rounded-full bg-green-100 px-3 py-1 text-green-700 dark:bg-green-900/30 dark:text-green-400">
          <div class="i-carbon-sprout" />
          <span class="font-medium">生长: {{ summary?.growing || 0 }}</span>
        </div>
        <div class="flex items-center gap-1.5 rounded-full bg-gray-100 px-3 py-1 text-gray-700 dark:bg-gray-800 dark:text-gray-400">
          <div class="i-carbon-checkbox" />
          <span class="font-medium">空闲: {{ summary?.empty || 0 }}</span>
        </div>
        <div class="flex items-center gap-1.5 rounded-full bg-red-100 px-3 py-1 text-red-700 dark:bg-red-900/30 dark:text-red-400">
          <div class="i-carbon-warning" />
          <span class="font-medium">枯萎: {{ summary?.dead || 0 }}</span>
        </div>
      </div>

      <div class="p-4">
        <div v-if="loading || statusLoading" class="flex justify-center py-12">
          <div class="i-svg-spinners-90-ring-with-bg text-4xl text-blue-500" />
        </div>

        <div v-else-if="!status?.connection?.connected" class="flex flex-col items-center justify-center gap-4 rounded-lg bg-white p-12 text-center text-gray-500 shadow dark:bg-gray-800">
          <div class="i-carbon-connection-signal-off text-4xl text-gray-400" />
          <div>
            <div class="text-lg text-gray-700 font-medium dark:text-gray-300">
              账号未登录
            </div>
            <div class="mt-1 text-sm text-gray-400">
              请先运行账号或检查网络连接
            </div>
          </div>
        </div>

        <div v-else-if="!displayLands.length" class="flex justify-center py-12 text-gray-500">
          暂无土地数据
        </div>

        <div v-else class="grid grid-cols-2 gap-4 lg:grid-cols-8 md:grid-cols-4 sm:grid-cols-3">
          <div
            v-for="land in displayLands"
            :key="land.id"
            :class="getLandWrapperClass(land)"
            @contextmenu.prevent="showContextMenu($event, land)"
          >
            <LandCard :land="land" />
          </div>
        </div>
      </div>
    </div>

    <ConfirmModal
      :show="confirmVisible"
      :title="confirmConfig.title"
      :message="confirmConfig.message"
      @confirm="executeOperate"
      @cancel="confirmVisible = false"
    />

    <!-- Context Menu -->
    <div
      v-if="contextMenuVisible"
      ref="contextMenu"
      class="fixed z-50 min-w-[140px] overflow-hidden rounded-md bg-white shadow-xl ring-1 ring-black ring-opacity-5 dark:bg-gray-800 dark:ring-white dark:ring-opacity-10"
      :style="{ top: `${contextMenuPosition.y}px`, left: `${contextMenuPosition.x}px` }"
    >
      <div class="py-1">
        <button
          class="block w-full flex items-center gap-2 px-4 py-2 text-left text-sm text-gray-700 hover:bg-gray-100 dark:text-gray-200 dark:hover:bg-gray-700"
          @click="handleMenuAction('plant')"
        >
          <div class="i-carbon-sprout text-green-500" />
          种植指定种子
        </button>
        <button
          class="block w-full flex items-center gap-2 px-4 py-2 text-left text-sm text-gray-700 hover:bg-gray-100 dark:text-gray-200 dark:hover:bg-gray-700"
          @click="handleMenuAction('fertilize_organic')"
        >
          <div class="i-carbon-chemistry text-amber-500" />
          施加有机肥
        </button>
        <div class="my-1 border-t border-gray-100 dark:border-gray-700" />
        <button
          class="block w-full flex items-center gap-2 px-4 py-2 text-left text-sm text-red-600 hover:bg-red-50 dark:text-red-400 dark:hover:bg-red-900/30"
          @click="handleMenuAction('remove')"
        >
          <div class="i-carbon-trash-can" />
          铲除作物
        </button>
      </div>
    </div>

    <!-- Seed Selection Modal -->
    <div v-if="seedModalVisible" class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 p-4 backdrop-blur-sm" @click="seedModalVisible = false">
      <div class="max-h-[80vh] max-w-lg w-full flex flex-col transform rounded-xl bg-white p-6 shadow-2xl transition-all dark:bg-gray-800" @click.stop>
        <div class="mb-4 flex items-center justify-between">
          <h3 class="flex items-center gap-2 text-xl font-bold dark:text-gray-100">
            <div class="i-carbon-sprout text-green-600" />
            选择种子
          </h3>
          <button class="text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-200" @click="seedModalVisible = false">
            <div class="i-carbon-close text-xl" />
          </button>
        </div>

        <div class="relative mb-4">
          <div class="pointer-events-none absolute inset-y-0 left-0 flex items-center pl-3">
            <div class="i-carbon-search text-gray-400" />
          </div>
          <input
            v-model="seedSearch"
            type="text"
            placeholder="搜索种子名称..."
            class="w-full border border-gray-300 rounded-lg bg-gray-50 p-2.5 pl-10 text-sm text-gray-900 dark:border-gray-600 focus:border-blue-500 dark:bg-gray-700 dark:text-white focus:ring-blue-500 dark:focus:border-blue-500 dark:focus:ring-blue-500 dark:placeholder-gray-400"
          >
        </div>

        <div class="custom-scrollbar min-h-[300px] flex-1 overflow-y-auto pr-1 -mr-1">
          <div v-if="seedLoading" class="h-full flex flex-col items-center justify-center py-12">
            <div class="i-svg-spinners-90-ring-with-bg mb-2 text-4xl text-blue-500" />
            <span class="text-gray-500">加载背包中...</span>
          </div>
          <div v-else-if="filteredSeeds.length === 0" class="h-full flex flex-col items-center justify-center py-12 text-gray-500">
            <div class="i-carbon-catalog mb-2 text-4xl text-gray-300" />
            <span>{{ seedSearch ? '未找到匹配的种子' : '背包中没有种子' }}</span>
          </div>
          <div v-else class="grid grid-cols-2 gap-3 pb-2 sm:grid-cols-3">
            <button
              v-for="seed in filteredSeeds"
              :key="seed.id"
              class="group relative flex flex-col items-center gap-2 border border-gray-200 rounded-lg p-3 text-center transition dark:border-gray-700 hover:border-blue-500 dark:bg-gray-800 hover:bg-blue-50 hover:shadow-md dark:hover:border-blue-500 dark:hover:bg-blue-900/20"
              @click="selectSeed(seed)"
            >
              <div class="h-14 w-14 flex items-center justify-center rounded-full bg-gray-100 p-1 transition-transform group-hover:scale-110 dark:bg-gray-700">
                <img v-if="seed.seedImage" :src="seed.seedImage" class="max-h-full max-w-full object-contain">
                <div v-else class="i-carbon-sprout text-3xl text-green-500" />
              </div>
              <div class="w-full">
                <div class="truncate text-sm text-gray-900 font-medium dark:text-gray-100" :title="seed.name">
                  {{ seed.name }}
                </div>
                <div class="mt-1 inline-block rounded bg-gray-100 px-2 py-0.5 text-xs text-gray-500 dark:bg-gray-700 dark:text-gray-400">
                  数量: {{ seed.amount || seed.count }}
                </div>
              </div>
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
