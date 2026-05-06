---
산출물명: DB설계서 (ERD + 테이블정의서 + 공통코드)
프로젝트: 사내 고객 Q&A 시스템 (Helpdesk Lite)
단계: 06 설계
작성: (수행사) DBA 박OO
검토: (수행사) 아키텍트 한OO / PM 이OO / (발주사) 정보화 최OO
승인: (발주사) 정보화팀장 정OO
버전: v1.0
작성일: 2026-06-10
---

# DB설계서 — Helpdesk Lite

## 변경이력

| 버전 | 일자 | 작성자 | 변경 내용 |
|---|---|---|---|
| v0.1 | 2026-05-25 | 박OO | 초안 — 5개 엔티티 + 공통코드 |
| v0.5 | 2026-06-03 | 박OO | 인덱스 설계·낙관적 락 컬럼·감사 정책 반영 |
| v1.0 | 2026-06-10 | 박OO | 발주사 승인본 (Mock-Impl 롤플레이 Green) |

---

## 1. 표준 (06 설계 단계 §4-1 표준 고정 따름)

> ⚠️ **본 DB설계서는 학습용 가상 프로젝트(Helpdesk Lite) 기준이며 DBMS 비종속 표기**. PostgreSQL/MySQL/Oracle 중 어느 것이라도 적용 가능한 표준 SQL 기준. 실 프로젝트 적용 시 발주사 DB 표준에 맞춰 (1) 데이터 타입 (예: TIMESTAMP vs DATETIME) (2) Identity 방식 (예: SERIAL vs AUTO_INCREMENT vs SEQUENCE) (3) 인덱스 옵션·FULLTEXT 등 — 발주사 사내 표준에 맞춰 변환 필요.

### 1.1 명명 규칙

| 항목 | 규칙 | 예 |
|---|---|---|
| 테이블명 | UPPER_SNAKE_CASE, 단수형 | INQUIRY, USER, CATEGORY |
| 컬럼명 | UPPER_SNAKE_CASE | INQUIRY_ID, REG_DT |
| PK | `{테이블명}_ID` (Identity 또는 Sequence) | INQUIRY_ID |
| FK | 참조 PK명 동일 | (ANSWER 테이블의) INQUIRY_ID |
| 인덱스 | `IX_{테이블}_{선행컬럼}` 또는 `UK_{컬럼}` | IX_INQUIRY_STATUS_SLA, UK_USER_LOGIN_ID |
| 코드 컬럼 | `{도메인}_CD` | CATEGORY_CD, STATUS |
| 명칭 컬럼 | `{도메인}_NM` | CATEGORY_NM |

### 1.2 데이터 타입 표준

| 분류 | 타입 |
|---|---|
| ID (Surrogate) | BIGINT |
| 코드 | VARCHAR(20) |
| 이름·라벨 | VARCHAR(100) |
| 제목 | VARCHAR(200) |
| 본문·내용 | TEXT |
| 일시 | TIMESTAMP (KST 저장, ISO 8601 처리) |
| 플래그 (Y/N) | CHAR(1) |
| 금액 | DECIMAL(18,0) — 본 시스템 미사용 |
| 버전 (낙관적 락) | BIGINT |

### 1.3 공통 컬럼 (모든 업무 테이블 공통)

| 컬럼 | 타입 | NULL | 기본값 | 의미 |
|---|---|---|---|---|
| REG_ID | VARCHAR(50) | N | — | 등록자 (USER.user_id) |
| REG_DT | TIMESTAMP | N | CURRENT_TIMESTAMP | 등록일시 |
| MOD_ID | VARCHAR(50) | Y | — | 최종 수정자 |
| MOD_DT | TIMESTAMP | Y | — | 최종 수정일시 |
| USE_YN | CHAR(1) | N | 'Y' | 논리삭제 (감사 추적용) |
| VERSION | BIGINT | N | 0 | 낙관적 락 버전 |

> 코드성 테이블(CATEGORY)은 USE_YN·VERSION 보유, MOD_ID/MOD_DT는 선택. 첨부 테이블(ATTACHMENT)은 INSERT 후 변경 없음 가정으로 MOD 컬럼 미보유.

---

## 2. ERD (텍스트 다이어그램)

```
                ┌──────────────┐
                │   CATEGORY   │
                │  (코드성)    │
                │ category_cd  │ (PK)
                │ category_nm  │
                └───────┬──────┘
                        │ 1:N
                        ▼
┌──────────┐ 1:N  ┌─────────────┐ 1:N  ┌─────────────┐
│   USER   │─────▶│  INQUIRY    │─────▶│   ANSWER    │
│ user_id  │ (작성자│ inquiry_id  │      │ answer_id   │
│ login_id │ reg_id)│ title       │      │ inquiry_id  │ (FK)
│ password │      │ status      │      │ content     │
│ role     │      │ category_cd │ (FK) │ agent_id    │ (FK→USER)
│ ...      │      │ sla_due_dt  │      │ reg_dt      │
└──────────┘      │ version     │      └─────────────┘
      │           │ ...         │
      │ 1:N       └──────┬──────┘
      │ (상담사)         │ 1:N
      │ agent_id         ▼
      │            ┌─────────────┐
      └───────────▶│ ATTACHMENT  │
                   │ attachment  │
                   │ inquiry_id  │ (FK)
                   │ file_name   │
                   │ file_path   │
                   │ ...         │
                   └─────────────┘
```

### 관계 요약

| 관계 | 카디널리티 | 외래키 | ON DELETE |
|---|---|---|---|
| USER → INQUIRY (작성자) | 1:N | INQUIRY.REG_ID → USER.USER_ID | RESTRICT (사용자 삭제 시 문의 보존) |
| USER → ANSWER (상담사) | 1:N | ANSWER.AGENT_ID → USER.USER_ID | RESTRICT |
| CATEGORY → INQUIRY | 1:N | INQUIRY.CATEGORY_CD → CATEGORY.CATEGORY_CD | RESTRICT |
| INQUIRY → ANSWER | 1:N | ANSWER.INQUIRY_ID → INQUIRY.INQUIRY_ID | CASCADE (문의 삭제 시 답변도 삭제 — 단, USE_YN 논리삭제 우선) |
| INQUIRY → ATTACHMENT | 1:N | ATTACHMENT.INQUIRY_ID → INQUIRY.INQUIRY_ID | CASCADE |

> **논리삭제 우선 원칙**: 모든 업무 테이블은 USE_YN='N'으로 논리삭제. 물리삭제는 운영 감사 후 분기 1회만. ON DELETE CASCADE는 물리삭제 시점에만 발동.

---

## 3. 테이블 정의서

### 3.1 INQUIRY (문의)

| # | 컬럼명 | 타입·길이 | NULL | 기본값 | 키·인덱스 | 설명 |
|---|---|---|---|---|---|---|
| 1 | INQUIRY_ID | BIGINT | N | (Identity) | PK | 문의 ID |
| 2 | TITLE | VARCHAR(200) | N | — | — | 제목 (1~200자, FR-INQ-001) |
| 3 | CONTENT | TEXT | N | — | — | 내용 (1~5000자) |
| 4 | STATUS | VARCHAR(20) | N | 'NEW' | IX_STATUS_SLA | NEW/ANSWERED/REOPENED/CLOSED/CANCELLED |
| 5 | CATEGORY_CD | VARCHAR(20) | N | — | FK→CATEGORY, IX_CATEGORY | 카테고리 코드 |
| 6 | SLA_DUE_DT | TIMESTAMP | N | — | IX_STATUS_SLA | SLA 만료 시각 (REG_DT + 24h, REOPENED 시 재기산 X) |
| 7 | REG_ID | VARCHAR(50) | N | — | FK→USER (작성자) | 등록자 (GUEST) |
| 8 | REG_DT | TIMESTAMP | N | CURRENT_TIMESTAMP | — | 등록일시 |
| 9 | MOD_ID | VARCHAR(50) | Y | — | — | 최종 수정자 |
| 10 | MOD_DT | TIMESTAMP | Y | — | — | 최종 수정일시 |
| 11 | USE_YN | CHAR(1) | N | 'Y' | — | 논리삭제 |
| 12 | VERSION | BIGINT | N | 0 | — | 낙관적 락 |

**제약**:
- CHECK (`STATUS` IN ('NEW','ANSWERED','REOPENED','CLOSED','CANCELLED'))
- CHECK (`SLA_DUE_DT` > `REG_DT`)
- CHECK (LENGTH(`TITLE`) BETWEEN 1 AND 200)
- CHECK (LENGTH(`CONTENT`) BETWEEN 1 AND 5000)

**인덱스**:
| 인덱스명 | 컬럼 | 용도 |
|---|---|---|
| PK_INQUIRY | INQUIRY_ID | 기본키 |
| IX_INQUIRY_STATUS_SLA | (STATUS, SLA_DUE_DT) | FR-INQ-004 SLA 임박 알림 조회 (status='NEW' AND sla_due_dt 4h 이내) |
| IX_INQUIRY_CATEGORY | (CATEGORY_CD, REG_DT DESC) | FR-INQ-003 카테고리별 검색 |
| IX_INQUIRY_REG_ID | (REG_ID, REG_DT DESC) | GUEST 본인 문의 조회 |

### 3.2 ANSWER (답변)

| # | 컬럼명 | 타입·길이 | NULL | 기본값 | 키·인덱스 | 설명 |
|---|---|---|---|---|---|---|
| 1 | ANSWER_ID | BIGINT | N | (Identity) | PK | 답변 ID |
| 2 | INQUIRY_ID | BIGINT | N | — | FK→INQUIRY, IX_ANSWER_INQUIRY | 문의 ID |
| 3 | CONTENT | TEXT | N | — | — | 답변 내용 (1~5000자) |
| 4 | AGENT_ID | VARCHAR(50) | N | — | FK→USER (상담사) | 답변 작성자 |
| 5 | REG_DT | TIMESTAMP | N | CURRENT_TIMESTAMP | — | 등록일시 (= 답변 시각) |
| 6 | USE_YN | CHAR(1) | N | 'Y' | — | 논리삭제 |

**제약**:
- CHECK (LENGTH(`CONTENT`) BETWEEN 1 AND 5000)

**인덱스**:
| 인덱스명 | 컬럼 | 용도 |
|---|---|---|
| PK_ANSWER | ANSWER_ID | 기본키 |
| IX_ANSWER_INQUIRY | (INQUIRY_ID, REG_DT) | 문의 1건의 답변 이력 조회 (REOPENED 대응 1:N) |

> **MOD 컬럼 없음**: 답변은 등록 후 수정 불가 정책 (감사·신뢰성). 잘못 등록 시 USE_YN='N' + 새 답변 등록.

### 3.3 ATTACHMENT (첨부파일)

| # | 컬럼명 | 타입·길이 | NULL | 기본값 | 키·인덱스 | 설명 |
|---|---|---|---|---|---|---|
| 1 | ATTACHMENT_ID | BIGINT | N | (Identity) | PK | 첨부 ID |
| 2 | INQUIRY_ID | BIGINT | N | — | FK→INQUIRY, IX_ATTACHMENT_INQUIRY | 문의 ID |
| 3 | FILE_NAME | VARCHAR(255) | N | — | — | 원본 파일명 |
| 4 | FILE_PATH | VARCHAR(500) | N | — | — | 저장 경로 (사내 NAS 기준 상대) |
| 5 | FILE_SIZE | BIGINT | N | — | — | 파일 크기 (Byte, 최대 10MB = 10,485,760) |
| 6 | CONTENT_TYPE | VARCHAR(100) | N | — | — | MIME (image/png 등) |
| 7 | REG_ID | VARCHAR(50) | N | — | FK→USER | 등록자 |
| 8 | REG_DT | TIMESTAMP | N | CURRENT_TIMESTAMP | — | 등록일시 |

**제약**:
- CHECK (`FILE_SIZE` <= 10485760)
- (응용 레벨) 확장자 화이트리스트: png/jpg/jpeg/gif/pdf/doc/docx/xls/xlsx/zip

**인덱스**:
| 인덱스명 | 컬럼 | 용도 |
|---|---|---|
| PK_ATTACHMENT | ATTACHMENT_ID | 기본키 |
| IX_ATTACHMENT_INQUIRY | INQUIRY_ID | 문의의 첨부 일괄 조회 |

### 3.4 USER (사용자)

| # | 컬럼명 | 타입·길이 | NULL | 기본값 | 키·인덱스 | 설명 |
|---|---|---|---|---|---|---|
| 1 | USER_ID | VARCHAR(50) | N | — | PK | 사번 또는 UUID |
| 2 | LOGIN_ID | VARCHAR(50) | N | — | UK_USER_LOGIN_ID | 로그인 ID (사번) |
| 3 | PASSWORD | VARCHAR(100) | N | — | — | BCrypt 해시 (60자, 여유 100) |
| 4 | NAME | VARCHAR(100) | N | — | — | 이름 |
| 5 | ROLE | VARCHAR(20) | N | 'GUEST' | IX_USER_ROLE | GUEST/AGENT/ADMIN |
| 6 | DEPT | VARCHAR(100) | Y | — | — | 부서명 |
| 7 | EMAIL | VARCHAR(200) | N | — | — | 이메일 (마스킹 대상) |
| 8 | REG_DT | TIMESTAMP | N | CURRENT_TIMESTAMP | — | 등록일시 |
| 9 | MOD_DT | TIMESTAMP | Y | — | — | 최종 수정일시 |
| 10 | USE_YN | CHAR(1) | N | 'Y' | — | 활성 여부 (퇴사 시 'N') |
| 11 | VERSION | BIGINT | N | 0 | — | 낙관적 락 |

**제약**:
- CHECK (`ROLE` IN ('GUEST','AGENT','ADMIN'))
- CHECK (`EMAIL` LIKE '%@%')

**인덱스**:
| 인덱스명 | 컬럼 | 용도 |
|---|---|---|
| PK_USER | USER_ID | 기본키 |
| UK_USER_LOGIN_ID | LOGIN_ID | 로그인 시 조회 (유니크) |
| IX_USER_ROLE | (ROLE, USE_YN) | AGENT 전체 알림 조회 (FR-INQ-004) |

### 3.5 CATEGORY (카테고리 코드)

| # | 컬럼명 | 타입·길이 | NULL | 기본값 | 키·인덱스 | 설명 |
|---|---|---|---|---|---|---|
| 1 | CATEGORY_CD | VARCHAR(20) | N | — | PK | 카테고리 코드 |
| 2 | CATEGORY_NM | VARCHAR(100) | N | — | — | 카테고리 명칭 |
| 3 | SORT_ORD | INT | N | 0 | — | 정렬 순서 |
| 4 | USE_YN | CHAR(1) | N | 'Y' | — | 사용 여부 |
| 5 | REG_DT | TIMESTAMP | N | CURRENT_TIMESTAMP | — | 등록일시 |

> **공통코드 관리 방식**: DDL이 아닌 **INSERT 스크립트**로 관리. 운영 중 추가는 ADMIN 화면에서.

---

## 4. 공통코드 INSERT 스크립트 (초기 데이터)

```sql
-- CATEGORY 초기값
INSERT INTO CATEGORY (CATEGORY_CD, CATEGORY_NM, SORT_ORD, USE_YN, REG_DT) VALUES
  ('PRODUCT',  '제품문의', 1, 'Y', CURRENT_TIMESTAMP),
  ('DELIVERY', '배송문의', 2, 'Y', CURRENT_TIMESTAMP),
  ('REFUND',   '환불문의', 3, 'Y', CURRENT_TIMESTAMP),
  ('ETC',      '기타',    9, 'Y', CURRENT_TIMESTAMP);

-- 초기 ADMIN 1명 (실제 운영 시 비밀번호 별도 생성)
INSERT INTO USER (USER_ID, LOGIN_ID, PASSWORD, NAME, ROLE, DEPT, EMAIL, REG_DT, USE_YN, VERSION) VALUES
  ('admin', 'admin', '$2b$10$EXAMPLE_BCRYPT_HASH_REPLACE_AT_DEPLOY', '시스템관리자', 'ADMIN', '정보화팀', 'admin@example.com', CURRENT_TIMESTAMP, 'Y', 0);
```

---

## 5. 동시성 제어 정책

| 시나리오 | 전략 | 구현 |
|---|---|---|
| 동시 답변 작성 (FR-INQ-002) | 낙관적 락 | INQUIRY.VERSION 컬럼 — UPDATE 시 WHERE VERSION = ? AND VERSION = VERSION + 1. 충돌 시 CONFLICT(409) 응답 |
| 동시 문의 취소 (FR-INQ-006) | 낙관적 락 | 동일 (VERSION 검사) |
| SLA 알림 배치 중복 발송 방지 | 분산 락 | 배치 시작 시 `SELECT ... FOR UPDATE` 또는 분산 락 (Redis 권장). 동일 시각 다중 인스턴스 실행 시 1개만 처리 |

> 동일 사용자의 단순 등록·검색은 락 불필요 (Read 동시성 자연 처리).

---

## 6. 트랜잭션 경계

| 기능 | 단일 트랜잭션 범위 |
|---|---|
| FR-INQ-001 (문의 등록) | INQUIRY INSERT + ATTACHMENT INSERT (N건) |
| FR-INQ-002 (답변 작성) | ANSWER INSERT + INQUIRY UPDATE (status, MOD_DT, VERSION) — 이메일 발송은 트랜잭션 외부 비동기 |
| FR-INQ-006 (문의 취소) | INQUIRY UPDATE (status='CANCELLED', MOD_DT, VERSION) — 알림 큐 제거는 트랜잭션 외부 |
| FR-INQ-004 (SLA 알림 배치) | 1건당 별도 트랜잭션 (실패 격리) |

---

## 7. 데이터 보존·삭제 정책

| 데이터 | 보존 | 삭제 방식 |
|---|---|---|
| 문의 / 답변 | 5년 | 5년 경과 + USE_YN='N' → 분기 1회 물리삭제 |
| 첨부 파일 | 1년 | 1년 경과 → NAS 파일 삭제 + DB row USE_YN='N' |
| 운영 로그 | 3개월 | hot 90일, 이후 자동 삭제 |
| 감사 로그 | 1년 | 1년 보관 후 자동 삭제 |
| 사용자(USER) | 영구 | 퇴사 시 USE_YN='N', 영구 보관 (감사 추적용) |

---

## 8. Mock-Impl 롤플레이 결과 (Green 판정)

리뷰어가 개발자 페르소나로 1테이블·1API·1화면 의사코드 작성 시도 → 질문 0개 → **Green**.

| 점검 | 결과 |
|---|---|
| Enum 전수 (STATUS, ROLE) | ✅ |
| Validation 전수 (길이·범위·NULL·기본값) | ✅ |
| 인덱스 (단일·복합 선행컬럼 결정) | ✅ |
| 공통컬럼 (REG/MOD/USE_YN/VERSION) | ✅ |
| 동시성·트랜잭션 경계 | ✅ |
| 권한 매트릭스 (별도 권한매트릭스.md 참조) | ✅ |

---

## 9. 후속 단계 영향

- **07 개발**: 본 설계서 기반 마이그레이션 스크립트 작성 (`V1__create_helpdesk_lite.sql`)
- **08 테스트**: 인덱스 효율성 점검 — IX_INQUIRY_STATUS_SLA가 실제 SLA 알림 쿼리 plan에 사용되는지 EXPLAIN으로 검증
- **09 인수인계**: 본 설계서 v1.0이 산출물 목록에 등재 + 운영매뉴얼에 발췌

---

## 10. 승인

| 구분 | 소속·직위 | 성명 | 서명·날인 | 일자 |
|---|---|---|---|---|
| 발주사 작성 검토 | 정보화팀 | 최OO | | 2026-06-10 |
| 발주사 승인 | 정보화팀장 | 정OO | | 2026-06-10 |
| 수행사 작성 | DBA | 박OO | | 2026-06-10 |
| 수행사 검토 | 아키텍트 | 한OO | | 2026-06-10 |
| 수행사 검토 | PM | 이OO | | 2026-06-10 |

---

*본 설계서 v1.0은 SSOT 폴더에 영구 지정된다 (경로 변경 금지). 변경은 변경관리 절차로만 가능.*
