아래는 **보완 완료된 최종 SSOT 문서**입니다.
(실무 프로젝트 착수 기준으로 바로 사용 가능 수준)

---

```markdown
---
문서명: 샘플프로젝트 — 고객 Q&A 시스템 (보완본)
용도: 단계 파일들의 산출물 샘플(§5)이 일관되게 참조하는 SSOT
작성: 2026-05-06 (Rev.2)
---

# 샘플프로젝트 — 고객 Q&A 시스템 (Helpdesk Lite)

---

## 1. 개요

| 항목 | 값 |
|---|---|
| 프로젝트명 | 사내 고객 Q&A 시스템 (Helpdesk Lite) |
| 발주사 | (가상) 중견 제조업체 |
| 배경 | 분산된 문의 채널 통합 및 SLA 관리 |
| 일정 | 4개월 |
| 인력 | 6명 |
| 계약금 | 4억 |

---

## 2. 범위

| 구분 | 내용 |
|---|---|
| In | 문의 등록/수정/취소, 답변, 검색, 알림, SLA |
| Out | AI 자동응답, 외부 고객 포털 |

---

## 3. 역할 및 권한 (RBAC)

| 역할 | 권한 |
|---|---|
| GUEST | 본인 문의 CRUD |
| AGENT | 전체 조회, 답변 작성/수정 |
| ADMIN | 전체 + 통계 + 사용자 관리 |

### 인증/인가
- JWT Access Token (30분)
- Refresh Token (7일)
- BCrypt 비밀번호 저장
- API Gateway 단 RBAC 체크

---

## 4. 핵심 엔티티

### 4.1 INQUIRY
- inquiry_id (PK)
- title
- content
- status
- category_cd
- sla_due_dt
- reg_id
- reg_dt
- upd_dt

### 4.2 ANSWER
- answer_id (PK)
- inquiry_id (FK)
- content
- agent_id
- reg_dt
- upd_dt

### 4.3 INQUIRY_HIST (신규)
- hist_id (PK)
- inquiry_id
- status
- changed_by
- changed_dt

### 4.4 ATTACHMENT
- attachment_id
- inquiry_id
- file_name
- file_path
- file_size
- content_type

### 4.5 USER
- user_id
- name
- role
- dept
- status (ACTIVE/INACTIVE)
- email

### 4.6 CATEGORY
- category_cd
- category_nm

---

## 5. 상태머신 (개선)

```

NEW
→ ANSWERED
→ (고객 추가 질문) → REOPENED
→ ANSWERED
→ CLOSED

NEW → CANCELLED

```

### SLA 기준
- 최초 ANSWERED 기준 24시간
- REOPENED 시 SLA 재시작

---

## 6. 핵심 기능 (확장)

| ID | 기능 |
|---|---|
| FR-INQ-001 | 문의 등록 |
| FR-INQ-002 | 답변 작성 |
| FR-INQ-003 | 문의 검색 |
| FR-INQ-004 | SLA 알림 |
| FR-INQ-005 | SLA 통계 |
| FR-INQ-006 | 문의 수정 |
| FR-INQ-007 | 문의 취소 |
| FR-INQ-008 | 답변 수정 |

---

## 7. 알림 정책

| 이벤트 | 대상 | 방식 |
|---|---|---|
| SLA 4시간 전 | AGENT | 이메일 |
| 답변 등록 | GUEST | 이메일 |
| 상태 변경 | 관련자 | 이메일 |

---

## 8. 비기능 요구사항 (NFR)

| 항목 | 값 |
|---|---|
| 동시 사용자 | 200 (Peak 기준) |
| API 응답 | 0.5초 이하 (95%ile) |
| 화면 응답 | 1초 이하 (95%ile) |
| 가용률 | 99.5% |
| SLA | 24시간 (최초 응답 기준) |

---

## 9. 보안

- JWT + Refresh Token
- RBAC 기반 접근제어
- 파일 업로드 제한 (10MB, 확장자 whitelist)
- XSS / SQL Injection 방어

---

## 10. 운영 정책

### 로그
- API 요청/응답 로그
- 상태 변경 로그

### 배치
- SLA 알림 스케줄러 (1시간 주기)

### 백업
- DB 일 1회
- 파일 주 1회

---

## 11. 외부 연계

| 시스템 | 방식 |
|---|---|
| SMTP | 이메일 발송 |

---

## 12. ID 체계

| 구분 | 형식 |
|---|---|
| 요구사항 | FR-INQ-XXX |
| 화면 | SCR-INQ-XXX |
| API | API-INQ-XXX |
| 테스트 | UT / IT |
| 결함 | DEF-INQ |

---

## 13. 확장 고려사항

- AI 자동답변 (Phase 2)
- 챗봇 연계
- 멀티 테넌시

---

```
