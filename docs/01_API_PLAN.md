# MuinGuard API 계획서

## 1. 공통 규칙

- Base URL: `/api/v1`
- 인증: `Authorization: Bearer {token}`
- 성공: `{ "success": true, "data": {}, "error": null }`
- 실패: `{ "success": false, "data": null, "error": { "code": "...", "message": "..." } }`
- 날짜: ISO 8601 UTC 저장, 화면은 한국 시간 표시
- 아래에 명시한 중복 영향이 큰 생성·상태변경 요청은 `Idempotency-Key: {UUID}` 헤더를 사용한다.

### 멱등성(Idempotency) 정책

네트워크 재시도나 사용자의 중복 클릭으로 동일한 요청이 여러 번 전송되어도 결과가 한 번만 반영되게 한다.

| 적용 API | 멱등성 키 범위 | 중복 요청 처리 |
| --- | --- | --- |
| `POST /places/{placeId}/reservation-requests` | 회원+장소+키 | 기존 예약 문의 결과 반환 |
| `POST /places/{placeId}/incidents` | 회원+장소+키 | 기존 시설 제보 결과 반환 |
| `POST /subscriptions/checkout` | 운영자+요금제+키 | 기존 결제 세션 결과 반환 |
| `POST /webhooks/payments` | PG event ID/payment ID | 이미 처리된 이벤트는 무시하고 200 반환 |
| `PATCH /owner/incidents/{incidentId}` | 제보+상태변경 키 | 같은 상태 전환을 한 번만 기록 |
| `POST /places` | 운영자+키 | 기존 장소 등록 요청 결과 반환 |
| `POST /organizations` | 운영자+키 | 기존 조직 생성 결과 반환 |
| `POST /organizations/{organizationId}/members` | 조직+초대 대상+키 | 기존 초대 결과 반환 |

- 서버는 키, 요청 본문 해시, 처리 결과, 생성 시각을 저장한다.
- 동일 키에 다른 요청 본문이 오면 `409 Conflict`로 거절한다.
- 처리 중인 동일 키는 기존 처리 상태를 반환하거나 짧게 대기한다.
- 키 보관 기간은 API 특성에 따라 정하고, 결제·웹훅 이벤트는 감사 목적에 맞게 별도 보관한다.

## 2. 인증·회원 API

| Method | Endpoint | 설명 | 권한 |
| --- | --- | --- | --- |
| POST | `/auth/signup` | 회원가입 | 공개 |
| POST | `/auth/login` | 로그인 | 공개 |
| POST | `/auth/logout` | 로그아웃 | 회원 |
| GET | `/users/me` | 내 정보 조회 | 회원 |
| PATCH | `/users/me` | 내 정보 수정 | 회원 |

## 3. 장소·지도 API

| Method | Endpoint | 설명 | 권한 |
| --- | --- | --- | --- |
| GET | `/places` | 지도·목록 검색 | 공개 |
| GET | `/places/{placeId}` | 장소 상세 | 공개 |
| POST | `/places` | 사장 장소 등록 요청 및 자동 승인 규칙 평가 | 운영자 |
| PATCH | `/places/{placeId}` | 장소 수정 | 소유 운영자 |
| POST | `/places/{placeId}/images` | 장소 사진 등록 | 소유 운영자 |
| POST | `/owner/places/{placeId}/instagram-posts` | 공개 Instagram 게시물/Reel URL 등록 (P1) | 소유 운영자 |
| GET | `/places/{placeId}/social-posts` | 공식 임베드용 최근 소식 조회 (P1) | 공개 |

`GET /places` Query: `latitude`, `longitude`, `radius`, `category`, `open24h`, `parking`, `reservable`, `sort`.

응답 핵심 필드: `id`, `name`, `category`, `latitude`, `longitude`, `distanceM`, `rating`, `businessHours`, `amenities`, `thumbnailUrl`, `instagramProfileUrl`, `instagramVerificationStatus`.

장소 상세에는 `dataSource`, `verifiedAt`, `verificationStatus`를 제공해 정보의 출처와 최근 확인 여부를 표시한다.

### Instagram 연동 정책

| 단계 | API/데이터 | 목적 |
| --- | --- | --- |
| P0 | 장소 등록·수정의 `instagramHandle`, `instagramProfileUrl`, `instagramVerificationStatus` | 인증된 운영자 계정으로 외부 이동 |
| P1 | `POST /owner/places/{placeId}/instagram-posts` | 운영자가 선택한 공개 게시물/Reel URL 등록 |
| P1 | `GET /places/{placeId}/social-posts` | 공식 oEmbed를 통한 표시용 데이터 조회 |

- P0은 단순 외부 링크이며, Instagram 로그인·API 토큰이 필요 없다.
- P1은 공개 게시물/Reel만 공식 임베드로 표시하고, 비공개 계정·스토리·무단 크롤링은 지원하지 않는다.
- 서비스는 게시물 원문·이미지·영상 콘텐츠를 자체 저장하거나 분석 데이터로 재사용하지 않는다.

## 4. 리뷰·즐겨찾기·예약 문의 API

| Method | Endpoint | 설명 | 권한 |
| --- | --- | --- | --- |
| GET | `/places/{placeId}/reviews` | 리뷰 목록 | 공개 |
| POST | `/places/{placeId}/reviews` | 리뷰 작성 | 회원 |
| POST/DELETE | `/places/{placeId}/favorites` | 즐겨찾기 추가/해제 | 회원 |
| POST | `/places/{placeId}/reservation-requests` | 예약 문의 (P1) | 회원 |
| GET | `/reservation-requests` | 예약 문의 목록 (P1) | 회원/운영자 |
| POST | `/places/{placeId}/incidents` | 시설·운영 문제 제보 | 회원 |
| GET | `/places/{placeId}/incidents` | 장소별 제보 목록 | 소유 운영자/관리자 |

예약 문의는 실제 확정 예약이 아님을 UI와 API 응답에 명시한다.

## 5. 운영자·관리자·구독 API

| Method | Endpoint | 설명 | 권한 |
| --- | --- | --- | --- |
| GET | `/owner/places` | 내 장소 목록 | 운영자 |
| GET | `/owner/dashboard` | P0: 시설 제보·지점 현황 / P1: 유입 통계 | 운영자 |
| GET | `/owner/incidents` | 내 매장 시설 장애 제보 통합 목록 | 운영자 |
| PATCH | `/owner/incidents/{incidentId}` | 담당·상태·처리 메모 변경 | 운영자 |
| GET | `/organizations/{organizationId}/dashboard` | 다점포 통합 현황 | 조직 멤버 |
| POST | `/organizations` | 사업자 조직 생성 | 운영자 |
| POST | `/organizations/{organizationId}/members` | 직원/지점 관리자 초대 | 조직 관리자 |
| GET | `/plans` | Basic/Pro/Business 요금제 (P1) | 공개 |
| GET | `/subscriptions/me` | 현재 구독 상태 (P1) | 운영자 |
| POST | `/subscriptions/checkout` | 테스트 결제 세션 생성 (P1) | 운영자 |
| POST | `/subscriptions/cancel` | 해지 예약 (P1) | 운영자 |
| POST | `/webhooks/payments` | PG 결제 상태 수신 (P1) | PG 전용 |
| GET | `/admin/places/pending` | 승인 대기 장소 | 관리자 |
| PATCH | `/admin/places/{placeId}/approval` | 승인/반려 | 관리자 |
| GET | `/admin/approval-policies` | 자동 승인 규칙 조회 | 관리자 |
| PATCH | `/admin/approval-policies` | 자동 승인 규칙 변경 | 관리자 |
| GET | `/admin/recovery-logs` | 장애·복구 이력 | 관리자 |

## 6. 운영 API와 보안

- `GET /health`: 서비스 생존 확인
- `GET /ready`: DB·Redis·필수 의존성 준비 상태
- `GET /operations/tasks/{taskId}`: Handoff·시도 횟수·결과 확인
- 시설 제보에는 `category`, `severity`, `description`, `photoUrl`, `reportedAt`을 기록한다.
- 제보 상태는 `RECEIVED → ASSIGNED → IN_PROGRESS → RESOLVED`로 관리한다.
- P1 결제 웹훅은 PG 서명과 provider event ID/payment ID로 중복 처리하지 않는다. 외부 PG에 `Idempotency-Key` 헤더를 요구하지 않는다.
- 비밀번호는 단방향 해시로 저장하고, API 키·결제정보는 DB에 저장하지 않는다.
- 입력 검증, 권한 검사, Rate Limit, 감사 로그를 적용한다.
- 조직 단위 API는 URL의 조직 ID만 믿지 않고, 토큰의 조직 멤버십과 역할을 서버에서 대조한다.
- 장소 등록·수정·제보 상태 변경은 작성자·수정 시각을 감사 로그에 남긴다.

### 사장 등록·승인 상태

| 상태 | 의미 | 다음 처리 주체 |
| --- | --- | --- |
| DRAFT | 사장이 작성 중 | 사장 |
| PENDING_REVIEW | 규칙 미충족 또는 위험 신호 | 관리자 |
| AUTO_APPROVED | 자동 승인되어 공개 | 시스템·사장 |
| APPROVED | 관리자가 승인하여 공개 | 관리자 |
| REJECTED | 관리자 반려 | 사장 수정 후 재제출 |
| SUSPENDED | 신고·정책 위반으로 비공개 | 관리자 |

### P0 자동 승인 규칙

아래 조건을 모두 만족하면 `AUTO_APPROVED`로 공개한다. 규칙은 관리자만 변경한다.

1. 사업자 인증이 완료되고 사업자 정보의 유효성을 확인할 수 있다.
2. 장소명, 카테고리, 주소, 좌표, 전화번호, 운영시간, 대표 사진이 모두 있다.
3. 주소·전화번호·좌표 기준으로 기존 공개 장소와 중복되지 않는다.
4. 업종별 필수 고지(예: 숙소 체크인 정보)가 입력되어 있다.
5. 금지어·허위 연락처·비정상 이미지 등 기본 정책 위반 신호가 없다.

조건을 하나라도 충족하지 못하면 자동 반려하지 않고 `PENDING_REVIEW`로 보내며, 사장에게 누락 사유를 보여준다.
