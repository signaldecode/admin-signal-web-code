# Admin 페이지 리팩터링 계획

## 현재 상태

- Nuxt 4 + Tailwind CSS 기반 관리자 대시보드
- 메인 디스플레이 관리: `/admin/display` (섹션 순서/활성화/타이틀 편집)
- 배너 관리: `/admin/design/banners` (HERO/SLIDE/FULL/HALF)
- 팝업 관리: `/admin/design/popups` (CENTER/FLOATING)
- 헤더 메뉴 관리: `/admin/design/header`

## Front 메인페이지 리팩터링에 따른 Admin 영향

> 백엔드 API 수정 없음 — Admin 코드 변경 불필요

Front에서 섹션 레지스트리 패턴을 도입하더라도, Admin이 관리하는 데이터 구조는 동일:

| Admin 관리 항목 | API | Front 소비 방식 |
|----------------|-----|----------------|
| 섹션 순서/활성화 | `PUT /admin/displays` | `GET /main/shop-info` → sections 배열 |
| 배너 CRUD | `POST/PATCH /admin/banners` | `GET /main/banners` |
| 팝업 CRUD | `POST/PATCH /admin/popups` | `GET /popups` |
| 헤더 메뉴 | `PUT /admin/tenant/header-menu` | `GET /main/shop-info` |

## 향후 Admin 개선 고려사항

### 1. 디스플레이 관리 UX 개선
- 현재: 섹션 리스트 + 드래그 정렬
- 개선 가능: 실시간 프리뷰, 섹션별 상세 설정 확장

### 2. 새 섹션 타입 추가 시
- Admin `display/index.vue`의 `sectionConfig`에 새 keyword 추가
- Front의 섹션 레지스트리에 대응 컴포넌트 추가
- 백엔드 keyword 추가 필요 (API 변경 발생 시에만)

### 3. 배너 관리 개선
- 현재: position별 필터링 + CRUD
- 개선 가능: 배너 스케줄링 캘린더 뷰, A/B 테스트 지원
