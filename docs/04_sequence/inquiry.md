# 문의(Inquiry) 시퀀스

## 개요

시스템 관리자(`users.role = ADMIN`)를 제외한 모든 사용자가 문의를 남길 수 있는
기능입니다. 문의는 두 유형으로 나뉩니다.

- **일반 문의(GENERAL)**: 재배와 무관한 문의입니다. 관리자가 텍스트로 답변을
  남깁니다.
- **경작 문의(CULTIVATION)**: 특정 재배(`cultivationId`)에 대한 문의입니다. 예를
  들어 재배에 오류가 발생해 삭제하고 다시 만들어야 하는 경우에 사용합니다. 관리자는
  텍스트로 답변하지 않고, 문의를 읽은 뒤 그 재배를 삭제하는 것으로만 처리합니다.

두 유형 모두 작성은 Cultivation Service의 `POST /inquiries`로 동일하게 처리되며,
관리자가 처리하는 방식(답변 vs 삭제)만 다릅니다.

---

# Sequence — ① 문의 등록

```text
Client (관리자가 아닌 사용자)
↓
API Gateway (JWT role 확인 — ADMIN이면 거부)
↓
Cultivation Service
↓
inquiry 저장 (status = OPEN)
↓
Client (등록 완료 응답)
```

---

# Sequence — ② 일반 문의 답변 (관리자)

```text
관리자 Client
↓
API Gateway (JWT role 확인 — ADMIN만 허용)
↓
Cultivation Service
↓
inquiry 조회 (type = GENERAL)
↓
answer 저장 + status: OPEN → ANSWERED
↓
관리자 Client (완료 응답)

(작성자에게 별도 알림은 발송하지 않음 — 사용자가 목록에서 직접 확인)
```

---

# Sequence — ③ 경작 문의 처리 (관리자, 재배 삭제)

```text
관리자 Client
↓
API Gateway (JWT role 확인 — ADMIN만 허용)
↓
Cultivation Service
↓
inquiry 조회 (type = CULTIVATION, cultivationId 확인)
↓
DELETE /admin/cultivations/{cultivationId}
↓
같은 트랜잭션에서 harvest/photo/sensor/environment_setting CASCADE 삭제
↓
CultivationDeletedEvent 발행
↓
inquiry.status: OPEN → CLOSED (cultivation_id 값은 그대로 보존)
↓
관리자 Client (완료 응답)
```

---

# 상세 과정

## 1. 문의 등록 요청

```http
POST /api/v1/inquiries
```

```json
{
    "type": "CULTIVATION",
    "cultivationId": 12,
    "title": "재배가 이상하게 생성됐어요",
    "content": "환경 설정이 저장되지 않고 계속 CREATED 상태로 남아있습니다. 삭제하고 다시 만들고 싶습니다."
}
```

`type`이 `GENERAL`이면 `cultivationId`는 보내지 않습니다(보내도 무시하지 않고
검증 실패로 처리합니다 — 유형과 맞지 않는 필드 조합).

---

## 2. 작성 권한 확인

API Gateway 또는 Cultivation Service가 JWT의 `role` 클레임을 확인합니다.
`role = ADMIN`이면 문의 작성을 거부합니다(관리자는 문의를 남기는 주체가 아니라
처리하는 주체이기 때문).

---

## 3. inquiry 저장

```sql
INSERT INTO inquiry (user_id, type, cultivation_id, title, content, status)
VALUES (7, 'CULTIVATION', 12, '재배가 이상하게 생성됐어요', '...', 'OPEN');
```

---

## 4. 관리자 문의 목록 조회 (별도 흐름)

```http
GET /api/v1/admin/inquiries?status=OPEN
```

관리자는 미처리(`OPEN`) 문의를 유형과 무관하게 함께 조회할 수 있습니다. 유형에 따라
프론트엔드가 "답변하기" 또는 "재배 삭제" 중 알맞은 처리 화면을 보여줍니다.

---

## 5-A. 일반 문의 답변

```http
PATCH /api/v1/admin/inquiries/{id}/answer
```

```json
{ "answer": "문의 주신 내용은 다음 배포에서 반영될 예정입니다. 이용해 주셔서 감사합니다." }
```

```sql
UPDATE inquiry
SET answer = '...', status = 'ANSWERED', admin_id = 1, answered_at = NOW()
WHERE id = ? AND type = 'GENERAL';
```

`type`이 `CULTIVATION`인 문의에 이 API를 호출하면 에러를 반환합니다.

---

## 5-B. 경작 문의 처리 (재배 삭제)

```http
DELETE /api/v1/admin/cultivations/{cultivationId}
```

Cultivation Service는 해당 재배와 연관된 `harvest`/`photo`/`sensor`/
`environment_setting`을 같은 트랜잭션에서 함께 삭제합니다(같은 DB 안의 실제 FK에
`ON DELETE CASCADE`가 걸려 있어 애플리케이션이 각 테이블을 따로 지울 필요가
없습니다). 삭제 후 `CultivationDeletedEvent`를 발행합니다.

이어서 해당 `cultivationId`를 가리키던 `CULTIVATION` 유형 문의(들)의 상태를 갱신합니다.

```sql
UPDATE inquiry
SET status = 'CLOSED', admin_id = 1, answered_at = NOW()
WHERE cultivation_id = ? AND type = 'CULTIVATION' AND status = 'OPEN';
```

`inquiry.cultivation_id`는 FK가 아니므로 재배가 삭제되어도 이 값은 그대로 남아
"어떤 재배에 대한 문의였는지" 기록을 보존합니다.

---

## 6. 사용자 본인 문의 조회 (별도 흐름)

```http
GET /api/v1/inquiries?userId=me
```

작성자 본인은 자신이 남긴 문의 목록과 상태(OPEN/ANSWERED/CLOSED), 일반 문의의 경우
`answer`를 조회할 수 있습니다. 경작 문의가 `CLOSED`로 바뀐 것을 보고 재배가
삭제되었음을 확인할 수 있습니다(재배 목록에서도 사라짐).

---

# 사용 Database

## PostgreSQL

```
inquiry (Cultivation DB, 쓰기 + 조회)
cultivation / harvest / photo / sensor / environment_setting (Cultivation DB, 경작 문의 처리 시 삭제 대상)
```

---

# RabbitMQ

Publish

```
CultivationDeletedEvent (경작 문의 처리로 재배가 삭제될 때)
```

현재 이 이벤트를 구독하는 서비스는 없습니다.

---

# 예외 상황

- 관리자가 아닌 사용자의 문의 작성은 정상 동작이며, 관리자 본인의 문의 작성만 거부
- 존재하지 않는 문의/재배에 대한 답변·삭제 시도
- GENERAL 문의에 삭제를 시도하거나 CULTIVATION 문의에 답변을 시도하는 등 유형에 맞지
  않는 처리 시도
- 이미 `ANSWERED`/`CLOSED`된 문의에 대한 재처리 시도
- 다른 사용자의 문의 조회 시도 (본인 문의만 조회 가능, 관리자는 예외)

---

# 고려 사항

- 문의 작성 가능 여부는 JWT `role` 클레임으로만 판단합니다. Cultivation DB에는
  `role` 정보가 없으므로 Auth Service가 발급한 토큰을 신뢰합니다.
- 일반 문의 답변은 사용자에게 알림을 보내지 않습니다. 사용자가 직접 문의 목록을
  조회해 확인해야 합니다.
- 경작 문의는 재배 삭제 없이 종료 처리하는 기능이 아직 없습니다 — 관리자가 조치가
  필요 없다고 판단해도 문의는 `OPEN`으로 남습니다(추후 개발 예정).
- 재배 삭제는 하드 삭제이며 되돌릴 수 없습니다. `harvest`/`photo`/`sensor`/
  `environment_setting`이 함께 사라지지만, 다른 서비스(AI DB의 `growth_record`/
  `daily_feedback`/`insight`, Notification DB의 `notification_endpoint`)에 남아있는
  이 재배 관련 데이터는 자동으로 정리되지 않습니다(소프트 참조라 별도 이벤트 구독이
  필요, 현재는 미구현).
