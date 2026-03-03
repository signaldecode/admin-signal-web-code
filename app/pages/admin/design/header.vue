<script setup>
/**
 * 헤더 관리 페이지
 * - 쇼핑몰 헤더 네비게이션 구성 설정
 * - 칩으로 선택 → 헤더 미리보기에 표시 → 드래그앤드롭으로 순서 변경
 */

import { useUiStore } from '~/stores/ui'

const { $api } = useNuxtApp()
const uiStore = useUiStore()

const isLoading = ref(true)
const isSaving = ref(false)
const isEditMode = ref(false)
const originalLabels = ref({}) // 수정 전 원본 라벨 저장

// 수정 모드 시작
const toggleEditMode = () => {
  // 원본 라벨 저장
  originalLabels.value = {}
  availableNavItems.value.forEach(item => {
    originalLabels.value[item.id] = item.label
  })
  isEditMode.value = true
}

// 수정 모드 취소 (원본 복원)
const cancelEditMode = () => {
  // 원본 라벨 복원
  availableNavItems.value.forEach(item => {
    if (originalLabels.value[item.id]) {
      item.label = originalLabels.value[item.id]
    }
  })
  // selectedNavItems도 복원
  selectedNavItems.value.forEach(item => {
    if (originalLabels.value[item.id]) {
      item.label = originalLabels.value[item.id]
    }
  })
  isEditMode.value = false
}

// 사용 가능한 네비게이션 항목
const availableNavItems = ref([
  { id: 'best', label: '베스트' },
  { id: 'category', label: '카테고리' },
  { id: 'qna', label: 'Q&A' },
  { id: 'faq', label: 'FAQ' },
  { id: 'notice', label: '공지사항' },
  { id: 'review', label: '리뷰' },
  { id: 'support', label: '고객센터' },
  { id: 'coupons', label: '쿠폰' },
])

// 선택된 네비게이션 항목 (순서대로)
const selectedNavItems = ref([])

// 드래그 상태
const draggedIndex = ref(null)
const dragOverIndex = ref(null)

// 항목이 선택되어 있는지 확인
const isSelected = (id) => {
  return selectedNavItems.value.some(item => item.id === id)
}

// 칩 토글 (선택/해제)
const toggleNavItem = (item) => {
  if (isEditMode.value) return // 수정 모드에서는 토글 비활성화

  const index = selectedNavItems.value.findIndex(i => i.id === item.id)
  if (index > -1) {
    // 이미 선택됨 → 제거
    selectedNavItems.value.splice(index, 1)
  } else {
    // 선택 안 됨 → 추가 (availableNavItems에서 현재 label 가져오기)
    const sourceItem = availableNavItems.value.find(i => i.id === item.id)
    selectedNavItems.value.push({ id: item.id, label: sourceItem?.label || item.label })
  }
}

// selectedNavItems의 label도 업데이트 (수정 모드에서 label 변경 시)
const updateSelectedItemLabel = (id, newLabel) => {
  const item = selectedNavItems.value.find(i => i.id === id)
  if (item) {
    item.label = newLabel
  }
}

// 헤더에서 항목 제거
const removeFromHeader = (index) => {
  selectedNavItems.value.splice(index, 1)
}

// 드래그 시작
const onDragStart = (e, index) => {
  draggedIndex.value = index
  e.dataTransfer.effectAllowed = 'move'
}

// 드래그 오버
const onDragOver = (e, index) => {
  e.preventDefault()
  dragOverIndex.value = index
}

// 드래그 떠남
const onDragLeave = () => {
  dragOverIndex.value = null
}

// 드롭
const onDrop = (e, index) => {
  e.preventDefault()
  if (draggedIndex.value !== null && draggedIndex.value !== index) {
    const items = [...selectedNavItems.value]
    const [draggedItem] = items.splice(draggedIndex.value, 1)
    items.splice(index, 0, draggedItem)
    selectedNavItems.value = items
  }
  draggedIndex.value = null
  dragOverIndex.value = null
}

// 드래그 종료
const onDragEnd = () => {
  draggedIndex.value = null
  dragOverIndex.value = null
}

// 데이터 로드
const fetchHeaderConfig = async () => {
  isLoading.value = true
  try {
    const response = await $api.get('/admin/tenant/header-menu')

    // order 기준으로 정렬하여 저장
    const menus = response.data?.menus || []

    // API에서 받은 label로 availableNavItems 업데이트
    menus.forEach(menu => {
      const item = availableNavItems.value.find(i => i.id === menu.id)
      if (item && menu.label) {
        item.label = menu.label
      }
    })

    selectedNavItems.value = menus
      .sort((a, b) => a.order - b.order)
      .map(item => ({ id: item.id, label: item.label }))
  } catch (error) {
    uiStore.showToast({ type: 'error', message: '헤더 설정을 불러오지 못했습니다.' })
  } finally {
    isLoading.value = false
  }
}

// 저장
const handleSave = async () => {
  isSaving.value = true
  try {
    const payload = {
      menus: selectedNavItems.value.map((item, index) => ({
        id: item.id,
        label: item.label,
        order: index,
      }))
    }

    await $api.put('/admin/tenant/header-menu', payload)

    uiStore.showToast({ type: 'success', message: '헤더 설정이 저장되었습니다.' })

    // 수정 모드였으면 종료
    if (isEditMode.value) {
      isEditMode.value = false
    }
  } catch (error) {
    uiStore.showToast({ type: 'error', message: '저장에 실패했습니다.' })
  } finally {
    isSaving.value = false
  }
}

onMounted(() => {
  fetchHeaderConfig()
})
</script>

<template>
  <LayoutFormPage
    title="헤더 관리"
    description="쇼핑몰 헤더에 표시될 네비게이션을 설정합니다."
    :is-saving="isSaving"
    @save="handleSave"
  >
    <div v-if="isLoading" class="flex items-center justify-center py-20">
      <UiSpinner size="lg" />
    </div>

    <div v-else class="space-y-6">
      <!-- 헤더 미리보기 -->
      <UiCard>
        <template #header>
          <div class="flex items-center justify-between">
            <h3 class="font-semibold text-neutral-900">헤더 미리보기</h3>
            <span class="text-xs text-neutral-400">드래그하여 순서를 변경하세요</span>
          </div>
        </template>

        <!-- 미리보기 영역 -->
        <div class="bg-white border border-neutral-200 rounded-lg overflow-hidden">
          <div class="bg-white border-b border-neutral-200 px-6 py-4">
            <div class="flex items-center">
              <!-- 로고 -->
              <div class="text-neutral-900 font-bold text-lg">LOGO</div>

              <!-- 네비게이션 (드래그 가능) -->
              <nav class="flex items-center gap-1 ml-6">
                <div
                  v-for="(item, index) in selectedNavItems"
                  :key="item.id"
                  draggable="true"
                  :class="[
                    'group relative px-4 py-2 text-neutral-900 text-sm cursor-move rounded transition-all',
                    dragOverIndex === index ? 'bg-primary-100' : 'hover:bg-neutral-100',
                    draggedIndex === index ? 'opacity-50' : '',
                  ]"
                  @dragstart="onDragStart($event, index)"
                  @dragover="onDragOver($event, index)"
                  @dragleave="onDragLeave"
                  @drop="onDrop($event, index)"
                  @dragend="onDragEnd"
                >
                  {{ item.label }}
                  <!-- 삭제 버튼 (호버 시 표시) -->
                  <button
                    type="button"
                    class="absolute -top-1 -right-1 w-4 h-4 bg-error-500 text-white rounded-full opacity-0 group-hover:opacity-100 transition-opacity flex items-center justify-center"
                    @click.stop="removeFromHeader(index)"
                  >
                    <svg class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                    </svg>
                  </button>
                </div>

                <!-- 빈 상태 -->
                <span v-if="selectedNavItems.length === 0" class="text-neutral-400 text-sm px-4 py-2">
                  아래에서 항목을 선택하세요
                </span>
              </nav>

              <!-- 아이콘 영역 -->
              <div class="flex items-center gap-4 text-neutral-700 ml-auto">
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
                </svg>
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 3h2l.4 2M7 13h10l4-8H5.4M7 13L5.4 5M7 13l-2.293 2.293c-.63.63-.184 1.707.707 1.707H17m0 0a2 2 0 100 4 2 2 0 000-4zm-8 2a2 2 0 11-4 0 2 2 0 014 0z" />
                </svg>
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M16 7a4 4 0 11-8 0 4 4 0 018 0zM12 14a7 7 0 00-7 7h14a7 7 0 00-7-7z" />
                </svg>
              </div>
            </div>
          </div>
        </div>
      </UiCard>

      <!-- 네비게이션 항목 선택 -->
      <UiCard>
        <template #header>
          <div class="flex items-center justify-between">
            <h3 class="font-semibold text-neutral-900">네비게이션 항목</h3>
            <UiButton
              v-if="!isEditMode"
              variant="outline"
              size="sm"
              @click="toggleEditMode"
            >
              <svg class="w-4 h-4 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
              </svg>
              수정
            </UiButton>
            <UiButton
              v-else
              variant="ghost"
              size="sm"
              @click="cancelEditMode"
            >
              취소
            </UiButton>
          </div>
        </template>

        <!-- 수정 모드: input 필드 -->
        <div v-if="isEditMode" class="space-y-3">
          <!-- 안내 문구 -->
          <div class="flex items-center gap-2 p-3 bg-amber-50 border border-amber-200 rounded-lg mb-4">
            <svg class="w-4 h-4 text-amber-500 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
            </svg>
            <p class="text-sm text-amber-800">수정 후 <span class="font-semibold">하단의 헤더 저장</span> 버튼을 클릭해주세요.</p>
          </div>

          <div
            v-for="item in availableNavItems"
            :key="item.id"
            class="flex items-center gap-3"
          >
            <span class="w-24 text-sm text-neutral-500">{{ item.id }}</span>
            <input
              v-model="item.label"
              type="text"
              class="flex-1 px-3 py-2 border border-neutral-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-primary-500"
              @input="updateSelectedItemLabel(item.id, item.label)"
            >
          </div>
          <p class="text-xs text-neutral-400 mt-3">항목의 표시 이름을 수정할 수 있습니다.</p>
        </div>

        <!-- 일반 모드: 칩 버튼 -->
        <div v-else>
          <div class="flex flex-wrap gap-2">
            <button
              v-for="item in availableNavItems"
              :key="item.id"
              type="button"
              :class="[
                'px-4 py-2 rounded-full text-sm font-medium text-white transition-colors',
                isSelected(item.id)
                  ? 'bg-primary-600'
                  : 'bg-neutral-500 hover:bg-neutral-600',
              ]"
              @click="toggleNavItem(item)"
            >
              {{ item.label }}
            </button>
          </div>
          <p class="text-xs text-neutral-400 mt-3">클릭하여 선택/해제합니다. 선택된 항목은 헤더에 표시됩니다.</p>
        </div>
      </UiCard>
    </div>
  </LayoutFormPage>
</template>
