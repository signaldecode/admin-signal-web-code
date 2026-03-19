<script setup>
/**
 * 관리자 계정 관리 페이지
 * - PATCH /admin/admins/{id}/status - 관리자 상태 변경
 */

import { useUiStore } from '~/stores/ui'
import { useAuthStore } from '~/stores/auth'

const { $api } = useNuxtApp()
const uiStore = useUiStore()
const authStore = useAuthStore()

// 관리자 상태 옵션
const statusOptions = [
  { value: 'ACTIVE', label: '활성', variant: 'success', description: '정상적으로 로그인 및 관리 기능을 사용할 수 있습니다.' },
  { value: 'INACTIVE', label: '비활성', variant: 'warning', description: '로그인이 제한되며 관리 기능을 사용할 수 없습니다.' },
  { value: 'SUSPENDED', label: '정지', variant: 'error', description: '계정이 정지되어 모든 접근이 차단됩니다.' },
]

// 폼 데이터
const adminId = ref('')
const selectedStatus = ref('')
const isChanging = ref(false)

// 상태 변경 실행
const handleStatusChange = async () => {
  if (!adminId.value || !selectedStatus.value) {
    uiStore.showToast({ type: 'error', message: '관리자 ID와 변경할 상태를 선택해주세요.' })
    return
  }

  const id = Number(adminId.value)
  if (isNaN(id) || id <= 0) {
    uiStore.showToast({ type: 'error', message: '올바른 관리자 ID를 입력해주세요.' })
    return
  }

  // 자기 자신 상태 변경 방지
  if (authStore.user?.id === id) {
    uiStore.showToast({ type: 'error', message: '본인 계정의 상태는 변경할 수 없습니다.' })
    return
  }

  const statusLabel = statusOptions.find((s) => s.value === selectedStatus.value)?.label
  if (!confirm(`관리자 ID ${id}의 상태를 "${statusLabel}"(으)로 변경하시겠습니까?`)) return

  isChanging.value = true

  try {
    await $api.patch(`/admin/admins/${id}/status`, {
      status: selectedStatus.value,
    })

    uiStore.showToast({
      type: 'success',
      message: `관리자 ID ${id}의 상태가 "${statusLabel}"(으)로 변경되었습니다.`,
    })

    // 폼 초기화
    adminId.value = ''
    selectedStatus.value = ''
  } catch (error) {
    const message = error.data?.error?.message || error.data?.message || error.message || '상태 변경에 실패했습니다.'
    uiStore.showToast({ type: 'error', message })
  } finally {
    isChanging.value = false
  }
}
</script>

<template>
  <LayoutFormPage
    title="관리자 계정 관리"
    description="관리자(ADMIN, STAFF) 계정의 상태를 변경합니다."
    save-text="상태 변경"
    :is-saving="isChanging"
    :show-cancel="false"
    @save="handleStatusChange"
  >
    <!-- 안내 -->
    <div class="p-4 bg-primary-50 border border-primary-200 rounded-lg">
      <div class="flex gap-3">
        <svg class="w-5 h-5 text-primary-500 flex-shrink-0 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
        </svg>
        <div class="text-sm text-primary-800">
          <p class="font-medium mb-1">관리자 상태 변경 안내</p>
          <ul class="text-primary-700 space-y-0.5">
            <li>- 상태를 변경하면 해당 관리자의 로그인 및 접근 권한이 즉시 변경됩니다.</li>
            <li>- 본인 계정의 상태는 변경할 수 없습니다.</li>
            <li>- ADMIN 역할만 관리자 상태를 변경할 수 있습니다.</li>
          </ul>
        </div>
      </div>
    </div>

    <!-- 관리자 ID 입력 -->
    <UiCard title="관리자 상태 변경">
      <div class="space-y-6">
        <div>
          <label class="block text-sm font-medium text-neutral-700 mb-2">관리자 ID</label>
          <input
            v-model="adminId"
            type="number"
            min="1"
            class="w-full max-w-xs px-3 py-2 border border-neutral-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
            placeholder="관리자 ID를 입력하세요"
          >
        </div>

        <!-- 상태 선택 -->
        <div>
          <label class="block text-sm font-medium text-neutral-700 mb-2">변경할 상태</label>
          <div class="space-y-2">
            <label
              v-for="option in statusOptions"
              :key="option.value"
              :class="[
                'flex items-center gap-4 p-4 border rounded-lg cursor-pointer transition-colors',
                selectedStatus === option.value
                  ? 'border-primary-500 bg-primary-50'
                  : 'border-neutral-200 hover:border-neutral-300',
              ]"
            >
              <input
                v-model="selectedStatus"
                type="radio"
                :value="option.value"
                class="w-4 h-4 text-primary-600 focus:ring-primary-500"
              >
              <div class="flex-1">
                <div class="flex items-center gap-2">
                  <span class="text-sm font-medium text-neutral-900">{{ option.label }}</span>
                  <UiBadge :variant="option.variant" size="sm">
                    {{ option.value }}
                  </UiBadge>
                </div>
                <p class="text-xs text-neutral-500 mt-0.5">{{ option.description }}</p>
              </div>
            </label>
          </div>
        </div>
      </div>
    </UiCard>
  </LayoutFormPage>
</template>
