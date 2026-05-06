---
산출물명: API 명세서
프로젝트: 사내 고객 Q&A 시스템 (Helpdesk Lite)
단계: 06 설계
작성: (수행사) 백엔드 설계자 송OO
검토: (수행사) 아키텍트 한OO / PL 한OO / PM 이OO / (발주사) 정보화 최OO
승인: (발주사) 정보화팀장 정OO
버전: v1.0
작성일: 2026-06-13
---

# API 명세서 — Helpdesk Lite

## 변경이력

| 버전 | 일자 | 작성자 | 변경 내용 |
|---|---|---|---|
| v0.1 | 2026-05-30 | 송OO | 초안 — 11개 API |
| v0.5 | 2026-06-08 | 송OO | 검토 반영 — 취소 API(API-INQ-CANCEL-001) 추가 / 응답 envelope 표준 통일 |
| v1.0 | 2026-06-13 | 송OO | 발주사 승인본 (Mock-Impl Green) |

---

## 1. 공통 표준

### 1.1 베이스 URL·버전

```
{base}/api          (예: https://helpdesk.example.com/api)
```
- 버전 prefix는 v1 도입 전까지 `/api` 사용. 호환성 깨질 시 `/api/v2` 도입.

### 1.2 인증·인가

| 항목 | 정책 |
|---|---|
| 인증 | JWT Bearer Token (HTTP Header `Authorization: Bearer {token}`) |
| Access Token | 30분, payload: { sub, role, name, exp, iat } |
| Refresh Token | 7일, 별도 저장소 (HttpOnly Cookie 권장) |
| 권한 검증 | API Gateway 단에서 RBAC 매트릭스 체크 (역할 + 본인 데이터 한정) |
| 미인증 | 401 UNAUTHORIZED |
| 권한 부족 | 403 FORBIDDEN |

### 1.3 응답 Envelope (전 API 공통)

```json
{
  "isSuccess": true,
  "httpStatus": 200,
  "data": { ... },
  "errors": [],
  "timestamp": "2026-06-13T10:23:45+09:00"
}
```

실패 시:
```json
{
  "isSuccess": false,
  "httpStatus": 400,
  "data": null,
  "errors": [
    { "code": "INVALID_PARAMETER", "field": "title", "message": "제목은 1자 이상 입력하세요." }
  ],
  "timestamp": "2026-06-13T10:23:45+09:00"
}
```

### 1.4 에러 코드 표준 (UPPER_SNAKE_CASE, 숫자 접두사 금지)

| 코드 | HTTP | 의미 |
|---|---|---|
| INVALID_PARAMETER | 400 | 검증 실패 (field·message 동봉) |
| UNAUTHORIZED | 401 | 인증 필요 (또는 토큰 만료) |
| FORBIDDEN | 403 | 권한 부족 |
| NOT_FOUND | 404 | 리소스 없음 |
| CONFLICT | 409 | 동시성 충돌 (낙관적 락) |
| PAYLOAD_TOO_LARGE | 413 | 첨부 크기 초과 |
| UNSUPPORTED_MEDIA_TYPE | 415 | 첨부 확장자 거부 |
| INTERNAL_ERROR | 500 | 서버 오류 (재시도 안내) |

### 1.5 페이지네이션 응답 표준

```json
"data": {
  "items": [ ... ],
  "page": 0,
  "size": 20,
  "total": 137
}
```

### 1.6 날짜·금액 포맷

| 항목 | 포맷 |
|---|---|
| 날짜·일시 | ISO 8601 with offset (`2026-06-13T10:23:45+09:00`) |
| 금액 | (본 시스템 미사용) |
| 파일 크기 | Byte 정수 |

### 1.7 멱등성

| 메서드 | 정책 |
|---|---|
| GET | 자연 멱등 |
| POST (등록) | 클라이언트가 `Idempotency-Key` 헤더 제공 시 중복 차단 (등록·답변 권장) |
| PUT/PATCH | 자연 멱등 |
| DELETE | 자연 멱등 (재호출 시 NOT_FOUND 또는 OK) |

---

## 2. API 목록 (12건)

| # | API ID | 메서드 | 경로 | 설명 | 권한 | 관련 화면 / 요구 |
|---|---|---|---|---|---|---|
| 1 | API-AUTH-LOGIN-001 | POST | /api/auth/login | 로그인 | 비로그인 | SCR-AUTH-001 / NFR-INQ-003 |
| 2 | API-AUTH-REFRESH-001 | POST | /api/auth/refresh | 토큰 갱신 | Refresh Token | (전 화면) |
| 3 | API-AUTH-LOGOUT-001 | POST | /api/auth/logout | 로그아웃 | 인증 | (전 화면) |
| 4 | API-INQ-CREATE-001 | POST | /api/inquiries | 문의 등록 | GUEST 이상 | SCR-INQ-FORM-001 / FR-INQ-001 |
| 5 | API-INQ-LIST-001 | GET | /api/inquiries | 전체 문의 검색 | AGENT 이상 | SCR-INQ-LIST-001 / FR-INQ-003 |
| 6 | API-INQ-MY-001 | GET | /api/my/inquiries | 본인 문의 검색 | GUEST 이상 (본인) | SCR-INQ-LIST-002 / FR-INQ-003 |
| 7 | API-INQ-DETAIL-001 | GET | /api/inquiries/{id} | 문의 상세 | 권한·소유자 검증 | SCR-INQ-DTL-001 |
| 8 | API-INQ-CANCEL-001 | DELETE | /api/inquiries/{id} | 문의 취소 (논리) | GUEST (본인, NEW만) | SCR-INQ-DTL-001 / FR-INQ-006 |
| 9 | API-INQ-ANSWER-001 | POST | /api/inquiries/{id}/answers | 답변 등록 | AGENT 이상 | SCR-INQ-DTL-001 / FR-INQ-002 |
| 10 | API-INQ-ATT-DL-001 | GET | /api/inquiries/{id}/attachments/{aid}/download | 첨부 다운로드 | 권한·소유자 검증 | SCR-INQ-DTL-001 |
| 11 | API-INQ-STAT-001 | GET | /api/stats/sla | SLA 통계 조회 | ADMIN | SCR-INQ-STAT-001 / FR-INQ-005 |
| 12 | API-CMN-CAT-001 | GET | /api/categories | 카테고리 코드 조회 | 인증 | SCR-INQ-FORM-001, LIST |

---

## 3. API 상세 명세

### 3.1 API-AUTH-LOGIN-001 — 로그인

| 항목 | 내용 |
|---|---|
| 메서드·URL | POST /api/auth/login |
| 인증·권한 | 비로그인 |
| Body | `{ "loginId": "string", "password": "string" }` |
| Validation | loginId 1~50자, password 8~64자 |
| 응답 (200) | `data: { "accessToken": "...", "refreshToken": "...", "user": { "userId", "name", "role" } }` |
| 응답 (4xx) | INVALID_PARAMETER (검증 실패), UNAUTHORIZED (자격 불일치) |
| 보안 | (1) BCrypt 비교 (2) 5회 연속 실패 시 30분 잠금 (3) 로그인 시도 감사 로그 |
| 성능 SLA | 95%ile ≤ 0.3초 |

### 3.2 API-AUTH-REFRESH-001 — 토큰 갱신

| 항목 | 내용 |
|---|---|
| 메서드·URL | POST /api/auth/refresh |
| 인증·권한 | Refresh Token (HttpOnly Cookie 권장) |
| Body | (Cookie 기반이면 빈 Body) 또는 `{ "refreshToken": "..." }` |
| 응답 (200) | `data: { "accessToken": "..." }` |
| 응답 (4xx) | UNAUTHORIZED (Refresh Token 만료·위조) |
| 멱등성 | GET 아님이지만 동일 Refresh로 재호출 시 동일 효과 |

### 3.3 API-AUTH-LOGOUT-001 — 로그아웃

| 항목 | 내용 |
|---|---|
| 메서드·URL | POST /api/auth/logout |
| 인증·권한 | 인증 |
| 응답 (200) | `data: { "loggedOut": true }` |
| 동작 | Refresh Token 폐기 (블랙리스트 또는 DB 삭제) |

### 3.4 API-INQ-CREATE-001 — 문의 등록 (FR-INQ-001)

| 항목 | 내용 |
|---|---|
| 메서드·URL | POST /api/inquiries |
| 인증·권한 | GUEST 이상 |
| Headers | `Idempotency-Key: {uuid}` (선택, 권장) |
| Body | `{ "title": "string", "content": "string", "categoryCd": "string", "attachmentIds": [int] }` |
| Validation | title 1~200자, content 1~5000자, categoryCd CATEGORY 코드값, attachmentIds ≤ 5개 |
| 응답 (201) | `data: { "inquiryId": 137, "status": "NEW", "slaDueDt": "2026-06-11T14:32:00+09:00" }` |
| 응답 (4xx) | INVALID_PARAMETER, UNAUTHORIZED |
| 동시성 | Idempotency-Key 기반 중복 차단 (1분 내 동일 키는 동일 inquiry 반환) |
| 트랜잭션 | INQUIRY INSERT + ATTACHMENT 매핑 1트랜잭션 (DB설계서 §6) |
| 후속 동작 | 발송 큐에 SLA 알림 작업 적재 (status=NEW, sla_due_dt-4h) |
| 성능 SLA | 95%ile ≤ 0.5초 |

### 3.5 API-INQ-LIST-001 — 전체 문의 검색 (FR-INQ-003, AGENT 이상)

| 항목 | 내용 |
|---|---|
| 메서드·URL | GET /api/inquiries |
| 인증·권한 | AGENT 이상 |
| Query | status, categoryCd, regDtFrom, regDtTo, keyword, page (≥0), size (1~100), sort (`reg_dt,desc` 등) |
| Validation | status ∈ {NEW,ANSWERED,REOPENED,CLOSED,CANCELLED}; size 1~100; regDtFrom ≤ regDtTo |
| 응답 (200) | `data: { items: [{inquiryId, title, categoryCd, status, slaDueDt, regId(마스킹), regDt}], page, size, total }` |
| 응답 (4xx) | INVALID_PARAMETER, UNAUTHORIZED, FORBIDDEN |
| 보안 | regId·email 마스킹 적용 (NFR-INQ-003) |
| 인덱스 | IX_INQUIRY_STATUS_SLA, IX_INQUIRY_CATEGORY 활용 |
| 성능 SLA | 95%ile ≤ 0.5초 @ 100 TPS |

### 3.6 API-INQ-MY-001 — 본인 문의 검색 (FR-INQ-003, GUEST)

| 항목 | 내용 |
|---|---|
| 메서드·URL | GET /api/my/inquiries |
| 인증·권한 | 인증 사용자 (GUEST·AGENT·ADMIN). 단, AGENT/ADMIN도 본인 등록건만 반환 |
| Query | (API-INQ-LIST-001과 동일) — 단 reg_id는 자동 주입 (Query 파라미터 거부) |
| 보안 | reg_id 자동 주입 (인증 사용자 ID로 강제 필터). Query에 reg_id 시 무시 |
| 응답 | API-INQ-LIST-001과 동일 형식 |

### 3.7 API-INQ-DETAIL-001 — 문의 상세

| 항목 | 내용 |
|---|---|
| 메서드·URL | GET /api/inquiries/{id} |
| 인증·권한 | 인증 + (GUEST는 본인만 / AGENT·ADMIN은 전체) |
| Path | id: 문의 ID (BIGINT) |
| 응답 (200) | `data: { inquiry: {...}, answers: [...], attachments: [...] }` (answers 배열 — REOPENED 1:N 대응) |
| 응답 (4xx) | UNAUTHORIZED, FORBIDDEN, NOT_FOUND |
| 보안 | reg_id != 인증사용자 AND role=GUEST 시 403. AGENT/ADMIN은 전체 조회. 마스킹 정책 적용. |

### 3.8 API-INQ-CANCEL-001 — 문의 취소 (FR-INQ-006)

| 항목 | 내용 |
|---|---|
| 메서드·URL | DELETE /api/inquiries/{id} |
| 인증·권한 | GUEST (본인, NEW 상태) — AGENT/ADMIN은 별도 관리 API (본 명세 외) |
| Path | id: 문의 ID |
| Body | (없음) |
| 응답 (200) | `data: { "inquiryId": 137, "status": "CANCELLED" }` |
| 응답 (4xx) | UNAUTHORIZED / FORBIDDEN (본인 아님) / CONFLICT (status != NEW) / NOT_FOUND |
| 동시성 | 낙관적 락 (VERSION 검사) — 충돌 시 CONFLICT |
| 트랜잭션 | INQUIRY UPDATE (status='CANCELLED', mod_id, mod_dt, version+1) |
| 후속 동작 | SLA 알림 큐에서 해당 inquiry 제거 (트랜잭션 외 비동기) |
| 성능 SLA | 95%ile ≤ 0.3초 |

### 3.9 API-INQ-ANSWER-001 — 답변 등록 (FR-INQ-002)

| 항목 | 내용 |
|---|---|
| 메서드·URL | POST /api/inquiries/{id}/answers |
| 인증·권한 | AGENT 이상 |
| Headers | `Idempotency-Key: {uuid}` (권장) |
| Path | id: 문의 ID |
| Body | `{ "content": "string" }` |
| Validation | content 1~5000자, 문의 status ∈ {NEW, REOPENED} |
| 응답 (201) | `data: { "answerId": 9001, "inquiryStatus": "ANSWERED" }` |
| 응답 (4xx) | INVALID_PARAMETER / FORBIDDEN / CONFLICT (동시 답변 또는 잘못된 상태) / NOT_FOUND |
| 동시성 | 낙관적 락 (INQUIRY.VERSION 검사) — 충돌 시 CONFLICT |
| 트랜잭션 | ANSWER INSERT + INQUIRY UPDATE (status='ANSWERED', mod_id, mod_dt, version+1) — 1트랜잭션 |
| 후속 동작 | (1) GUEST 이메일 발송 (트랜잭션 외 비동기, 재시도 3회) (2) SLA 알림 큐에서 해당 inquiry 제거 |
| 성능 SLA | 95%ile ≤ 0.5초 |
| SLA 정책 | 본 답변이 최초 1차 답변일 때만 SLA 측정 종료. REOPENED 시 추가 답변은 SLA 재측정 없음 (요구사항정의서 v1.1) |

### 3.10 API-INQ-ATT-DL-001 — 첨부 다운로드

| 항목 | 내용 |
|---|---|
| 메서드·URL | GET /api/inquiries/{id}/attachments/{aid}/download |
| 인증·권한 | 인증 + (GUEST는 본인 문의만 / AGENT·ADMIN은 전체) |
| Path | id: 문의 ID, aid: 첨부 ID |
| 응답 (200) | (binary) `Content-Disposition: attachment; filename="..."` + Content-Type |
| 응답 (4xx) | UNAUTHORIZED, FORBIDDEN, NOT_FOUND |
| 보안 | 파일 경로 직접 접근 금지 (서버에서 검증 후 스트림 응답). Content-Disposition 인코딩 (RFC 6266 utf-8) |

### 3.11 API-INQ-STAT-001 — SLA 통계 (FR-INQ-005, ADMIN)

| 항목 | 내용 |
|---|---|
| 메서드·URL | GET /api/stats/sla |
| 인증·권한 | ADMIN |
| Query | period (D/W/M, 필수), startDt, endDt, categoryCd (선택) |
| Validation | startDt ≤ endDt, 최대 1년 |
| 응답 (200) | `data: { summary: { received, answered, slaCompliance, avgFirstResponseHours }, series: [{ date, received, answered, slaMissed }], byCategory: [{categoryCd, count, ratio}] }` |
| 응답 (4xx) | INVALID_PARAMETER, FORBIDDEN |
| 동작 | 사전 집계 테이블 + 당일 분 합산 (DB설계서 §3 보완 — 별도 통계 테이블은 운영 정책에 따라) |
| 보안 | 마스킹 정책: 카테고리·집계만 노출, 개별 inquiry·user 정보 X |
| 성능 SLA | 95%ile ≤ 1초 |

### 3.12 API-CMN-CAT-001 — 카테고리 코드 조회

| 항목 | 내용 |
|---|---|
| 메서드·URL | GET /api/categories |
| 인증·권한 | 인증 사용자 |
| Query | (없음) |
| 응답 (200) | `data: { items: [{ categoryCd, categoryNm, sortOrd }] }` (use_yn='Y'만, sort_ord 정렬) |
| 캐시 | 클라이언트 사이드 5분 캐시 권장 (코드값 변경 빈도 낮음) |

---

## 4. 보안 공통 정책

| 항목 | 정책 |
|---|---|
| 전송 | TLS 1.2+ (HTTPS only) |
| 입력 검증 | 모든 Body·Query·Path 파라미터 — 화이트리스트 검증 |
| XSS | 응답 시 HTML 이스케이프, Content-Type `application/json; charset=utf-8` 강제 |
| SQL Injection | Prepared Statement / ORM Bind Variable 의무화 |
| CSRF | SameSite=Strict Cookie 또는 Double Submit Cookie |
| 첨부 업로드 | 확장자 화이트리스트 + Magic Number 검증 + 10MB 제한 |
| 마스킹 | 응답 시 reg_id/email 마스킹 (NFR-INQ-003). ADMIN은 별도 권한으로 원본 조회 가능 (감사 로그 적재) |
| Rate Limit | 사용자별 분당 60req (DDoS·Brute Force 방어) |
| 감사 로그 | 인증·권한 변경·취소·통계 조회는 감사 로그 적재 (1년 보존) |

---

## 5. OpenAPI (Swagger) 스펙 위치

본 명세는 학습용 요약본이며, 실제 OpenAPI YAML은 `/docs/openapi.yaml`에 위치한다 (개발 단계 산출). 자동 생성 도구로 검증 가능하도록 본 명세의 모든 항목이 OpenAPI 3.x 표기법으로 매핑된다.

---

## 6. Mock-Impl 롤플레이 결과 (Green 판정)

리뷰어가 개발자 페르소나로 API-INQ-ANSWER-001 의사코드 작성 시도 → 질문 0개 → **Green**.

| 점검 | 결과 |
|---|---|
| 메서드·URL·Path/Query/Body 전수 | ✅ |
| Validation 명시 (길이·범위·NULL·정규식) | ✅ |
| 응답 envelope·상태코드·에러코드 | ✅ |
| 동시성 (낙관적 락·멱등성 키) | ✅ |
| 트랜잭션 경계 + 비동기 후속 동작 | ✅ |
| 권한 매트릭스 (역할 + 본인 데이터 검증) | ✅ |

---

## 7. 후속 단계 영향

- **07 개발**: 본 명세 기반 컨트롤러·서비스·DTO 구현 + 단위테스트(UT-INQ-001~030)
- **08 테스트**: 통합테스트 시나리오에서 API 호출 시퀀스로 검증 (특히 IT-INQ-SLA-007)
- **09 인수인계**: OpenAPI YAML + 본 명세서 v1.0이 산출물 목록에 등재

---

## 8. 승인

| 구분 | 소속·직위 | 성명 | 서명·날인 | 일자 |
|---|---|---|---|---|
| 발주사 작성 검토 | 정보화팀 | 최OO | | 2026-06-13 |
| 발주사 승인 | 정보화팀장 | 정OO | | 2026-06-13 |
| 수행사 작성 | 백엔드 설계자 | 송OO | | 2026-06-13 |
| 수행사 검토 | 아키텍트 | 한OO | | 2026-06-13 |
| 수행사 검토 | PM | 이OO | | 2026-06-13 |

---

*본 API명세서 v1.0은 SSOT 폴더에 영구 지정. 변경은 변경관리 절차로만 가능. 호환성 깨질 변경은 v2 prefix(`/api/v2`)로 분리.*
