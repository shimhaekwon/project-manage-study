---
산출물명: 인터페이스 정의서 (IF Definition)
프로젝트: 사내 고객 Q&A 시스템 (Helpdesk Lite)
단계: 06 설계
작성: PL 박OO
검토: 아키텍트 한OO / PM 이OO / (발주사) 인프라팀
승인: (발주사) 정보화팀장 정OO
버전: v1.0
작성일: 2026-06-17
---

# 인터페이스 정의서 — Helpdesk Lite

## 변경이력

| 버전 | 일자 | 작성자 | 변경 내용 |
|---|---|---|---|
| v1.0 | 2026-06-17 | 박OO | 외부 SMTP 1건 명세 (발주사 인프라팀 협의 반영) |

---

## 1. 인터페이스 목록

| IF ID | 연계 시스템 | 방식 | 방향 | 용도 | 관련 요구 |
|---|---|---|---|---|---|
| IF-INQ-001 | 사내 SMTP 서버 | SMTP (포트 465) | Helpdesk → SMTP | 알림 메일 발송 | IR-INQ-001 / FR-INQ-002, FR-INQ-004 |

> Phase 1은 외부 연계 1건만. Phase 2에서 사내 인사 시스템·SMS·챗봇 연계 검토.

---

## 2. IF-INQ-001 — 사내 SMTP 메일 발송

### 2.1 기본 정보

| 항목 | 값 |
|---|---|
| IF ID | IF-INQ-001 |
| 연계 시스템 | 사내 SMTP 서버 |
| 방향 | Helpdesk Lite (Sender) → SMTP (Server) |
| 방식 | SMTP over TLS (SMTPS, 포트 465) |
| 동기·비동기 | **비동기** (트랜잭션 외부, 큐 기반) |
| 호출 모듈 | inquiry-sla-notifier (배치) + inquiry-core (답변 등록 시) |

### 2.2 연결 정보 (운영 환경)

| 항목 | 값 (예시) | 비고 |
|---|---|---|
| Host | smtp.a-corp.local | 사내 도메인 |
| Port | 465 | SMTPS |
| 인증 방식 | SMTP AUTH (PLAIN over TLS) |
| 사용자명 | helpdesk-lite@a-corp.local | 발송 전용 계정 |
| 비밀번호 | (Vault 저장, 환경변수 주입) | 운영팀 발급 |
| 발신자 (From) | helpdesk-lite@a-corp.local | 고정 |
| Reply-To | (선택) | 답변 메일은 helpdesk@a-corp.local 등으로 회신 안내 |

### 2.3 메시지 포맷

#### 2.3.1 SLA 임박 알림 (FR-INQ-004)

| 항목 | 내용 |
|---|---|
| 제목 | `[Helpdesk] [SLA 임박] {inquiry.title}` (제목 길이 제한 100자, 초과 시 `...`) |
| 본문 | HTML + 텍스트 (Multipart) |
| 본문 주요 항목 | (1) 인사말 (2) 문의 ID·제목·카테고리·등록일·SLA 만료시각 (3) 잔여시간 (4) 상세 화면 링크 (5) 푸터 (자동 발송 안내·문의 채널) |
| 첨부 | 없음 |
| 우선순위 | Normal |
| 수신자 | 전체 AGENT (또는 ADMIN 설정 그룹). To 또는 Bcc로 분리 발송 (개인정보 노출 방지 시 Bcc) |

#### 2.3.2 답변 완료 알림 (FR-INQ-002)

| 항목 | 내용 |
|---|---|
| 제목 | `[Helpdesk] [답변 등록] {inquiry.title}` |
| 본문 | (1) 인사말 (2) 문의 ID·제목 (3) 답변 작성자 (마스킹) (4) 답변 미리보기 (100자) (5) 상세 화면 링크 (6) 푸터 |
| 수신자 | INQUIRY.REG_ID (등록자 GUEST의 이메일) |

#### 2.3.3 본문 템플릿 (HTML 예시 — SLA 임박)

```html
<!DOCTYPE html>
<html>
<body style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
  <h2>[Helpdesk Lite] SLA 임박 알림</h2>
  <p>안녕하세요. 다음 문의의 SLA 마감이 임박하여 알림드립니다.</p>
  <table border="0" cellpadding="8" style="border-collapse: collapse;">
    <tr><td><b>문의 ID</b></td><td>{inquiry.id}</td></tr>
    <tr><td><b>제목</b></td><td>{inquiry.title}</td></tr>
    <tr><td><b>카테고리</b></td><td>{inquiry.categoryNm}</td></tr>
    <tr><td><b>등록일</b></td><td>{inquiry.regDt}</td></tr>
    <tr><td><b>SLA 만료</b></td><td>{inquiry.slaDueDt} (잔여 약 4시간)</td></tr>
  </table>
  <p><a href="{baseUrl}/inquiries/{inquiry.id}">상세 화면 바로가기</a></p>
  <hr>
  <p style="color: #888; font-size: 12px;">
    본 메일은 자동 발송됩니다. 문의 사항은 정보화팀에 회신 부탁드립니다.<br>
    Helpdesk Lite 시스템
  </p>
</body>
</html>
```

---

## 3. 발송 정책

### 3.1 발송 절차

```
[트리거] (배치 1시간 주기 또는 답변 등록 트랜잭션 후)
       ↓
[발송 큐 적재] (INQUIRY 식별 + 메시지 종류)
       ↓
[Worker 1~2 인스턴스] — 큐 소비
       ↓
[SMTP 연결·인증·전송]
       ↓ (성공)
[감사 로그 + 발송 이력 적재]
       ↓
[큐에서 제거]
```

### 3.2 실패 처리

| 실패 종류 | 대응 |
|---|---|
| SMTP 연결 실패 (네트워크) | 3회 재시도 (지수 백오프 30s/2m/10m), 실패 시 운영 알림 |
| SMTP 인증 실패 | 즉시 운영 알림 (시크릿 갱신 필요 가능성) |
| 수신자 주소 오류 (550 Bounce) | 발송 이력에 BOUNCED 표시, 사용자 마스터 alert |
| Rate Limit (SMTP 측) | 30초 대기 후 재시도 |

### 3.3 중복 발송 방지

- SLA 알림: `notification_log` 테이블에 (inquiry_id, type='SLA_4H_BEFORE')의 UNIQUE 제약 — 1회만 발송
- 답변 알림: 답변 1건당 1회

### 3.4 발송 이력 (notification_log 테이블)

| 컬럼 | 타입 | 설명 |
|---|---|---|
| log_id | BIGINT | PK |
| inquiry_id | BIGINT | FK |
| notification_type | VARCHAR(20) | SLA_4H_BEFORE, ANSWER_COMPLETED |
| recipient | VARCHAR(255) | 수신자 이메일 (마스킹 X — 운영 추적용) |
| status | VARCHAR(20) | QUEUED, SENT, FAILED, BOUNCED |
| sent_dt | TIMESTAMP | 발송 시각 |
| error_msg | TEXT | 실패 시 사유 |

> 본 테이블은 DB설계서에는 누락. **변경관리 CR-2026-002로 추가 예정** (별도 등재).

---

## 4. 보안 (NFR-INQ-003 일관성)

| 항목 | 정책 |
|---|---|
| 전송 암호화 | SMTPS (TLS) |
| 인증 정보 | Vault 저장, 코드·Git 노출 금지 |
| 수신자 마스킹 | 메일 본문에는 마스킹 적용 (등록자 이름 등) |
| 본문 첨부 | 본문에 첨부 파일 직접 X — 상세 화면 링크만 |
| 외부 도메인 발송 | 사내망 외 도메인은 미발송 (Phase 1) |
| 감사 로그 | 발송 1건당 적재 (1년 보존) |

---

## 5. 운영성

| 항목 | 정책 |
|---|---|
| 모니터링 | 발송 큐 적재량 / 실패율 / 평균 처리 시간 (Grafana) |
| 알림 | 실패율 5% 초과 시 운영 알림 |
| 큐 백압 | 큐 5,000건 초과 시 알림 (성수기 대비) |
| 로그 | App 로그 + ELK + 발송 이력 테이블 (3중) |

---

## 6. 환경별 설정

| 환경 | SMTP Host | 비고 |
|---|---|---|
| 개발 | MailHog 또는 Mailtrap | 외부 발송 X (테스트 도구) |
| 테스트 (스테이징) | smtp-test.a-corp.local | 스테이징 전용 발송 계정 |
| 운영 | smtp.a-corp.local | 운영 발송 계정 |

---

## 7. 시험 계획

| 시험 | 시나리오 | 합격 기준 |
|---|---|---|
| 정상 발송 (단위) | SLA 임박 문의 1건 → SMTP 정상 응답 | 발송 이력 SENT, 메일 수신 확인 |
| 재시도 | SMTP 일시 다운 → 3회 재시도 | 3회 후 실패 시 알림 발송 |
| 중복 방지 | 동일 inquiry SLA 알림 2회 호출 | 1회만 발송, UNIQUE 제약 발동 |
| Bounce | 잘못된 주소 → 550 응답 | BOUNCED 표시 + 알림 |
| 부하 | 1만 건 SLA 임박 일괄 발송 | 30분 내 처리 (배치 분산) |
| 보안 | 본문에 개인정보 원본 포함 시도 | 마스킹 적용 확인 |

---

## 8. Mock-Impl 롤플레이 결과

리뷰어가 개발자 페르소나로 "SMTP 일시 다운 시 재시도 + 중복 방지" 시나리오 의사코드 → 질문 0개 → **Green**.

| 점검 | 결과 |
|---|---|
| 연결 정보·인증·메시지 포맷 전수 | ✅ |
| 실패·재시도·중복 방지 정책 | ✅ |
| 발송 이력 테이블 (CR-2026-002 등재 예정) | ✅ |
| 보안·운영 정책 | ✅ |

---

## 9. 발주사 인프라팀 협의 사항

| # | 항목 | 결정 |
|---|---|---|
| 1 | SMTP Host·Port·계정 발급 | 운영 계정 helpdesk-lite@a-corp.local 발급 (2026-06-15) |
| 2 | TLS 인증서 | 사내 인증서 사용 (사내 CA 신뢰) |
| 3 | 발송 한도 | 시간당 1만 건 (사내 SMTP 한도 내, 충분) |
| 4 | 외부 도메인 발송 | Phase 1 미사용. Phase 2 검토 시 별도 협의 |

---

## 10. 승인

| 구분 | 소속·직위 | 성명 | 서명·날인 | 일자 |
|---|---|---|---|---|
| 발주사 인프라팀 검토 | 인프라팀 | (대리) | | 2026-06-17 |
| 발주사 정보화 검토 | 정보화 PM | 최OO | | 2026-06-17 |
| 발주사 승인 | 정보화팀장 | 정OO | | 2026-06-17 |
| 수행사 작성 | PL | 박OO | | 2026-06-17 |
| 수행사 검토 | 아키텍트 | 한OO | | 2026-06-17 |
| 수행사 검토 | PM | 이OO | | 2026-06-17 |

---

*본 IF정의서는 외부 시스템 인터페이스의 SSOT다. 변경은 변경관리 절차로만.*
