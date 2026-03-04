<script setup>
/**
 * 제작문의 상세/답변 페이지
 *
 * API:
 * - GET /admin/inquiries/{id} (상세)
 * - PATCH /admin/inquiries/{id}/answer (답변 등록)
 * - DELETE /admin/inquiries/{id} (삭제)
 *
 * 응답 데이터:
 * {
 *   id, userName, title, serviceType, fieldType,
 *   phone, email, referenceUrl, content, attachmentUrls,
 *   status, isAnswered, answer, answeredBy, answeredAt, createdAt
 * }
 */

import { useUiStore } from '~/stores/ui'
import { useApi } from '~/composables/useApi'
import { formatDate } from '~/utils/formatters'

const route = useRoute()
const router = useRouter()
const uiStore = useUiStore()
const { get, patch, del } = useApi()

const inquiryId = computed(() => route.params.id)

// 상태 옵션
const statusOptions = [
  { value: 'PENDING', label: '답변대기', color: 'warning' },
  { value: 'ANSWERED', label: '답변완료', color: 'success' },
  { value: 'CLOSED', label: '종료', color: 'neutral' },
]

// 서비스 타입 라벨
const serviceTypeLabels = {
  design: '디자인',
  development: '개발',
  marketing: '마케팅',
  consulting: '컨설팅',
}

// 문의 데이터
const inquiry = ref({
  id: null,
  userName: '',
  title: '',
  serviceType: '',
  fieldType: '',
  phone: '',
  email: '',
  referenceUrl: '',
  content: '',
  attachmentUrls: null,
  status: 'PENDING',
  isAnswered: false,
  answer: '',
  answeredBy: null,
  answeredAt: null,
  createdAt: '',
})

const isLoading = ref(false)
const isSaving = ref(false)

// 답변 폼
const answerForm = ref('')

// 문의 상세 조회
const fetchInquiry = async () => {
  isLoading.value = true

  try {
    const response = await get(`/admin/inquiries/${inquiryId.value}`)
    const data = response.data

    inquiry.value = {
      id: data.id,
      userName: data.userName || '',
      title: data.title || '',
      serviceType: data.serviceType || '',
      fieldType: data.fieldType || '',
      phone: data.phone || '',
      email: data.email || '',
      referenceUrl: data.referenceUrl || '',
      content: data.content || '',
      attachmentUrls: data.attachmentUrls || null,
      status: data.status || 'PENDING',
      isAnswered: data.isAnswered ?? false,
      answer: data.answer || '',
      answeredBy: data.answeredBy,
      answeredAt: data.answeredAt,
      createdAt: data.createdAt,
    }

    // 기존 답변이 있으면 폼에 세팅
    answerForm.value = data.answer || ''
  } catch (error) {
    uiStore.showToast({
      type: 'error',
      message: error.message || '문의를 불러오는데 실패했습니다.',
    })
    router.push('/admin/inquiries')
  } finally {
    isLoading.value = false
  }
}

// 헬퍼
const getStatusBadge = (status) => {
  return statusOptions.find((s) => s.value === status) || { label: status || '-', color: 'neutral' }
}

const getServiceTypeLabel = (type) => {
  return serviceTypeLabels[type] || type || '-'
}

// 전화번호 포맷팅
const formatPhone = (phone) => {
  if (!phone) return '-'
  const cleaned = phone.replace(/\D/g, '')
  if (cleaned.length === 11) {
    return `${cleaned.slice(0, 3)}-${cleaned.slice(3, 7)}-${cleaned.slice(7)}`
  }
  if (cleaned.length === 10) {
    return `${cleaned.slice(0, 3)}-${cleaned.slice(3, 6)}-${cleaned.slice(6)}`
  }
  return phone
}

// 답변 저장
const handleSave = async () => {
  if (!answerForm.value.trim()) {
    uiStore.showToast({ type: 'error', message: '메모를 입력해주세요.' })
    return
  }

  isSaving.value = true

  try {
    await patch(`/admin/inquiries/${inquiryId.value}/answer`, {
      answer: answerForm.value,
    })

    uiStore.showToast({ type: 'success', message: '메모가 등록되었습니다.' })
    router.push('/admin/inquiries')
  } catch (error) {
    uiStore.showToast({
      type: 'error',
      message: error.message || '메모 저장에 실패했습니다.',
    })
  } finally {
    isSaving.value = false
  }
}

// 삭제
const handleDelete = async () => {
  if (!confirm('이 문의를 삭제하시겠습니까?')) return

  try {
    await del(`/admin/inquiries/${inquiryId.value}`)
    uiStore.showToast({ type: 'success', message: '삭제되었습니다.' })
    router.push('/admin/inquiries')
  } catch (error) {
    uiStore.showToast({
      type: 'error',
      message: error.message || '삭제에 실패했습니다.',
    })
  }
}

const handleCancel = () => router.back()

onMounted(() => {
  fetchInquiry()
})
</script>

<template>
  <LayoutFormPage
    title="제작문의 상세"
    description="제작문의 내용을 확인하고 메모합니다."
    save-text="메모 저장"
    :is-saving="isSaving"
    show-cancel
    show-delete
    @save="handleSave"
    @cancel="handleCancel"
    @delete="handleDelete"
  >
    <div v-if="isLoading" class="flex items-center justify-center py-20">
      <UiSpinner size="lg" />
    </div>

    <template v-else>
      <!-- 문의 정보 -->
      <UiCard>
        <template #header>
          <div class="flex items-center justify-between">
            <h3 class="font-semibold text-neutral-900">문의 정보</h3>
            <UiBadge :variant="getStatusBadge(inquiry.status).color" size="sm">
              {{ getStatusBadge(inquiry.status).label }}
            </UiBadge>
          </div>
        </template>

        <div class="space-y-6">
          <!-- 제목 -->
          <div>
            <h3 class="text-lg font-medium text-neutral-900">{{ inquiry.title || '-' }}</h3>
            <p class="text-sm text-neutral-500 mt-1">
              {{ inquiry.createdAt ? formatDate(inquiry.createdAt) : '-' }}
            </p>
          </div>

          <!-- 문의 유형 -->
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4 p-4 bg-neutral-50 rounded-lg">
            <div>
              <p class="text-xs text-neutral-500 mb-1">서비스 유형</p>
              <p class="text-sm font-medium text-neutral-900">{{ getServiceTypeLabel(inquiry.serviceType) }}</p>
            </div>
            <div>
              <p class="text-xs text-neutral-500 mb-1">분야</p>
              <p class="text-sm font-medium text-neutral-900">{{ inquiry.fieldType || '-' }}</p>
            </div>
          </div>

          <!-- 고객 정보 -->
          <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
            <div>
              <p class="text-xs text-neutral-500 mb-1">작성자</p>
              <p class="text-sm font-medium text-neutral-900">{{ inquiry.userName || '-' }}</p>
            </div>
            <div>
              <p class="text-xs text-neutral-500 mb-1">연락처</p>
              <p class="text-sm font-medium text-neutral-900">{{ formatPhone(inquiry.phone) }}</p>
            </div>
            <div>
              <p class="text-xs text-neutral-500 mb-1">이메일</p>
              <p class="text-sm font-medium text-neutral-900">
                <a
                  v-if="inquiry.email"
                  :href="`mailto:${inquiry.email}`"
                  class="text-primary-600 hover:underline"
                >
                  {{ inquiry.email }}
                </a>
                <span v-else>-</span>
              </p>
            </div>
          </div>

          <!-- 참고 URL -->
          <div v-if="inquiry.referenceUrl">
            <p class="text-xs text-neutral-500 mb-1">참고 URL</p>
            <a
              :href="inquiry.referenceUrl"
              target="_blank"
              rel="noopener noreferrer"
              class="text-sm text-primary-600 hover:underline break-all"
            >
              {{ inquiry.referenceUrl }}
            </a>
          </div>

          <!-- 문의 내용 -->
          <div>
            <p class="text-xs text-neutral-500 mb-2">문의 내용</p>
            <div class="p-4 bg-neutral-50 rounded-lg">
              <p class="text-sm text-neutral-700 whitespace-pre-wrap">{{ inquiry.content || '-' }}</p>
            </div>
          </div>

          <!-- 첨부파일 -->
          <div v-if="inquiry.attachmentUrls && inquiry.attachmentUrls.length > 0">
            <p class="text-xs text-neutral-500 mb-2">첨부파일</p>
            <div class="flex flex-wrap gap-2">
              <a
                v-for="(url, index) in inquiry.attachmentUrls"
                :key="index"
                :href="url"
                target="_blank"
                rel="noopener noreferrer"
                class="inline-flex items-center gap-1.5 px-3 py-1.5 bg-neutral-100 hover:bg-neutral-200 rounded-lg text-sm text-neutral-700 transition-colors"
              >
                <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.172 7l-6.586 6.586a2 2 0 102.828 2.828l6.414-6.586a4 4 0 00-5.656-5.656l-6.415 6.585a6 6 0 108.486 8.486L20.5 13" />
                </svg>
                첨부파일 {{ index + 1 }}
              </a>
            </div>
          </div>
        </div>
      </UiCard>

      <!-- 답변 -->
      <UiCard>
        <template #header>
          <h3 class="font-semibold text-neutral-900">메모</h3>
        </template>

        <div class="space-y-3">
          <div v-if="inquiry.answeredAt" class="flex items-center gap-4 text-sm text-neutral-500">
            <span>메모일: {{ formatDate(inquiry.answeredAt) }}</span>
            <span v-if="inquiry.answeredBy">메모자: {{ inquiry.answeredBy }}</span>
          </div>
          <textarea
            v-model="answerForm"
            rows="8"
            class="w-full px-3 py-2 border border-neutral-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-primary-500"
            placeholder="메모를 입력하세요"
          />
        </div>
      </UiCard>
    </template>
  </LayoutFormPage>
</template>
