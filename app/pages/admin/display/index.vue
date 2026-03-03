<script setup>
/**
 * 전시 관리 - 메인 전시 페이지
 * - 메인 페이지 전시 영역 구성 관리
 * - 섹션별 와이어프레임 미리보기
 */

import { useUiStore } from '~/stores/ui'

const { $api } = useNuxtApp()
const uiStore = useUiStore()

const isLoading = ref(true)
const isSaving = ref(false)

// 섹션별 기본 설정 (type, displayCount 등 프론트에서 관리하는 값)
const sectionDefaults = {
  best: { type: 'PRODUCT', displayCount: 4 },
  new: { type: 'PRODUCT', displayCount: 4 },
  recommend: {
    type: 'RECOMMEND',
    displayCount: 2,
    brandName: 'SignalDecode', // 고정 (수정 불가)
  },
  review: { type: 'REVIEW', displayCount: 10 },
  half_banner: { type: 'HALF_BANNER' },
  full_banner: { type: 'FULL_BANNER' },
  slide_banner: { type: 'SLIDE_BANNER' },
  instagram: { type: 'INSTAGRAM', displayCount: 4 },
  category: {
    type: 'CATEGORY',
    displayCount: 4,
    categories: ['전체', '예시1', '예시2', '예시3', '예시4'],
    selectedCategory: '전체',
  },
}

// keywordCode → 프론트 id 매핑 (API에서 snake_case, 프론트에서 camelCase 사용)
const keywordCodeToId = {
  best: 'best',
  new: 'new',
  recommend: 'recommend',
  review: 'review',
  half_banner: 'halfBanner',
  full_banner: 'fullBanner',
  slide_banner: 'slideBanner',
  instagram: 'instagram',
  category: 'category',
}

// 전시 섹션 목록
const sections = ref([])

// 드래그 상태
const draggedIndex = ref(null)
const dragOverIndex = ref(null)

// 와이어프레임 프리뷰 토글 상태
const expandedSections = ref({})

// 초기 데이터 로드 완료 플래그 (중복 호출 방지)
const initialFetchDone = ref(false)

// 와이어프레임 토글
const toggleWireframe = (sectionId) => {
  expandedSections.value[sectionId] = !expandedSections.value[sectionId]
}

// 와이어프레임 확장 여부
const isWireframeExpanded = (sectionId) => {
  return expandedSections.value[sectionId] ?? false // 기본값: 접힘
}

// 섹션 활성화 토글
const toggleSection = (section) => {
  section.isActive = !section.isActive
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
    const items = [...sections.value]
    const [draggedItem] = items.splice(draggedIndex.value, 1)
    items.splice(index, 0, draggedItem)
    // sortOrder 재정렬
    items.forEach((item, idx) => {
      item.sortOrder = idx + 1
    })
    sections.value = items
  }
  draggedIndex.value = null
  dragOverIndex.value = null
}

// 드래그 종료
const onDragEnd = () => {
  draggedIndex.value = null
  dragOverIndex.value = null
}

// 이미지 데이터 파싱 헬퍼
const parseBannerImage = (item) => {
  // 1. bannerImage 객체가 있는 경우
  if (item.bannerImage && typeof item.bannerImage === 'object') {
    return {
      id: item.bannerImage.id,
      url: item.bannerImage.url,
      altText: item.bannerImage.altText || null,
    }
  }
  // 2. bannerImageUrl 문자열이 있는 경우
  if (item.bannerImageUrl) {
    return {
      id: null,
      url: item.bannerImageUrl,
      altText: null,
    }
  }
  // 3. 이미지 없음
  return null
}

// 데이터 로드
const fetchDisplayConfig = async () => {
  isLoading.value = true
  try {
    const response = await $api.get('/admin/displays')
    const apiData = response.data || []

    // API 데이터를 프론트 sections 형식으로 변환
    sections.value = apiData
      .sort((a, b) => a.sortOrder - b.sortOrder)
      .map(item => {
        const frontId = keywordCodeToId[item.keywordCode] || item.keywordCode
        const defaults = sectionDefaults[item.keywordCode] || {}

        return {
          // API 데이터
          apiId: item.id, // API의 실제 id (저장 시 필요)
          id: frontId, // 프론트에서 사용하는 id
          name: item.keyword,
          keywordCode: item.keywordCode,
          isActive: item.isActive ?? false,
          sortOrder: item.sortOrder,
          title: item.title || '',
          description: item.description || '',
          subTitle: item.subtitle || '', // API: subtitle → 프론트: subTitle
          // 이미지 객체 (id, url, altText) 또는 null
          bannerImage: parseBannerImage(item),
          bannerLink: item.linkUrl || '',
          // 프론트 기본값
          ...defaults,
        }
      })
  } catch (error) {
    uiStore.showToast({ type: 'error', message: '전시 설정을 불러오지 못했습니다.' })
  } finally {
    isLoading.value = false
  }
}

// 저장
const handleSave = async () => {
  // 중복 호출 방지
  if (isSaving.value) return
  isSaving.value = true
  try {
    // API 요청 페이로드 생성
    const payload = sections.value.map((section, index) => ({
      id: section.apiId,
      sortOrder: index + 1,
      isActive: section.isActive,
      title: section.title || null,
      description: section.description || null,
      subtitle: section.subTitle || null,
      linkUrl: section.bannerLink || null,
      // 기존 이미지 정보 (새 이미지가 아닌 경우에만)
      bannerImage: section.bannerImage && !section.bannerImage.isNew ? {
        id: section.bannerImage.id,
        url: section.bannerImage.url,
        altText: section.bannerImage.altText || null,
      } : null,
    }))

    // FormData 생성
    const formData = new FormData()
    formData.append('data', JSON.stringify(payload))

    // 새 이미지 파일이 있으면 추가 (recommend 섹션)
    const recommendSection = sections.value.find(s => s.id === 'recommend')
    if (recommendSection?.bannerImage?.isNew && recommendSection.bannerImage.file) {
      formData.append('bannerImage', recommendSection.bannerImage.file)
    }

    await $api.putFormData('/admin/displays', formData)

    // sortOrder 업데이트 및 이미지 상태 정리
    sections.value.forEach((section, index) => {
      section.sortOrder = index + 1
      // 새 이미지였던 것을 기존 이미지로 변환 (다음 저장 시 다시 업로드 방지)
      if (section.bannerImage?.isNew) {
        // 서버에서 새 이미지 정보를 받아오도록 다시 fetch
      }
    })

    uiStore.showToast({ type: 'success', message: '전시 설정이 저장되었습니다.' })

    // 저장 후 최신 데이터 다시 불러오기 (서버에서 이미지 URL 받기)
    await fetchDisplayConfig()
  } catch (error) {
    uiStore.showToast({ type: 'error', message: '저장에 실패했습니다.' })
  } finally {
    isSaving.value = false
  }
}

// 샘플 상품 데이터
const getSampleProducts = (count) => {
  return Array.from({ length: count }, (_, i) => ({
    id: i + 1,
    name: `Sample ${i + 1}`,
    discountRate: 63,
    price: '84,700원',
    originalPrice: '230,000원',
  }))
}

// 추천 섹션 이미지 업로드
const recommendFileInputRef = ref(null)
const currentRecommendSection = ref(null)

const triggerRecommendFileInput = (section) => {
  currentRecommendSection.value = section
  recommendFileInputRef.value?.click()
}

const handleRecommendImageSelect = (event) => {
  const file = event.target.files?.[0]
  if (!file) return

  // 파일 유효성 검사
  if (!file.type.startsWith('image/')) {
    uiStore.showToast({ type: 'error', message: '이미지 파일만 업로드 가능합니다.' })
    return
  }
  if (file.size > 10 * 1024 * 1024) {
    uiStore.showToast({ type: 'error', message: '파일 크기는 10MB 이하만 가능합니다.' })
    return
  }

  // 파일 객체와 미리보기 URL 저장 (저장 시점에 업로드)
  if (currentRecommendSection.value) {
    currentRecommendSection.value.bannerImage = {
      file, // 실제 파일 객체 (저장 시 FormData에 추가)
      preview: URL.createObjectURL(file), // 미리보기용 URL
      isNew: true, // 새로 추가된 이미지 표시
    }
  }

  // input 초기화
  if (recommendFileInputRef.value) {
    recommendFileInputRef.value.value = ''
  }
}

const removeRecommendImage = (section) => {
  // 미리보기 URL 해제
  if (section.bannerImage?.preview && section.bannerImage?.isNew) {
    URL.revokeObjectURL(section.bannerImage.preview)
  }
  section.bannerImage = null
}

// 샘플 리뷰 데이터
const getSampleReviews = (count) => {
  return Array.from({ length: count }, (_, i) => ({
    id: i + 1,
    rating: 5,
    content: '정말 좋아요! 피부가 촉촉해졌어요.',
    author: `user${i + 1}`,
    productName: `Sample Product ${i + 1}`,
  }))
}

onMounted(() => {
  // 중복 호출 방지 (Nuxt의 Suspense로 인해 2번 마운트될 수 있음)
  if (initialFetchDone.value) return
  initialFetchDone.value = true
  fetchDisplayConfig()
})
</script>

<template>
  <LayoutFormPage
    title="메인 전시"
    description="메인 페이지에 표시될 전시 영역을 설정합니다."
    :is-saving="isSaving"
    @save="handleSave"
  >
    <div v-if="isLoading" class="flex items-center justify-center py-20">
      <UiSpinner size="lg" />
    </div>

    <div v-else class="space-y-6">
      <!-- 숨겨진 파일 input (전역) -->
      <input
        ref="recommendFileInputRef"
        type="file"
        accept="image/*"
        class="hidden"
        @change="handleRecommendImageSelect"
      >

      <!-- 안내 -->
      <div class="bg-primary-50 border border-primary-200 rounded-lg p-4">
        <div class="flex items-start gap-3">
          <svg class="w-5 h-5 text-primary-600 mt-0.5 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
          </svg>
          <div>
            <p class="text-sm font-medium text-primary-900">전시 영역 관리</p>
            <p class="text-sm text-primary-700 mt-1">
              드래그하여 순서를 변경하고, 타이틀과 설명을 직접 입력할 수 있습니다.
              활성화된 섹션만 메인 페이지에 표시됩니다.
            </p>
          </div>
        </div>
      </div>

      <!-- 섹션 목록 -->
      <div class="space-y-4">
        <div
          v-for="(section, index) in sections"
          :key="section.id"
          draggable="true"
          :class="[
            'bg-white border rounded-xl overflow-hidden transition-all',
            dragOverIndex === index ? 'border-primary-400 shadow-lg' : 'border-neutral-200',
            draggedIndex === index ? 'opacity-50' : '',
          ]"
          @dragstart="onDragStart($event, index)"
          @dragover="onDragOver($event, index)"
          @dragleave="onDragLeave"
          @drop="onDrop($event, index)"
          @dragend="onDragEnd"
        >
          <!-- 카드 헤더: 순서변경 핸들 + 순서 + 섹션명 + 미리보기 토글 + 활성화 토글 -->
          <div class="flex items-center gap-3 md:gap-4 px-4 py-3 bg-neutral-50 border-b border-neutral-200">
            <!-- 드래그 핸들 -->
            <div class="cursor-move text-neutral-400 hover:text-neutral-600">
              <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 8h16M4 16h16" />
              </svg>
            </div>

            <!-- 순서 번호 -->
            <div class="w-8 h-8 flex items-center justify-center bg-white border border-neutral-200 rounded-lg text-sm font-semibold text-neutral-700">
              {{ index + 1 }}
            </div>

            <!-- 섹션명 -->
            <div class="flex-1 min-w-0">
              <span :class="['font-medium', section.isActive ? 'text-neutral-900' : 'text-neutral-500']">
                {{ section.name }}
              </span>
              <UiBadge v-if="!section.isActive" variant="neutral" size="sm" class="ml-2">비활성</UiBadge>
            </div>

            <!-- 미리보기 토글 버튼 -->
            <button
              type="button"
              class="flex items-center gap-1.5 px-2 py-1.5 text-sm text-neutral-500 hover:text-neutral-700 hover:bg-neutral-100 rounded-lg transition-colors"
              @click.stop="toggleWireframe(section.id)"
            >
              <svg
                :class="[
                  'w-4 h-4 transition-transform duration-200',
                  isWireframeExpanded(section.id) ? 'rotate-180' : ''
                ]"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
              >
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
              </svg>
              <span class="hidden md:inline">미리보기</span>
            </button>

            <!-- 활성화 토글 -->
            <div class="flex items-center gap-2">
              <span class="hidden md:inline text-sm text-neutral-500">활성화</span>
              <button
                type="button"
                role="switch"
                :aria-checked="section.isActive"
                :class="[
                  'relative inline-flex h-6 w-11 flex-shrink-0 cursor-pointer rounded-full border-2 border-transparent transition-colors duration-200 ease-in-out focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2',
                  section.isActive ? 'bg-primary-600' : 'bg-neutral-200',
                ]"
                @click.stop="toggleSection(section)"
              >
                <span
                  :class="[
                    'pointer-events-none inline-block h-5 w-5 transform rounded-full bg-white shadow ring-0 transition duration-200 ease-in-out',
                    section.isActive ? 'translate-x-5' : 'translate-x-0',
                  ]"
                />
              </button>
            </div>
          </div>

          <!-- 와이어프레임 미리보기 영역 (토글 가능) -->
          <div
            v-show="isWireframeExpanded(section.id)"
            class="p-4 md:p-6 transition-all"
          >
            <div class="max-w-[1200px]">
            <!-- 베스트 상품 섹션 와이어프레임 -->
            <template v-if="section.id === 'best'">
              <div class="bg-neutral-50 rounded-lg p-4 md:p-6 border border-neutral-200">
                <!-- 섹션 헤더: 타이틀 + 설명 + 네비게이션 -->
                <div class="flex flex-col md:flex-row md:items-end justify-between gap-3 md:gap-4 mb-6">
                  <div class="flex flex-col md:flex-row gap-3 md:gap-4">
                    <!-- 타이틀 입력 -->
                    <div>
                      <label class="block text-xs text-neutral-500 mb-1">타이틀</label>
                      <input
                        v-model="section.title"
                        type="text"
                        class="w-full md:w-48 px-3 py-2 text-lg font-bold text-neutral-900 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                        placeholder="타이틀 입력"
                      >
                    </div>
                    <!-- 설명 입력 -->
                    <div>
                      <label class="block text-xs text-neutral-500 mb-1">설명</label>
                      <input
                        v-model="section.description"
                        type="text"
                        class="w-full md:w-56 px-3 py-2 text-sm text-neutral-600 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                        placeholder="설명 입력"
                      >
                    </div>
                  </div>
                  <!-- 네비게이션 버튼 (데스크탑에서만 표시) -->
                  <div class="hidden md:flex items-center gap-2">
                    <button type="button" class="w-10 h-10 flex items-center justify-center rounded-full border border-neutral-300 text-neutral-400 hover:border-neutral-400 hover:text-neutral-500">
                      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
                      </svg>
                    </button>
                    <button type="button" class="w-10 h-10 flex items-center justify-center rounded-full border border-neutral-300 text-neutral-400 hover:border-neutral-400 hover:text-neutral-500">
                      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                      </svg>
                    </button>
                  </div>
                </div>

                <!-- 상품 카드: 모바일 슬라이드 / 데스크탑 그리드 -->
                <div class="md:hidden overflow-x-auto scrollbar-hide -mx-4 px-4">
                  <div class="flex gap-3 snap-x snap-mandatory" style="width: max-content;">
                    <div
                      v-for="product in getSampleProducts(4)"
                      :key="product.id"
                      class="w-40 flex-shrink-0 snap-start bg-white rounded-lg overflow-hidden border border-neutral-200"
                    >
                      <!-- 이미지 영역 (회색 배경) - 정방형 -->
                      <div class="relative aspect-square bg-neutral-200">
                        <!-- BEST 뱃지 -->
                        <span class="absolute top-2 left-2 px-2 py-0.5 bg-neutral-800 text-white text-xs font-semibold rounded">
                          BEST
                        </span>
                        <!-- 이미지 플레이스홀더 아이콘 -->
                        <div class="absolute inset-0 flex items-center justify-center">
                          <svg class="w-10 h-10 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                          </svg>
                        </div>
                      </div>
                      <!-- 상품 정보 -->
                      <div class="p-2.5">
                        <p class="text-xs text-neutral-900 font-medium mb-1.5 truncate">{{ product.name }}</p>
                        <div class="border-t border-neutral-100 pt-1.5">
                          <div class="flex items-center gap-1 flex-wrap">
                            <span class="px-1 py-0.5 bg-primary-600 text-white text-xs font-bold rounded">{{ product.discountRate }}%</span>
                            <span class="text-xs font-bold text-neutral-900">{{ product.price }}</span>
                          </div>
                          <span class="text-xs text-neutral-400 line-through">{{ product.originalPrice }}</span>
                        </div>
                      </div>
                    </div>
                  </div>
                  <!-- 슬라이드 힌트 -->
                  <p class="text-xs text-neutral-400 text-center mt-3">← 터치하여 슬라이드 →</p>
                </div>

                <!-- 데스크탑 그리드 -->
                <div class="hidden md:grid grid-cols-4 gap-4">
                  <div
                    v-for="product in getSampleProducts(4)"
                    :key="product.id"
                    class="bg-white rounded-lg overflow-hidden border border-neutral-200"
                  >
                    <!-- 이미지 영역 (회색 배경) - 정방형 -->
                    <div class="relative aspect-square bg-neutral-200">
                      <!-- BEST 뱃지 -->
                      <span class="absolute top-3 left-3 px-2.5 py-1 bg-neutral-800 text-white text-xs font-semibold rounded">
                        BEST
                      </span>
                      <!-- 이미지 플레이스홀더 아이콘 -->
                      <div class="absolute inset-0 flex items-center justify-center">
                        <svg class="w-12 h-12 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                      </div>
                    </div>
                    <!-- 상품 정보 -->
                    <div class="p-3">
                      <p class="text-sm text-neutral-900 font-medium mb-2 truncate">{{ product.name }}</p>
                      <div class="border-t border-neutral-100 pt-2">
                        <div class="flex items-center gap-2">
                          <span class="px-1.5 py-0.5 bg-primary-600 text-white text-xs font-bold rounded">{{ product.discountRate }}%</span>
                          <span class="text-sm font-bold text-neutral-900">{{ product.price }}</span>
                          <span class="text-xs text-neutral-400 line-through">{{ product.originalPrice }}</span>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </template>

            <!-- 추천 상품 섹션 와이어프레임 -->
            <template v-else-if="section.id === 'recommend'">
              <div class="bg-neutral-50 rounded-lg p-4 border border-neutral-200">
                <!-- 섹션 헤더: 타이틀 + 설명 -->
                <div class="flex flex-col md:flex-row gap-2 md:gap-3 mb-4">
                  <div>
                    <label class="block text-xs text-neutral-500 mb-1">타이틀</label>
                    <input
                      v-model="section.title"
                      type="text"
                      class="w-full md:w-40 px-2.5 py-1.5 text-base font-bold text-neutral-900 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                      placeholder="타이틀 입력"
                    >
                  </div>
                  <div>
                    <label class="block text-xs text-neutral-500 mb-1">설명</label>
                    <input
                      v-model="section.description"
                      type="text"
                      class="w-full md:w-48 px-2.5 py-1.5 text-sm text-neutral-600 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                      placeholder="설명 입력"
                    >
                  </div>
                </div>

                <!-- 데스크탑: 2컬럼 레이아웃 -->
                <div class="hidden md:flex gap-8">
                  <!-- 좌측: 배너 이미지 영역 (파일 업로드) + 링크 -->
                  <div class="w-[450px] flex-shrink-0">
                    <label class="block text-xs text-neutral-500 mb-1">배너 이미지</label>

                    <!-- 이미지가 있을 때 -->
                    <div
                      v-if="section.bannerImage?.url || section.bannerImage?.preview"
                      class="relative w-[450px] h-[450px] bg-neutral-100 rounded-lg overflow-hidden border border-neutral-200 group"
                    >
                      <img
                        :src="section.bannerImage.preview || section.bannerImage.url"
                        alt="배너 이미지"
                        class="w-full h-full object-cover"
                      >
                      <div class="absolute inset-0 bg-black/50 opacity-0 group-hover:opacity-100 transition-opacity flex items-center justify-center gap-2">
                        <button
                          type="button"
                          class="px-3 py-1.5 bg-white text-neutral-700 text-sm font-medium rounded-lg hover:bg-neutral-100"
                          @click="triggerRecommendFileInput(section)"
                        >
                          변경
                        </button>
                        <button
                          type="button"
                          class="px-3 py-1.5 bg-error-500 text-white text-sm font-medium rounded-lg hover:bg-error-600"
                          @click="removeRecommendImage(section)"
                        >
                          삭제
                        </button>
                      </div>
                    </div>

                    <!-- 이미지가 없을 때 (업로드 영역) -->
                    <div
                      v-else
                      class="w-[450px] h-[450px] bg-white rounded-lg border-2 border-dashed border-neutral-300 hover:border-primary-400 cursor-pointer transition-colors flex items-center justify-center"
                      @click="triggerRecommendFileInput(section)"
                    >
                      <div class="text-center p-4">
                        <svg class="w-12 h-12 mx-auto mb-2 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                        <p class="text-sm font-medium text-neutral-600">클릭하여 업로드</p>
                      </div>
                    </div>

                    <!-- 링크 입력 -->
                    <div class="mt-3">
                      <label class="block text-xs text-neutral-500 mb-1">새창으로 이동할 링크</label>
                      <input
                        v-model="section.bannerLink"
                        type="url"
                        class="w-full px-3 py-2 text-sm text-neutral-900 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                        placeholder="https://example.com"
                      >
                    </div>
                  </div>

                  <!-- 우측: 브랜드명 + 서브타이틀 + 상품 카드 -->
                  <div class="flex-1 flex flex-col items-center">
                    <!-- 브랜드명 (고정) + 서브타이틀 (편집 가능) -->
                    <div class="mb-4">
                      <div class="text-center mb-2">
                        <span class="inline-block px-3 py-1 bg-neutral-200 text-neutral-500 text-xs rounded-full">
                          {{ section.brandName }}
                        </span>
                      </div>
                      <div>
                        <label class="block text-xs text-neutral-500 mb-1 text-center">서브 타이틀</label>
                        <input
                          v-model="section.subTitle"
                          type="text"
                          class="w-full px-3 py-1.5 text-base font-bold text-neutral-900 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none text-center"
                          placeholder="서브 타이틀 입력"
                        >
                      </div>
                    </div>

                    <!-- 상품 카드 2개 + 네비게이션 -->
                    <div class="flex-1 flex flex-col justify-center">
                      <div class="relative">
                        <!-- 네비게이션 버튼 (좌) -->
                        <button
                          type="button"
                          class="flex absolute -left-3 top-1/2 -translate-y-1/2 z-10 w-8 h-8 items-center justify-center rounded-full border border-neutral-300 bg-white text-neutral-400 hover:border-neutral-400 hover:text-neutral-500 shadow-sm"
                        >
                          <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
                          </svg>
                        </button>

                        <!-- 상품 카드 -->
                        <div class="flex gap-3">
                          <div
                            v-for="product in getSampleProducts(2)"
                            :key="product.id"
                            class="w-[250px] flex-shrink-0 bg-white rounded-lg overflow-hidden border border-neutral-200"
                          >
                            <!-- 이미지 영역 -->
                            <div class="aspect-square bg-neutral-200 flex items-center justify-center">
                              <svg class="w-10 h-10 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                              </svg>
                            </div>
                            <!-- 상품 정보 -->
                            <div class="p-2.5">
                              <p class="text-sm text-neutral-900 font-medium mb-1 truncate">{{ product.name }}</p>
                              <div class="flex items-center gap-1 flex-wrap">
                                <span class="px-1 py-0.5 bg-primary-600 text-white text-xs font-bold rounded">{{ product.discountRate }}%</span>
                                <span class="text-sm font-bold text-neutral-900">{{ product.price }}</span>
                              </div>
                            </div>
                          </div>
                        </div>

                        <!-- 네비게이션 버튼 (우) -->
                        <button
                          type="button"
                          class="flex absolute -right-3 top-1/2 -translate-y-1/2 z-10 w-8 h-8 items-center justify-center rounded-full border border-neutral-300 bg-white text-neutral-400 hover:border-neutral-400 hover:text-neutral-500 shadow-sm"
                        >
                          <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                          </svg>
                        </button>
                      </div>
                    </div>
                  </div>
                </div>

                <!-- 모바일: 세로 레이아웃 (타이틀/설명 → 이미지 → 카드) -->
                <div class="md:hidden space-y-4">
                  <!-- 배너 이미지 -->
                  <div>
                    <label class="block text-xs text-neutral-500 mb-1">배너 이미지</label>

                    <!-- 이미지가 있을 때 -->
                    <div
                      v-if="section.bannerImage?.url || section.bannerImage?.preview"
                      class="relative aspect-square bg-neutral-100 rounded-lg overflow-hidden border border-neutral-200"
                    >
                      <img
                        :src="section.bannerImage.preview || section.bannerImage.url"
                        alt="배너 이미지"
                        class="w-full h-full object-cover"
                      >
                      <div class="absolute bottom-2 right-2 flex gap-2">
                        <button
                          type="button"
                          class="px-2.5 py-1 bg-white/90 text-neutral-700 text-xs font-medium rounded-lg"
                          @click="triggerRecommendFileInput(section)"
                        >
                          변경
                        </button>
                        <button
                          type="button"
                          class="px-2.5 py-1 bg-error-500/90 text-white text-xs font-medium rounded-lg"
                          @click="removeRecommendImage(section)"
                        >
                          삭제
                        </button>
                      </div>
                    </div>

                    <!-- 이미지가 없을 때 (업로드 영역) -->
                    <div
                      v-else
                      class="aspect-square bg-white rounded-lg border-2 border-dashed border-neutral-300 hover:border-primary-400 cursor-pointer transition-colors flex items-center justify-center"
                      @click="triggerRecommendFileInput(section)"
                    >
                      <div class="text-center p-4">
                        <svg class="w-12 h-12 mx-auto mb-3 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                        <p class="text-sm font-medium text-neutral-600">클릭하여 이미지 업로드</p>
                        <p class="text-xs text-neutral-400 mt-1">권장: 정방형 (1:1)</p>
                      </div>
                    </div>

                    <!-- 링크 입력 -->
                    <div class="mt-3">
                      <label class="block text-xs text-neutral-500 mb-1">새창으로 이동할 링크</label>
                      <input
                        v-model="section.bannerLink"
                        type="url"
                        class="w-full px-3 py-2 text-sm text-neutral-900 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                        placeholder="https://example.com"
                      >
                    </div>
                  </div>

                  <!-- 브랜드명 + 서브타이틀 + 상품 카드 -->
                  <div>
                    <!-- 브랜드명 (고정) + 서브타이틀 -->
                    <div class="mb-4">
                      <div class="text-center mb-3">
                        <span class="inline-block px-3 py-1 bg-neutral-200 text-neutral-500 text-xs rounded-full">
                          {{ section.brandName }} (고정)
                        </span>
                      </div>
                      <div>
                        <label class="block text-xs text-neutral-500 mb-1 text-center">서브 타이틀</label>
                        <input
                          v-model="section.subTitle"
                          type="text"
                          class="w-full px-3 py-2 text-lg font-bold text-neutral-900 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none text-center"
                          placeholder="서브 타이틀 입력"
                        >
                      </div>
                    </div>

                    <!-- 상품 카드 2개 (가로 슬라이드) -->
                    <div class="overflow-x-auto scrollbar-hide -mx-4 px-4">
                      <div class="flex gap-3 snap-x snap-mandatory" style="width: max-content;">
                        <div
                          v-for="product in getSampleProducts(2)"
                          :key="product.id"
                          class="w-36 flex-shrink-0 snap-start bg-white rounded-lg overflow-hidden border border-neutral-200"
                        >
                          <!-- 이미지 영역 - 정방형 -->
                          <div class="aspect-square bg-neutral-200 flex items-center justify-center">
                            <svg class="w-8 h-8 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                            </svg>
                          </div>
                          <!-- 상품 정보 -->
                          <div class="p-2">
                            <p class="text-xs text-neutral-900 font-medium mb-1 truncate">{{ product.name }}</p>
                            <div class="flex items-center gap-1 flex-wrap">
                              <span class="px-1 py-0.5 bg-primary-600 text-white text-xs font-bold rounded">{{ product.discountRate }}%</span>
                              <span class="text-xs font-bold text-neutral-900">{{ product.price }}</span>
                            </div>
                          </div>
                        </div>
                      </div>
                      <!-- 슬라이드 힌트 -->
                      <p class="text-xs text-neutral-400 text-center mt-3">← 터치하여 슬라이드 →</p>
                    </div>
                  </div>
                </div>
              </div>
            </template>

            <!-- 리뷰 섹션 와이어프레임 -->
            <template v-else-if="section.id === 'review'">
              <div class="bg-neutral-50 rounded-lg p-4 md:p-6 border border-neutral-200">
                <!-- 섹션 헤더: 타이틀 + 설명 (중앙정렬) -->
                <div class="text-center mb-6">
                  <div class="flex flex-col items-center gap-2">
                    <div>
                      <label class="block text-xs text-neutral-500 mb-1">타이틀</label>
                      <input
                        v-model="section.title"
                        type="text"
                        class="w-48 px-3 py-2 text-lg font-bold text-neutral-900 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none text-center"
                        placeholder="타이틀 입력"
                      >
                    </div>
                    <div>
                      <label class="block text-xs text-neutral-500 mb-1">설명</label>
                      <input
                        v-model="section.description"
                        type="text"
                        class="w-64 px-3 py-2 text-sm text-neutral-600 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none text-center"
                        placeholder="설명 입력"
                      >
                    </div>
                  </div>
                </div>

                <!-- 리뷰 카드: 모바일 슬라이드 -->
                <div class="md:hidden overflow-x-auto scrollbar-hide -mx-4 px-4">
                  <div class="flex gap-3 snap-x snap-mandatory" style="width: max-content;">
                    <div
                      v-for="review in getSampleReviews(5)"
                      :key="review.id"
                      class="w-32 flex-shrink-0 snap-start bg-white rounded-lg overflow-hidden border border-neutral-200 p-3"
                    >
                      <!-- 이미지 영역 -->
                      <div class="aspect-square bg-neutral-200 rounded-lg mb-2 flex items-center justify-center">
                        <svg class="w-6 h-6 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                      </div>
                      <!-- 별점 -->
                      <div class="flex gap-0.5 mb-1">
                        <svg v-for="i in 5" :key="i" class="w-3 h-3 text-yellow-400" fill="currentColor" viewBox="0 0 20 20">
                          <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z" />
                        </svg>
                      </div>
                      <!-- 리뷰 내용 -->
                      <p class="text-xs text-neutral-600 line-clamp-2 mb-1">{{ review.content }}</p>
                      <p class="text-xs text-neutral-400 truncate">{{ review.author }}</p>
                    </div>
                  </div>
                  <p class="text-xs text-neutral-400 text-center mt-3">← 터치하여 슬라이드 →</p>
                </div>

                <!-- 리뷰 카드: 데스크탑 그리드 (5개씩 2줄) -->
                <div class="hidden md:block space-y-3">
                  <div class="grid grid-cols-5 gap-3">
                    <div
                      v-for="review in getSampleReviews(5)"
                      :key="review.id"
                      class="bg-white rounded-lg overflow-hidden border border-neutral-200 p-3"
                    >
                      <!-- 이미지 영역 -->
                      <div class="aspect-square bg-neutral-200 rounded-lg mb-2 flex items-center justify-center">
                        <svg class="w-8 h-8 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                      </div>
                      <!-- 별점 -->
                      <div class="flex gap-0.5 mb-1">
                        <svg v-for="i in 5" :key="i" class="w-3.5 h-3.5 text-yellow-400" fill="currentColor" viewBox="0 0 20 20">
                          <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z" />
                        </svg>
                      </div>
                      <!-- 리뷰 내용 -->
                      <p class="text-xs text-neutral-600 line-clamp-2 mb-1">{{ review.content }}</p>
                      <p class="text-xs text-neutral-400 truncate">{{ review.author }}</p>
                    </div>
                  </div>
                  <div class="grid grid-cols-5 gap-3">
                    <div
                      v-for="review in getSampleReviews(5).map((r, i) => ({ ...r, id: r.id + 5 }))"
                      :key="review.id"
                      class="bg-white rounded-lg overflow-hidden border border-neutral-200 p-3"
                    >
                      <!-- 이미지 영역 -->
                      <div class="aspect-square bg-neutral-200 rounded-lg mb-2 flex items-center justify-center">
                        <svg class="w-8 h-8 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                      </div>
                      <!-- 별점 -->
                      <div class="flex gap-0.5 mb-1">
                        <svg v-for="i in 5" :key="i" class="w-3.5 h-3.5 text-yellow-400" fill="currentColor" viewBox="0 0 20 20">
                          <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z" />
                        </svg>
                      </div>
                      <!-- 리뷰 내용 -->
                      <p class="text-xs text-neutral-600 line-clamp-2 mb-1">{{ review.content }}</p>
                      <p class="text-xs text-neutral-400 truncate">{{ review.author }}</p>
                    </div>
                  </div>
                </div>
              </div>
            </template>

            <!-- 인스타그램 섹션 와이어프레임 -->
            <template v-else-if="section.id === 'instagram'">
              <div class="bg-neutral-50 rounded-lg p-4 md:p-6 border border-neutral-200">
                <!-- 안내 문구 -->
                <div class="flex items-start gap-2 mb-4 p-3 bg-amber-50 border border-amber-200 rounded-lg">
                  <svg class="w-4 h-4 text-amber-500 flex-shrink-0 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
                  </svg>
                  <div class="text-sm text-amber-800">
                    <p class="font-medium">Instagram API 연동 필수</p>
                    <p class="text-amber-700 mt-0.5">액세스 토큰은 60일마다 재발급이 필요합니다.</p>
                  </div>
                </div>

                <!-- 섹션 헤더: 타이틀 + 설명 (고정) + 네비게이션 -->
                <div class="flex flex-col md:flex-row md:items-end justify-between gap-3 md:gap-4 mb-6">
                  <div class="flex flex-col md:flex-row md:items-baseline gap-1 md:gap-3">
                    <p class="text-lg font-bold text-neutral-900">Instagram</p>
                    <p class="text-sm text-neutral-600">@instagramId</p>
                  </div>
                  <!-- 네비게이션 버튼 (데스크탑에서만 표시) -->
                  <div class="hidden md:flex items-center gap-2">
                    <button type="button" class="w-10 h-10 flex items-center justify-center rounded-full border border-neutral-300 text-neutral-400 hover:border-neutral-400 hover:text-neutral-500">
                      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
                      </svg>
                    </button>
                    <button type="button" class="w-10 h-10 flex items-center justify-center rounded-full border border-neutral-300 text-neutral-400 hover:border-neutral-400 hover:text-neutral-500">
                      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                      </svg>
                    </button>
                  </div>
                </div>

                <!-- 이미지 카드: 모바일 슬라이드 -->
                <div class="md:hidden overflow-x-auto scrollbar-hide -mx-4 px-4">
                  <div class="flex gap-3 snap-x snap-mandatory" style="width: max-content;">
                    <div
                      v-for="i in 4"
                      :key="i"
                      class="w-32 flex-shrink-0 snap-start"
                    >
                      <!-- 이미지 영역 (4:6 비율) -->
                      <div class="aspect-[4/6] bg-neutral-200 rounded-lg flex items-center justify-center">
                        <svg class="w-8 h-8 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                      </div>
                    </div>
                  </div>
                  <!-- 슬라이드 힌트 -->
                  <p class="text-xs text-neutral-400 text-center mt-3">← 터치하여 슬라이드 →</p>
                </div>

                <!-- 이미지 카드: 데스크탑 그리드 -->
                <div class="hidden md:grid grid-cols-4 gap-4">
                  <div
                    v-for="i in 4"
                    :key="i"
                  >
                    <!-- 이미지 영역 (4:6 비율) -->
                    <div class="aspect-[4/6] bg-neutral-200 rounded-lg flex items-center justify-center">
                      <svg class="w-12 h-12 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                      </svg>
                    </div>
                  </div>
                </div>
              </div>
            </template>

            <!-- 하프배너 섹션 와이어프레임 -->
            <template v-else-if="section.id === 'halfBanner'">
              <div class="bg-neutral-50 rounded-lg p-4 md:p-6 border border-neutral-200">
                <!-- 안내 문구 -->
                <div class="flex items-center gap-2 mb-4 p-3 bg-blue-50 border border-blue-200 rounded-lg">
                  <svg class="w-4 h-4 text-blue-500 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                  </svg>
                  <p class="text-sm text-blue-700">디자인관리 > 배너관리 에서 하프배너를 등록 후 사용하세요.</p>
                </div>

                <!-- 데스크탑: 2컬럼 레이아웃 (좌우) -->
                <div class="hidden md:grid grid-cols-2 gap-4">
                  <!-- 좌측 배너 영역 -->
                  <div class="aspect-square bg-neutral-300 rounded-lg flex items-center justify-center">
                    <div class="text-center">
                      <svg class="w-12 h-12 mx-auto mb-2 text-neutral-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                      </svg>
                      <p class="text-sm font-medium text-neutral-500">좌측 배너</p>
                    </div>
                  </div>
                  <!-- 우측 배너 영역 -->
                  <div class="aspect-square bg-neutral-400 rounded-lg flex items-center justify-center">
                    <div class="text-center">
                      <svg class="w-12 h-12 mx-auto mb-2 text-neutral-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                      </svg>
                      <p class="text-sm font-medium text-neutral-600">우측 배너</p>
                    </div>
                  </div>
                </div>

                <!-- 모바일: 세로 레이아웃 (위아래) -->
                <div class="md:hidden space-y-3">
                  <!-- 상단 배너 영역 -->
                  <div class="aspect-square bg-neutral-300 rounded-lg flex items-center justify-center">
                    <div class="text-center">
                      <svg class="w-10 h-10 mx-auto mb-2 text-neutral-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                      </svg>
                      <p class="text-sm font-medium text-neutral-500">상단 배너</p>
                    </div>
                  </div>
                  <!-- 하단 배너 영역 -->
                  <div class="aspect-square bg-neutral-400 rounded-lg flex items-center justify-center">
                    <div class="text-center">
                      <svg class="w-10 h-10 mx-auto mb-2 text-neutral-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                      </svg>
                      <p class="text-sm font-medium text-neutral-600">하단 배너</p>
                    </div>
                  </div>
                </div>
              </div>
            </template>

            <!-- 전체배너 섹션 와이어프레임 -->
            <template v-else-if="section.id === 'fullBanner'">
              <div class="bg-neutral-50 rounded-lg p-4 md:p-6 border border-neutral-200">
                <!-- 안내 문구 -->
                <div class="flex items-center gap-2 mb-4 p-3 bg-blue-50 border border-blue-200 rounded-lg">
                  <svg class="w-4 h-4 text-blue-500 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                  </svg>
                  <p class="text-sm text-blue-700">디자인관리 > 배너관리 에서 전체배너를 등록 후 사용하세요.</p>
                </div>

                <!-- 전체 배너 영역 (16:9) -->
                <div class="aspect-video bg-neutral-300 rounded-lg flex items-center justify-center">
                  <div class="text-center">
                    <svg class="w-16 h-16 mx-auto mb-3 text-neutral-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                    </svg>
                    <p class="text-sm font-medium text-neutral-500">전체 배너 (16:9)</p>
                  </div>
                </div>
              </div>
            </template>

            <!-- 슬라이드배너 섹션 와이어프레임 -->
            <template v-else-if="section.id === 'slideBanner'">
              <div class="bg-neutral-50 rounded-lg p-4 md:p-6 border border-neutral-200">
                <!-- 안내 문구 -->
                <div class="flex items-center gap-2 mb-4 p-3 bg-blue-50 border border-blue-200 rounded-lg">
                  <svg class="w-4 h-4 text-blue-500 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                  </svg>
                  <p class="text-sm text-blue-700">디자인관리 > 배너관리 에서 슬라이드배너를 등록 후 사용하세요.</p>
                </div>

                <!-- 데스크탑: 5.5:1 비율 -->
                <div class="hidden md:block">
                  <div class="relative" style="aspect-ratio: 5.5 / 1;">
                    <div class="absolute inset-0 bg-neutral-300 rounded-lg flex items-center justify-center">
                      <div class="text-center">
                        <svg class="w-12 h-12 mx-auto mb-2 text-neutral-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                        <p class="text-sm font-medium text-neutral-500">슬라이드 배너 (5.5:1)</p>
                      </div>
                    </div>
                    <!-- 슬라이드 인디케이터 -->
                    <div class="absolute bottom-3 left-1/2 -translate-x-1/2 flex gap-1.5">
                      <span class="w-2 h-2 rounded-full bg-neutral-600"></span>
                      <span class="w-2 h-2 rounded-full bg-neutral-400"></span>
                      <span class="w-2 h-2 rounded-full bg-neutral-400"></span>
                    </div>
                  </div>
                </div>

                <!-- 모바일: 4:1 비율 -->
                <div class="md:hidden">
                  <div class="relative" style="aspect-ratio: 4 / 1;">
                    <div class="absolute inset-0 bg-neutral-300 rounded-lg flex items-center justify-center">
                      <div class="text-center">
                        <svg class="w-8 h-8 mx-auto mb-1 text-neutral-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                        <p class="text-xs font-medium text-neutral-500">슬라이드 배너 (4:1)</p>
                      </div>
                    </div>
                    <!-- 슬라이드 인디케이터 -->
                    <div class="absolute bottom-2 left-1/2 -translate-x-1/2 flex gap-1">
                      <span class="w-1.5 h-1.5 rounded-full bg-neutral-600"></span>
                      <span class="w-1.5 h-1.5 rounded-full bg-neutral-400"></span>
                      <span class="w-1.5 h-1.5 rounded-full bg-neutral-400"></span>
                    </div>
                  </div>
                </div>
              </div>
            </template>

            <!-- 카테고리별 상품 섹션 와이어프레임 -->
            <template v-else-if="section.id === 'category'">
              <div class="bg-neutral-50 rounded-lg p-4 md:p-6 border border-neutral-200">
                <!-- 섹션 헤더: 타이틀 + 설명 + 네비게이션 -->
                <div class="flex flex-col md:flex-row md:items-end justify-between gap-3 md:gap-4 mb-4">
                  <div class="flex flex-col md:flex-row gap-3 md:gap-4">
                    <div>
                      <label class="block text-xs text-neutral-500 mb-1">타이틀</label>
                      <input
                        v-model="section.title"
                        type="text"
                        class="w-full md:w-48 px-3 py-2 text-lg font-bold text-neutral-900 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                        placeholder="타이틀 입력"
                      >
                    </div>
                    <div>
                      <label class="block text-xs text-neutral-500 mb-1">설명</label>
                      <input
                        v-model="section.description"
                        type="text"
                        class="w-full md:w-56 px-3 py-2 text-sm text-neutral-600 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                        placeholder="설명 입력"
                      >
                    </div>
                  </div>
                  <!-- 네비게이션 버튼 (데스크탑) -->
                  <div class="hidden md:flex items-center gap-2">
                    <button type="button" class="w-10 h-10 flex items-center justify-center rounded-full border border-neutral-300 text-neutral-400 hover:border-neutral-400 hover:text-neutral-500">
                      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
                      </svg>
                    </button>
                    <button type="button" class="w-10 h-10 flex items-center justify-center rounded-full border border-neutral-300 text-neutral-400 hover:border-neutral-400 hover:text-neutral-500">
                      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                      </svg>
                    </button>
                  </div>
                </div>

                <!-- 카테고리 칩 -->
                <div class="mb-4">
                  <label class="block text-xs text-neutral-500 mb-2">카테고리 필터</label>
                  <div class="flex flex-wrap gap-2">
                    <button
                      v-for="cat in section.categories"
                      :key="cat"
                      type="button"
                      :class="[
                        'px-3 py-1.5 text-sm font-medium rounded-full border transition-colors',
                        section.selectedCategory === cat
                          ? 'bg-neutral-900 text-white border-neutral-900'
                          : 'bg-white text-neutral-600 border-neutral-300 hover:border-neutral-400'
                      ]"
                      @click="section.selectedCategory = cat"
                    >
                      {{ cat }}
                    </button>
                  </div>
                </div>

                <!-- 상품 카드: 모바일 슬라이드 -->
                <div class="md:hidden overflow-x-auto scrollbar-hide -mx-4 px-4">
                  <div class="flex gap-3 snap-x snap-mandatory" style="width: max-content;">
                    <div
                      v-for="product in getSampleProducts(4)"
                      :key="product.id"
                      class="w-40 flex-shrink-0 snap-start bg-white rounded-lg overflow-hidden border border-neutral-200"
                    >
                      <div class="relative aspect-square bg-neutral-200">
                        <div class="absolute inset-0 flex items-center justify-center">
                          <svg class="w-10 h-10 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                          </svg>
                        </div>
                      </div>
                      <div class="p-2.5">
                        <p class="text-xs text-neutral-900 font-medium mb-1.5 truncate">{{ product.name }}</p>
                        <div class="border-t border-neutral-100 pt-1.5">
                          <div class="flex items-center gap-1 flex-wrap">
                            <span class="px-1 py-0.5 bg-primary-600 text-white text-xs font-bold rounded">{{ product.discountRate }}%</span>
                            <span class="text-xs font-bold text-neutral-900">{{ product.price }}</span>
                          </div>
                          <span class="text-xs text-neutral-400 line-through">{{ product.originalPrice }}</span>
                        </div>
                      </div>
                    </div>
                  </div>
                  <p class="text-xs text-neutral-400 text-center mt-3">← 터치하여 슬라이드 →</p>
                </div>

                <!-- 상품 카드: 데스크탑 그리드 -->
                <div class="hidden md:grid grid-cols-4 gap-4">
                  <div
                    v-for="product in getSampleProducts(4)"
                    :key="product.id"
                    class="bg-white rounded-lg overflow-hidden border border-neutral-200"
                  >
                    <div class="relative aspect-square bg-neutral-200">
                      <div class="absolute inset-0 flex items-center justify-center">
                        <svg class="w-12 h-12 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                      </div>
                    </div>
                    <div class="p-3">
                      <p class="text-sm text-neutral-900 font-medium mb-2 truncate">{{ product.name }}</p>
                      <div class="border-t border-neutral-100 pt-2">
                        <div class="flex items-center gap-2">
                          <span class="px-1.5 py-0.5 bg-primary-600 text-white text-xs font-bold rounded">{{ product.discountRate }}%</span>
                          <span class="text-sm font-bold text-neutral-900">{{ product.price }}</span>
                          <span class="text-xs text-neutral-400 line-through">{{ product.originalPrice }}</span>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </template>

            <!-- 다른 섹션들의 기본 와이어프레임 -->
            <template v-else>
              <div class="bg-neutral-50 rounded-lg p-4 md:p-6 border border-neutral-200">
                <!-- 섹션 헤더 -->
                <div class="flex flex-col md:flex-row md:items-end justify-between gap-3 md:gap-4 mb-6">
                  <div class="flex flex-col md:flex-row gap-3 md:gap-4">
                    <div>
                      <label class="block text-xs text-neutral-500 mb-1">타이틀</label>
                      <input
                        v-model="section.title"
                        type="text"
                        class="w-full md:w-48 px-3 py-2 text-lg font-bold text-neutral-900 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                        placeholder="타이틀 입력"
                      >
                    </div>
                    <div>
                      <label class="block text-xs text-neutral-500 mb-1">설명</label>
                      <input
                        v-model="section.description"
                        type="text"
                        class="w-full md:w-56 px-3 py-2 text-sm text-neutral-600 bg-white border border-neutral-300 rounded-lg focus:border-primary-500 focus:ring-2 focus:ring-primary-100 focus:outline-none"
                        placeholder="설명 입력"
                      >
                    </div>
                  </div>
                  <!-- 네비게이션 버튼 (데스크탑에서만 표시) -->
                  <div class="hidden md:flex items-center gap-2">
                    <button type="button" class="w-10 h-10 flex items-center justify-center rounded-full border border-neutral-300 text-neutral-400 hover:border-neutral-400 hover:text-neutral-500">
                      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
                      </svg>
                    </button>
                    <button type="button" class="w-10 h-10 flex items-center justify-center rounded-full border border-neutral-300 text-neutral-400 hover:border-neutral-400 hover:text-neutral-500">
                      <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                      </svg>
                    </button>
                  </div>
                </div>

                <!-- 모바일 슬라이드 -->
                <div class="md:hidden overflow-x-auto scrollbar-hide -mx-4 px-4">
                  <div class="flex gap-3 snap-x snap-mandatory" style="width: max-content;">
                    <div
                      v-for="i in section.displayCount"
                      :key="i"
                      class="w-40 flex-shrink-0 snap-start bg-white rounded-lg overflow-hidden border border-neutral-200"
                    >
                      <div class="aspect-square bg-neutral-200 flex items-center justify-center">
                        <svg class="w-10 h-10 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                        </svg>
                      </div>
                      <div class="p-2.5">
                        <p class="text-xs text-neutral-900 font-medium mb-1.5">Sample {{ i }}</p>
                        <div class="border-t border-neutral-100 pt-1.5">
                          <span class="text-xs font-bold text-neutral-900">가격</span>
                        </div>
                      </div>
                    </div>
                  </div>
                  <!-- 슬라이드 힌트 -->
                  <p class="text-xs text-neutral-400 text-center mt-3">← 터치하여 슬라이드 →</p>
                </div>

                <!-- 데스크탑 그리드 -->
                <div class="hidden md:grid grid-cols-4 gap-4">
                  <div
                    v-for="i in section.displayCount"
                    :key="i"
                    class="bg-white rounded-lg overflow-hidden border border-neutral-200"
                  >
                    <div class="aspect-square bg-neutral-200 flex items-center justify-center">
                      <svg class="w-12 h-12 text-neutral-300" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
                      </svg>
                    </div>
                    <div class="p-3">
                      <p class="text-sm text-neutral-900 font-medium mb-2">Sample {{ i }}</p>
                      <div class="border-t border-neutral-100 pt-2">
                        <span class="text-sm font-bold text-neutral-900">가격</span>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </template>
            </div>
          </div>
        </div>
      </div>
    </div>
  </LayoutFormPage>
</template>
