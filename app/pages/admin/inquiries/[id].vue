<script setup>
/**
 * 1:1 문의 상세/답변 페이지
 *
 * API:
 * - GET /admin/inquiries/{id} (상세)
 * - PATCH /admin/inquiries/{id}/answer (답변 등록)
 * - DELETE /admin/inquiries/{id} (삭제)
 *
 * 응답 데이터:
 * {
 *   id, userId, userName, title, content,
 *   isAnswered, answer, answeredBy, answeredAt, status,
 *   createdAt, updatedAt
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

// 문의 데이터
const inquiry = ref({
  id: null,
  userId: null,
  userName: '',
  title: '',
  content: '',
  isAnswered: false,
  answer: '',
  answeredBy: null,
  answeredAt: null,
  status: 'PENDING',
  createdAt: '',
  updatedAt: '',
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
      userId: data.userId,
      userName: data.userName || '',
      title: data.title || '',
      content: data.content || '',
      isAnswered: data.isAnswered ?? false,
      answer: data.answer || '',
      answeredBy: data.answeredBy,
      answeredAt: data.answeredAt,
      status: data.status || 'PENDING',
      createdAt: data.createdAt,
      updatedAt: data.updatedAt,
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

// 답변 저장
const handleSave = async () => {
  if (!answerForm.value.trim()) {
    uiStore.showToast({ type: 'error', message: '답변을 입력해주세요.' })
    return
  }

  isSaving.value = true

  try {
    await patch(`/admin/inquiries/${inquiryId.value}/answer`, {
      answer: answerForm.value,
    })

    uiStore.showToast({ type: 'success', message: '답변이 등록되었습니다.' })
    router.push('/admin/inquiries')
  } catch (error) {
    uiStore.showToast({
      type: 'error',
      message: error.message || '답변 저장에 실패했습니다.',
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
    title="1:1 문의 상세"
    description="고객 1:1 문의에 답변합니다."
    save-text="답변 저장"
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
      <UiCard title="문의 정보">
        <div class="space-y-4">
          <div class="flex items-center gap-3 flex-wrap">
            <UiBadge :variant="getStatusBadge(inquiry.status).color" size="sm">
              {{ getStatusBadge(inquiry.status).label }}
            </UiBadge>
          </div>

          <div>
            <h3 class="text-lg font-medium text-neutral-900">{{ inquiry.title || '-' }}</h3>
            <div class="flex items-center gap-3 mt-1 text-sm text-neutral-500">
              <span>작성자: {{ inquiry.userName || '-' }}</span>
              <span>{{ inquiry.createdAt ? formatDate(inquiry.createdAt) : '-' }}</span>
            </div>
          </div>

          <div class="p-4 bg-neutral-50 rounded-lg">
            <p class="text-sm text-neutral-700 whitespace-pre-wrap">{{ inquiry.content || '-' }}</p>
          </div>
        </div>
      </UiCard>

      <!-- 답변 -->
      <UiCard title="답변">
        <div class="space-y-3">
          <div v-if="inquiry.answeredAt" class="flex items-center gap-4 text-sm text-neutral-500">
            <span>답변일: {{ formatDate(inquiry.answeredAt) }}</span>
            <span v-if="inquiry.answeredBy">답변자 ID: {{ inquiry.answeredBy }}</span>
          </div>
          <textarea
            v-model="answerForm"
            rows="8"
            class="w-full px-3 py-2 border border-neutral-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-primary-500"
            placeholder="답변을 입력하세요"
          />
        </div>
      </UiCard>
    </template>
  </LayoutFormPage>
</template>
