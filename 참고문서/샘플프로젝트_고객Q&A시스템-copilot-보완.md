## ✅ 검토 요약 (보완/개선 포인트)

다음 항목을 중심으로 실무 적용 관점에서 보완했습니다.

1.  **범위 명확화**
    *   In/Out 범위 구체화 및 Phase2 경계 명확 처리

2.  **권한/인증 강화**
    *   JWT, 세션 정책 구체화
    *   역할별 데이터 접근 범위 명확화

3.  **엔티티 정합성 보완**
    *   FK 관계 및 필수 컬럼 보강
    *   USER / ANSWER / ATTACHMENT 관계 명확화

4.  **상태머신 예외 케이스 반영**
    *   SLA 초과 / 재오픈 흐름 추가

5.  **기능 정의 정밀화**
    *   각 기능별 주요 처리 로직 및 트리거 조건 보강

6.  **NFR 실무 기준 보정**
    *   성능 및 장애 대응 지표 추가 (Recovery, 로그 등)

7.  **운영/배포 고려사항 추가**
    *   로그, 모니터링, 알림 등 최소 운영요건 포함

8.  **테스트/추적성 강화**
    *   ID 체계 활용 예시 일부 반영

***

## 🧾 보완된 전체 텍스트 (최종본)

```markdown
---
문서명: 샘플프로젝트 — 고객 Q&A 시스템
용도: 단계 파일들의 산출물 샘플(§5)이 일관되게 참조하는 가상 프로젝트 (SSOT)
작성: 2026-05-06
버전: v1.1 (보완)
---

# 샘플프로젝트 — 고객 Q&A 시스템 (가칭 "Helpdesk Lite")

> 단계별 파일의 §5 산출물 샘플은 모두 본 정의를 기준으로 한다. 학습용 가상 프로젝트이며, 실무 기준 최소 수준으로 단순화했다.

---

## 1. 개요

| 항목 | 값 |
|---|---|
| 프로젝트명 | 사내 고객 Q&A 시스템 (가칭 **Helpdesk Lite**) |
| 발주사 | (가상) 중견 제조업체 1곳 |
| 배경 | 메일·전화 등 분산 채널 운영 → 문의 누락 및 중복 대응 발생 → 통합 관리 필요 |
| 일정 | 4개월 (분석 1 + 설계 1 + 개발 1.5 + 테스트 0.5) |
| 인력 | 6명 (PM 1, 분석·설계 2, 개발 2, QA 1) |
| 계약금 | (가상) 4억 |

---

## 2. 범위

| 구분 | 내용 |
|---|---|
| In Scope | 문의 등록 / 답변 처리 / 검색 / 이메일 알림 / SLA 통계 |
| Out Scope | AI 자동답변, 챗봇, 외부 고객사 직접 접속 (Phase 2) |

> Phase 2에서 고객 포털 및 AI 응답 기능 추가 예정

---

## 3. 역할 및 권한

| 역할 | 핵심 권한 |
|---|---|
| GUEST | 문의 등록, 본인 문의 및 답변 조회 |
| AGENT (상담사) | 전체 문의 조회, 답변 작성, 상태 변경 |
| ADMIN | 전체 조회/관리, SLA 통계, 사용자 관리 |

### 인증 및 세션 정책
- 인증 방식: ID/PW (BCrypt 해시 저장)
- 세션 방식: JWT 기반
- 세션 타임아웃: 30분 (무활동 기준)
- 재발급: Refresh Token 사용 가능 (선택)

### 데이터 접근 정책
- GUEST: 본인 문의만 접근
- AGENT: 전체 문의 접근 가능
- ADMIN: 모든 데이터 + 관리 기능

---

## 4. 핵심 엔티티

### 4.1 엔티티 목록

| 엔티티 | 핵심 속성 |
|---|---|
| **INQUIRY** | inquiry_id(PK), title, content, status, category_cd, sla_due_dt, reg_id(FK USER), reg_dt |
| **ANSWER** | answer_id(PK), inquiry_id(FK), content, agent_id(FK USER), reg_dt |
| **ATTACHMENT** | attachment_id(PK), inquiry_id(FK), file_name, file_size, content_type, reg_dt |
| **USER** | user_id(PK), name, role, email |
| **CATEGORY** | category_cd(PK), category_nm |

### 4.2 관계
- INQUIRY 1:N ANSWER
- INQUIRY 1:N ATTACHMENT
- USER 1:N INQUIRY (작성자)
- USER 1:N ANSWER (상담사)

### 4.3 코드 예

| 코드 | 설명 |
|---|---|
| PRODUCT | 제품문의 |
| DELIVERY | 배송문의 |
| REFUND | 환불문의 |
| ETC | 기타 |

---

## 5. 상태머신 (문의)

```

\[NEW] ──상담사 답변──> \[ANSWERED]
│                        │
│                        └─고객 확인 or 7일 경과──> \[CLOSED]
│
└──고객 취소──> \[CANCELLED]

\[ANSWERED] ──고객 재문의──> \[NEW]
\[NEW|ANSWERED] ──SLA 초과──> \[DELAYED]

```

### 상태 설명
- NEW: 신규 접수
- ANSWERED: 답변 완료
- CLOSED: 종결
- CANCELLED: 취소
- DELAYED: SLA 초과

---

## 6. 핵심 기능

| 기능 ID | 기능명 | 주요 내용 |
|---|---|---|
| FR-INQ-001 | 문의 등록 | 제목, 내용, 카테고리, 첨부 업로드 |
| FR-INQ-002 | 답변 작성 | 상담사 답변 등록, 상태 변경 |
| FR-INQ-003 | 문의 검색 | 키워드, 카테고리, 상태, 기간 필터 |
| FR-INQ-004 | SLA 알림 | 마감 4시간 전 이메일 자동 발송 |
| FR-INQ-005 | SLA 통계 | 일/주/월 기준 처리율 및 평균 응답시간 |

### SLA 기준
- 기본 응답 시간: 24시간
- SLA 기준 초과 시 DELAYED 상태 표시

---

## 7. 비기능 요구사항 (NFR)

| 항목 | 값 |
|---|---|
| 동시 사용자 | 200 |
| 화면 응답 (95%ile) | 1초 |
| API 응답 (95%ile) | 0.5초 |
| 가용률 | 99.5% (월 기준) |
| SLA 응답 | 24시간 |
| 첨부 파일 제한 | 10MB / 파일 |

### 추가 운영 기준
- 로그 보관기간: 3개월
- 장애 대응 (MTTR): 4시간 내 복구 목표
- 모니터링: API 응답시간 / 오류율 / SLA 실패율

---

## 8. 외부 연계

| 시스템 | 방식 | 설명 |
|---|---|---|
| SMTP 이메일 서버 | SMTP | SLA 알림 / 답변 완료 알림 |

---

## 9. ID 체계

| 분류 | 형식 | 예 |
|---|---|---|
| 요구사항 | FR-INQ-XXX / NFR-INQ-XXX | FR-INQ-004 |
| 화면 | SCR-INQ-XXX | SCR-INQ-LIST-001 |
| API | API-INQ-XXX | API-INQ-LIST-001 |
| 테이블 | 대문자 단수 | INQUIRY |
| 단위테스트 | UT-INQ-XXX | UT-INQ-001 |
| 통합테스트 | IT-INQ-XXX | IT-INQ-SLA-007 |
| 결함 | DEF-INQ-NNN | DEF-INQ-007 |

---

## 10. 운영 고려사항 (추가)

- 배포 방식: 무중단 배포 (Blue/Green 권장)
- 백업 정책: 일 1회 DB 백업
- 보안:
  - 파일 업로드 확장자 제한
  - SQL Injection / XSS 방어
- 감사 로그:
  - 문의 상태 변경 이력 기록

---


