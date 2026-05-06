---
문서명: 샘플프로젝트 — 고객 Q&A 시스템
용도: 단계 파일들의 산출물 샘플(§5)이 일관되게 참조하는 가상 프로젝트 (SSOT)
작성: 2026-05-06
---

# 샘플프로젝트 — 고객 Q&A 시스템 (가칭 "Helpdesk Lite")

> 단계별 파일의 §5 산출물 샘플은 모두 본 정의를 기준으로 한다. 학습용 가상 프로젝트이며, 최대한 단순화했다.

---

## 1. 개요

| 항목 | 값 |
|---|---|
| 프로젝트명 | 사내 고객 Q&A 시스템 (가칭 **Helpdesk Lite**) |
| 발주사 | (가상) 중견 제조업체 1곳 |
| 배경 | 메일·전화 분산 채널 → 누락·중복답변 → 통합 채널 필요 |
| 일정 | 4개월 (분석 1 + 설계 1 + 개발 1.5 + 테스트 0.5) |
| 인력 | 6명 (PM 1, 분석·설계 2, 개발 2, QA 1) |
| 계약금 | (가상) 4억 |

> **기술 스택 (권장 예시)**: FE 모던 SPA / BE Spring Boot 또는 Node.js / DB PostgreSQL 또는 MySQL / Infra 클라우드.
>
> ⚠️ **본 SSOT는 학습용 비종속 표기**. 실 프로젝트 적용 시 발주사 표준 또는 사업 합의 결과로 기술 스택 확정 — 본 SSOT는 도메인·요구사항 정의가 중심이며 기술 비종속이 학습 가치.

## 2. 범위

| 구분 | 내용 |
|---|---|
| In | 문의 등록 / 답변 / 검색 / 이메일 알림 / SLA 통계 |
| Out | AI 자동답변, 챗봇, 외부 고객사 직접 접속 (Phase 2) |

## 3. 역할 (3등급)

| 역할 | 핵심 권한 |
|---|---|
| GUEST | 문의 등록, 본인 문의·답변 조회 |
| AGENT (상담사) | 모든 문의 조회, 답변 작성 |
| ADMIN | 전체 + 통계 + 사용자 관리 |

> **인증·세션**: ID/PW 자체 인증 (BCrypt 저장) / JWT Access Token 30분 + Refresh Token 7일

## 4. 핵심 엔티티 (5개)

| 엔티티 | 핵심 속성 |
|---|---|
| **INQUIRY** | inquiry_id(PK), title, content, status, category_cd(FK), sla_due_dt, reg_id(FK→USER), reg_dt |
| **ANSWER** | answer_id(PK), inquiry_id(FK→INQUIRY), content, agent_id(FK→USER), reg_dt |
| **ATTACHMENT** | attachment_id(PK), inquiry_id(FK→INQUIRY), file_name, file_path, file_size, content_type, reg_dt |
| **USER** | user_id(PK), login_id, password(BCrypt), name, role, dept, email |
| **CATEGORY** (코드) | category_cd(PK), category_nm |

> **관계**: INQUIRY 1:N ANSWER / INQUIRY 1:N ATTACHMENT / USER 1:N INQUIRY (작성자) / USER 1:N ANSWER (상담사)
>
> **카테고리 코드 예**: 제품문의(PRODUCT) / 배송문의(DELIVERY) / 환불문의(REFUND) / 기타(ETC)

## 5. 상태머신 (문의)

```
[NEW] ──상담사 답변──> [ANSWERED] ──고객 확인 or 7일 경과──> [CLOSED]
   │                       │
   │                       └──고객 추가질문──> [REOPENED] ──상담사 재답변──> [ANSWERED]
   │
   └──고객 취소──> [CANCELLED]
```

> **REOPENED**: 답변 후 7일 이내 고객이 추가 질문 등록 시. **SLA는 최초 접수 시점 기준 24시간이며, REOPENED 시 재기산하지 않는다.**

## 6. 핵심 기능 6건

| 기능 ID | 기능명 |
|---|---|
| FR-INQ-001 | 문의 등록 (제목·내용·카테고리·첨부) |
| FR-INQ-002 | 답변 작성 (상담사) — REOPENED 시 추가 답변 허용 |
| FR-INQ-003 | 문의 검색 (키워드·카테고리·상태) |
| FR-INQ-004 | SLA 임박 자동 이메일 알림 (마감 4시간 전) |
| FR-INQ-005 | SLA 통계 (일·주·월 단위) |
| FR-INQ-006 | 문의 취소 (GUEST, NEW 상태에서만) |

## 7. 비기능 정량 (NFR)

| 분류 | 항목 | 값 |
|---|---|---|
| 성능 | 동시 사용자 | 200 |
| 성능 | 화면 95%ile 응답 | 1초 |
| 성능 | API 95%ile 응답 | 0.5초 |
| 가용성 | 가용률 (월간) | 99.5% |
| 가용성 | MTTR (장애 복구 목표) | 4시간 |
| 업무 | SLA 응답 | 24시간 |
| 업무 | 첨부 크기 | 10MB / 파일 |
| 보안 | 비밀번호 저장 | 단방향 (BCrypt) |
| 보안 | 개인정보 마스킹 | 화면·로그·내보내기 일괄 적용 (이메일·전화번호 등) |
| 보안 | 입력 검증 | XSS / SQL Injection 방어, 첨부 확장자 화이트리스트 |
| 운영 | 로그 보존 | 3개월 (감사 로그는 1년) |
| 운영 | 모니터링 | API 응답시간·오류율·SLA 미달율 대시보드 |

## 8. 외부 연계 (1건만)

| 시스템 | 방식 |
|---|---|
| 사내 이메일 서버 (SMTP) | 알림 발송 (SLA 임박·답변 완료) |

## 9. ID 체계 (단계 간 추적용)

| 분류 | 형식 | 예 |
|---|---|---|
| 요구사항 | `FR-INQ-XXX` / `NFR-INQ-XXX` | FR-INQ-004 |
| 화면 | `SCR-INQ-XXX` | SCR-INQ-LIST-001 |
| API | `API-INQ-XXX` | API-INQ-LIST-001 |
| 테이블 | (대문자 단수) | INQUIRY |
| 단위테스트 | `UT-INQ-XXX` | UT-INQ-001 |
| 통합테스트 | `IT-INQ-XXX` | IT-INQ-SLA-007 |
| 결함 | `DEF-INQ-NNN` | DEF-INQ-007 |

---

*본 문서는 학습용 가상 프로젝트 정의서일 뿐, 실제 발주·계약 산출물이 아니다.*
