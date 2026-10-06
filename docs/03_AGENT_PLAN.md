# MuinGuard 에이전트 계획서

## 1. 목적

멀티 에이전트는 장소 검색, 정보 정리, 예약·구독 상태 확인, 결과 검증을 분리하여 처리한다. 각 에이전트는 정해진 책임만 수행하고, Supervisor가 전체 성공·재시도·종료를 결정한다.

## 2. 에이전트 정의

| 에이전트 | 입력 | 책임 | 출력 | 금지사항 |
| --- | --- | --- | --- | --- |
| Supervisor | 사용자 요청, 작업 상태 | 작업 분배, 재시도, 종료·알림 판단 | 다음 Handoff, 최종 상태 | 도구 결과를 임의 수정 |
| Search Agent | 좌표, 반경, 필터 | DB/외부 API 장소 검색 | 후보 장소·거리·근거 | 근거 없는 장소 생성 |
| Info Agent | 장소 ID | 운영시간·전화·시설·예약 정보 정리 | 표준 장소 상세 | 원본 정보 변경 |
| Approval Rule Engine | 사장 등록 정보·사업자 인증·중복 후보 | 규칙 기반 자동 승인 또는 검토 대기 분기 | 상태·누락 사유·감사 이벤트 | LLM 임의 승인, 사업자 인증 우회 |
| Reservation Agent (P1) | 장소·방문일·인원 | 예약 링크 또는 문의 생성 | 예약 경로·접수 상태 | 확정되지 않은 예약 확정 처리 |
| Billing Agent (P1) | 구독·결제 이벤트 | 결제 상태 검증, 권한 갱신 | ACTIVE/PAST_DUE 등 상태 | 카드정보 저장, 중복 승인 |
| Incident Triage Agent | 시설 장애 제보 | 유형·심각도 분류, 유사 제보 묶기, 담당 지점 알림 | 분류·우선순위·알림 대상 | 실제 수리 완료 처리 |
| Operations Insight Agent (P1) | 다점포 통계·제보 | 지점별 추이 요약과 개선 신호 탐지 | 운영 리포트 | 근거 없는 매출·원인 단정 |
| Social Content Agent (P1) | 운영자가 등록한 공개 게시물 URL | 공식 임베드 가능 여부 확인 | 표시 가능/링크 폴백 상태 | 게시물 무단 수집·내용 저장·비공개 계정 접근 |
| Critic Agent | 중간 결과·정책 | 필수 값·근거·형식 검증 | PASS/RETRY/FAIL, 사유 | 결과를 임의로 보완 |
| Memory Keeper | 모든 실행 이벤트 | task 상태·실패 원인·캐시 참조 저장 | state ref | 시크릿/원문 결제정보 저장 |

## 3. 공통 Handoff 형식

```json
{
  "task_id": "task-2026-001",
  "from_agent": "search",
  "to_agent": "info",
  "objective": "주변 무인숙소 정보 보강",
  "inputs": { "place_ids": ["p-101"] },
  "constraints": ["근거 없는 운영시간 생성 금지"],
  "expected_output": ["place_detail", "evidence"],
  "attempt": 1,
  "memory_ref": "mem-301"
}
```

## 4. 정상 흐름

```text
Supervisor → Search Agent → Info Agent → Critic Agent
  └─ PASS: 사용자 응답
  └─ RETRY: 실패 사유와 함께 담당 Agent에 재요청
  └─ FAIL: 안전한 대체 안내·관리자 알림
```

P0은 Supervisor, Search, Info, Incident Triage, Critic, Memory Keeper와 Approval Rule Engine을 구현한다.
Approval Rule Engine은 P0의 결정적 규칙 엔진으로 구현하며, Agent의 자연어 판단에 맡기지 않는다.
예약 문의는 P1에서 Reservation Agent를, 테스트 구독 결제는 P1에서 Billing Agent를 별도 경로로 호출한다.
시설 장애 제보는 Incident Triage Agent가 먼저 규칙 기반으로 분류하고, 심각도 높은 제보는 운영자에게 즉시 전달한다. P1의 Operations Insight Agent는 조직·지점 단위 통계를 바탕으로 주간 리포트를 만든다.
P1의 Social Content Agent는 운영자가 직접 입력한 공개 Instagram URL만 검증하고, 임베드가 불가능하면 원본 계정 링크로 폴백한다.

## 5. 종료·재시도 정책

| 상황 | 정책 |
| --- | --- |
| 외부 API 타임아웃 | 최대 2회 지수 백오프 재시도 후 캐시 폴백 |
| DB 읽기 오류 | 재연결 1회 후 읽기 전용 캐시 제공 |
| 필수 정보 누락 | Info Agent 재조회 1회, 계속 누락 시 미확인 표시 |
| 결제 웹훅 지연 (P1) | 결제 상태 조회 후 작업 큐 재처리 |
| 동일 결제 이벤트 (P1) | event ID 멱등성 검사 후 무시 |
| 반복 실패 | 최대 시도 횟수 도달 시 FAIL 및 운영자 알림 |

재시도 요청에는 이전 실패 원인, 이전 시도 횟수, 허용된 대체 경로를 반드시 포함한다. 동일한 입력으로 무한 반복하지 않는다.

## 6. 평가 지표

- Agent별 성공/실패/재시도 횟수
- Handoff 누락률
- 자동 복구율
- 평균 복구 시간(MTTR)
- Critic의 근거 없는 결과 차단 건수
- 각 Agent가 원본 근거와 함께 출력한 비율
- 장애 대응 중 사용자에게 안전한 대체 안내를 제공한 비율

