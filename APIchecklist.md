# API 연동 체크리스트

## Auth

- [x] `POST /api/v1/auth/login` - 로그인
- [x] `POST /api/v1/auth/logout` - 로그아웃
- [x] `GET /api/v1/auth/admin/me` - 내 정보 조회
- [ ] `POST /api/v1/auth/signup` - 회원가입
- [ ] `POST /api/v1/auth/refresh` - 토큰 재발급
- [ ] `POST /api/v1/auth/email-verifications` - 인증 코드 발송
- [ ] `PATCH /api/v1/auth/email-verifications` - 인증 코드 확인
- [ ] `POST /api/v1/auth/password-reset/send` - 비밀번호 재설정 코드 발송
- [ ] `PATCH /api/v1/auth/password-reset` - 비밀번호 재설정
- [ ] `POST /api/v1/auth/find-email/send` - 아이디 찾기 SMS 발송
- [ ] `POST /api/v1/auth/find-email/verify` - 아이디 찾기 인증번호 확인
- [ ] `POST /api/v1/auth/oauth2/naver` - 네이버 로그인
- [ ] `POST /api/v1/auth/oauth2/naver/link` - 네이버 계정 연동
- [ ] `POST /api/v1/auth/oauth2/google` - 구글 로그인
- [ ] `POST /api/v1/auth/oauth2/google/link` - 구글 계정 연동
- [ ] `POST /api/v1/auth/oauth2/signup` - 소셜 회원가입 완료

## Admin Tenant (설정)

- [x] `GET /api/v1/admin/tenant` - 전체 설정 조회
- [x] `PUT /api/v1/admin/tenant` - 전체 설정 수정
- [x] `GET /api/v1/admin/tenant/header-menu` - 헤더 메뉴 조회
- [x] `PUT /api/v1/admin/tenant/header-menu` - 헤더 메뉴 수정

## Admin Policy (정책)

- [x] `GET /api/v1/admin/policy` - 전체 정책 조회
- [x] `PUT /api/v1/admin/policy` - 전체 정책 수정

## Admin Product (상품)

- [x] `GET /api/v1/admin/products` - 상품 목록 조회
- [x] `POST /api/v1/admin/products` - 상품 등록
- [x] `GET /api/v1/admin/products/{productId}` - 상품 상세 조회
- [x] `PATCH /api/v1/admin/products/{productId}` - 상품 수정
- [x] `DELETE /api/v1/admin/products/{productId}` - 상품 삭제

## Admin Category (카테고리)

- [x] `PUT /api/v1/admin/categories/sync` - 카테고리 동기화

## Admin Tag (태그)

- [x] `GET /api/v1/admin/tags` - 태그 목록 조회
- [x] `POST /api/v1/admin/tags` - 태그 생성

## Admin Display (전시)

- [x] `GET /api/v1/admin/displays` - 전시 섹션 전체 조회
- [x] `PUT /api/v1/admin/displays` - 전시 섹션 일괄 수정

## Admin Promotion (프로모션)

- [x] `GET /api/v1/admin/promotions` - 프로모션 목록 조회
- [x] `POST /api/v1/admin/promotions` - 프로모션 등록
- [x] `GET /api/v1/admin/promotions/{promotionId}` - 프로모션 상세 조회
- [x] `PUT /api/v1/admin/promotions/{promotionId}` - 프로모션 수정
- [x] `DELETE /api/v1/admin/promotions/{promotionId}` - 프로모션 삭제

## Admin Coupon (쿠폰)

- [x] `GET /api/v1/admin/coupons` - 쿠폰 목록 조회
- [x] `POST /api/v1/admin/coupons` - 쿠폰 등록
- [x] `GET /api/v1/admin/coupons/{couponId}` - 쿠폰 상세 조회
- [x] `PATCH /api/v1/admin/coupons/{couponId}` - 쿠폰 수정
- [x] `DELETE /api/v1/admin/coupons/{couponId}` - 쿠폰 삭제
- [x] `PATCH /api/v1/admin/coupons/{couponId}/visibility` - 쿠폰 노출/비노출 토글
- [x] `PATCH /api/v1/admin/coupons/{couponId}/status` - 쿠폰 발급상태 변경
- [x] `POST /api/v1/admin/coupons/{couponId}/recall` - 쿠폰 회수

## Admin Banner (배너)

- [x] `GET /api/v1/admin/banners` - 배너 목록 조회
- [x] `POST /api/v1/admin/banners` - 배너 등록
- [x] `GET /api/v1/admin/banners/{bannerId}` - 배너 상세 조회
- [x] `PATCH /api/v1/admin/banners/{bannerId}` - 배너 수정
- [x] `DELETE /api/v1/admin/banners/{bannerId}` - 배너 삭제

## Admin Notice (공지사항)

- [x] `GET /api/v1/admin/notices` - 공지사항 목록 조회
- [x] `POST /api/v1/admin/notices` - 공지사항 등록
- [x] `GET /api/v1/admin/notices/{noticeId}` - 공지사항 상세 조회
- [x] `PATCH /api/v1/admin/notices/{noticeId}` - 공지사항 수정
- [x] `DELETE /api/v1/admin/notices/{noticeId}` - 공지사항 삭제

## Admin Popup (팝업)

- [x] `GET /api/v1/admin/popups` - 팝업 목록 조회
- [x] `POST /api/v1/admin/popups` - 팝업 등록
- [x] `GET /api/v1/admin/popups/{popupId}` - 팝업 상세 조회
- [x] `PATCH /api/v1/admin/popups/{popupId}` - 팝업 수정
- [x] `DELETE /api/v1/admin/popups/{popupId}` - 팝업 삭제

## Admin FAQ

- [x] `GET /api/v1/admin/faqs` - FAQ 목록 조회
- [x] `POST /api/v1/admin/faqs` - FAQ 등록
- [x] `GET /api/v1/admin/faqs/{faqId}` - FAQ 상세 조회
- [x] `PATCH /api/v1/admin/faqs/{faqId}` - FAQ 수정
- [x] `DELETE /api/v1/admin/faqs/{faqId}` - FAQ 삭제
- [x] `GET /api/v1/admin/faqs/categories` - FAQ 카테고리 목록 조회
- [x] `POST /api/v1/admin/faqs/categories` - FAQ 카테고리 등록
- [x] `PATCH /api/v1/admin/faqs/categories/{categoryId}` - FAQ 카테고리 수정
- [x] `DELETE /api/v1/admin/faqs/categories/{categoryId}` - FAQ 카테고리 삭제

## Admin QNA (Q&A)

- [x] `GET /api/v1/admin/qnas` - Q&A 목록 조회
- [x] `PATCH /api/v1/admin/qnas/{qnaId}/answer` - Q&A 답변 등록

## Admin Inquiry (제작문의)

- [x] `GET /api/v1/admin/inquiries` - 제작문의 목록 조회
- [x] `PATCH /api/v1/admin/inquiries/{id}/answer` - 제작문의 답변 등록

## Admin Review (리뷰)

- [x] `GET /api/v1/admin/reviews` - 리뷰 목록 조회
- [x] `GET /api/v1/admin/reviews/{id}` - 리뷰 상세 조회
- [x] `DELETE /api/v1/admin/reviews` - 리뷰 삭제
- [x] `PATCH /api/v1/admin/reviews/visibility` - 리뷰 노출/비노출 토글

## Admin Order (주문)

- [x] `GET /api/v1/admin/orders` - 주문 목록 조회
- [x] `GET /api/v1/admin/orders/{id}` - 주문 상세 조회
- [x] `GET /api/v1/admin/orders/statuses` - 주문 상태 목록 조회
- [x] `PATCH /api/v1/admin/orders/statuses` - 주문 상태 변경

## Admin Claim (클레임)

- [x] `GET /api/v1/admin/claims` - 교환/반품/취소 목록 조회
- [x] `POST /api/v1/admin/claims` - 클레임 생성
- [x] `GET /api/v1/admin/claims/orders/{id}` - 주문별 클레임 조회
- [x] `POST /api/v1/admin/claims/{claimId}/approve` - 클레임 승인
- [x] `POST /api/v1/admin/claims/{claimId}/reject` - 클레임 거절
- [x] `POST /api/v1/admin/claims/{claimId}/return-shipping` - 반송장번호 등록
- [x] `POST /api/v1/admin/claims/{claimId}/reship` - 거절 후 재배송 송장 등록
- [x] `POST /api/v1/admin/claims/{claimId}/receive-inspect` - 입고 검수 완료
- [x] `POST /api/v1/admin/claims/{claimId}/exchange/complete` - 교환 완료

## Admin Refund (환불)

- [x] `POST /api/v1/admin/refunds` - 환불 처리

## Admin Delivery (배송)

- [x] `GET /api/v1/admin/delivery/carriers` - 택배사 목록 조회
- [x] `POST /api/v1/admin/delivery/orders/{orderId}/shipments` - 배송 생성 (분할배송)
- [x] `POST /api/v1/admin/delivery/track/shipment/{shipmentId}/refresh` - 배송 추적 정보 갱신
- [x] `POST /api/v1/admin/delivery/track/order/{orderId}/refresh` - 주문별 배송 추적 정보 갱신

## Admin User (회원)

- [x] `GET /api/v1/admin/users` - 회원 목록 조회
- [x] `GET /api/v1/admin/users/{id}` - 회원 상세 조회
- [x] `GET /api/v1/admin/users/{id}/points` - 회원 포인트 조회
- [x] `POST /api/v1/admin/users/{id}/points` - 회원 포인트 수동 조정
- [x] `POST /api/v1/admin/users/{id}/cs-memos` - 회원 CS 메모 작성
- [x] `PUT /api/v1/admin/users/{id}/cs-memos/{memoId}` - 회원 CS 메모 수정
- [x] `DELETE /api/v1/admin/users/{id}/cs-memos/{memoId}` - 회원 CS 메모 삭제
- [x] `GET /api/v1/admin/users/grades` - 등급 목록 조회
- [x] `PATCH /api/v1/admin/users/grades/{gradeId}` - 등급 정책 수정
- [x] `PATCH /api/v1/admin/users/grades` - 회원 등급 일괄 변경

## Admin Dashboard

- [x] `GET /api/v1/admin/dashboard` - 대시보드 조회

## Admin Image (이미지)

- [x] `POST /api/v1/admin/images` - 이미지 업로드

## Admin 관리자

- [x] `PATCH /api/v1/admin/admins/{id}/status` - 관리자 상태 변경

---

## 미연동 Admin API 요약

모든 Admin API 연동 완료!

## 미연동 사용자 프론트용 API (Admin에서 불필요)

| # | 엔드포인트 | 설명 |
|---|-----------|------|
| 1 | `POST /auth/signup` | 회원가입 |
| 2 | `POST /auth/refresh` | 토큰 재발급 |
| 3 | `POST /auth/email-verifications` | 인증 코드 발송 |
| 4 | `PATCH /auth/email-verifications` | 인증 코드 확인 |
| 5 | `POST /auth/password-reset/send` | 비밀번호 재설정 코드 발송 |
| 6 | `PATCH /auth/password-reset` | 비밀번호 재설정 |
| 7 | `POST /auth/find-email/send` | 아이디 찾기 SMS 발송 |
| 8 | `POST /auth/find-email/verify` | 아이디 찾기 인증 확인 |
| 9 | `POST /auth/oauth2/naver` | 네이버 로그인 |
| 10 | `POST /auth/oauth2/naver/link` | 네이버 계정 연동 |
| 11 | `POST /auth/oauth2/google` | 구글 로그인 |
| 12 | `POST /auth/oauth2/google/link` | 구글 계정 연동 |
| 13 | `POST /auth/oauth2/signup` | 소셜 회원가입 |
