# 문의(Inquiry) 시퀀스

## 개요

시스템 관리자(`users.role = ADMIN`)를 제외한 모든 사용자가 문의를 남길 수 있는
기능입니다. 문의는 `inquiry_category` 참조 테이블로 관리되는 카테고리로 나뉩니다
(시드 데이터: `id=1` "일반 문의", `id=2` "경작 문의").

- **일반 문의(category_id=1)**: 재배와 무관한 문의입니다. 관리자가 텍스트로 답변을
  남깁니다.
- **경작 문의(category_id=2)**: 특정 재배(`cultivationId`)에 대한 문의입니다. 예를
  들어 재배에 오류가 발생해 삭제하고 다시 만들어야 하는 경우에 사용합니다. 관리자는
  텍스트로 답변하지 않고, 문의를 읽은 뒤 그 재배를 삭제하는 것으로만 처리합니다.

두 카테고리 모두 작성은 Cultivation Service의 `POST /inquiries`로 동일하게 처리되며,
관리자가 처리하는 방식(답변 vs 삭제)만 다릅니다.

기존에는 문의 한 건을 `inquiry` 테이블 한 행(단일 `answer` 컬럼)으로 표현했지만,
외부 ERD 기준으로 `inquiry`(메타데이터 + 상태)와 `inquiry_answer`(문의 본문/답변을
시간순으로 쌓는 스레드) 두 테이블로 분리했습니다. `inquiry.status`도 기존 3단계
(OPEN/ANSWERED/CLOSED)에서 `PENDING`/`RESOLVED` 2단계로 축소했습니다 — 실제 처리
내용(답변 유무, 재배 삭제 여부)은 `inquiry_answer` 행 유무나 재배 존재 여부로 구분할
수 있기 때문입니다.

---

# Sequence — ① 문의 등록

```text
Client (관리자가 아닌 사용자)
↓
API Gateway (JWT role 확인 — ADMIN이면 거부)
↓
Cultivation Service
↓
inquiry 저장 (status = PENDING)
↓
inquiry_answer 저장 (content = 문의 내용, answer_content = NULL, pre_id = NULL)
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
inquiry 조회 (category_id = 1, "일반 문의")
↓
최초 inquiry_answer 행(pre_id IS NULL) 조회
↓
inquiry_answer 저장 (content = NULL, answer_content = 답변 내용, pre_id = 최초 행 id)
↓
inquiry.status: PENDING → RESOLVED
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
inquiry 조회 (category_id = 2, "경작 문의", cultivationId 확인)
↓
DELETE /admin/cultivations/{cultivationId}
↓
같은 트랜잭션에서 harvest/cultivation_photo/cultivation_sensor/environment_setting CASCADE 삭제
↓
CultivationDeletedEvent 발행
↓
inquiry.status: PENDING → RESOLVED (cultivation_id 값은 그대로 보존, inquiry_answer 행은 생성하지 않음)
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
    "categoryId": 2,
    "cultivationId": 12,
    "title": "재배가 이상하게 생성됐어요",
    "content": "환경 설정이 저장되지 않고 계속 CREATED 상태로 남아있습니다. 삭제하고 다시 만들고 싶습니다."
}
```

`categoryId`가 1("일반 문의")이면 `cultivationId`는 보내지 않습니다(보내도 무시하지
않고 검증 실패로 처리합니다 — 카테고리와 맞지 않는 필드 조합). `content`는
`inquiry`에는 저장되지 않고, 같이 생성되는 `inquiry_answer`의 최초 행에 담깁니다.

---

## 2. 작성 권한 확인

API Gateway 또는 Cultivation Service가 JWT의 `role` 클레임을 확인합니다.
`role = ADMIN`이면 문의 작성을 거부합니다(관리자는 문의를 남기는 주체가 아니라
처리하는 주체이기 때문).

---

## 3. inquiry + inquiry_answer 저장

```sql
INSERT INTO inquiry (user_id, category_id, cultivation_id, title, status)
VALUES (7, 2, 12, '재배가 이상하게 생성됐어요', 'PENDING')
RETURNING id;

INSERT INTO inquiry_answer (inquiry_id, content, answer_content, pre_id)
VALUES (?, '환경 설정이 저장되지 않고 계속 CREATED 상태로 남아있습니다. 삭제하고 다시 만들고 싶습니다.', NULL, NULL);
```

두 INSERT는 같은 트랜잭션으로 처리합니다.

---

## 4. 관리자 문의 목록 조회 (별도 흐름)

```http
GET /api/v1/admin/inquiries?status=PENDING
```

관리자는 미처리(`PENDING`) 문의를 카테고리와 무관하게 함께 조회할 수 있습니다.
`category_id`에 따라 프론트엔드가 "답변하기" 또는 "재배 삭제" 중 알맞은 처리 화면을
보여줍니다.

---

## 5-A. 일반 문의 답변

```http
PATCH /api/v1/admin/inquiries/{id}/answer
```

```json
{ "answerContent": "문의 주신 내용은 다음 배포에서 반영될 예정입니다. 이용해 주셔서 감사합니다." }
```

```sql
INSERT INTO inquiry_answer (inquiry_id, content, answer_content, pre_id)
VALUES (?, NULL, '문의 주신 내용은 다음 배포에서 반영될 예정입니다. 이용해 주셔서 감사합니다.',
        (SELECT id FROM inquiry_answer WHERE inquiry_id = ? AND pre_id IS NULL));

UPDATE inquiry SET status = 'RESOLVED' WHERE id = ? AND category_id = 1;
```

`category_id`가 2("경작 문의")인 문의에 이 API를 호출하면 에러를 반환합니다.
사용자가 답변을 보고 추가 질문을 남기는 스레드 확장(`pre_id`를 답변 행으로 지정해
새 `inquiry_answer`를 계속 이어가는 것)은 스키마상 가능하지만, 현재 범위에서는
"관리자 답변 1회"로 단순하게 다룹니다(추후 확장 여지로만 남겨둡니다).

---

## 5-B. 경작 문의 처리 (재배 삭제)

```http
DELETE /api/v1/admin/cultivations/{cultivationId}
```

Cultivation Service는 해당 재배와 연관된 `harvest`/`cultivation_photo`/
`cultivation_sensor`/`environment_setting`을 같은 트랜잭션에서 함께 삭제합니다(같은 DB
안의 실제 FK에 `ON DELETE CASCADE`가 걸려 있어 애플리케이션이 각 테이블을 따로 지울
필요가 없습니다). 삭제 후 `CultivationDeletedEvent`를 발행합니다.

이어서 해당 `cultivationId`를 가리키던 경작 문의(들)의 상태를 갱신합니다. 텍스트
답변이 없으므로 `inquiry_answer` 행은 추가하지 않습니다.

```sql
UPDATE inquiry
SET status = 'RESOLVED'
WHERE cultivation_id = ? AND category_id = 2 AND status = 'PENDING';
```

`inquiry.cultivation_id`는 FK가 아니므로 재배가 삭제되어도 이 값은 그대로 남아
"어떤 재배에 대한 문의였는지" 기록을 보존합니다.

---

## 6. 사용자 본인 문의 조회 (별도 흐름)

```http
GET /api/v1/inquiries?userId=me
```

```http
GET /api/v1/inquiries/{id}
```

작성자 본인은 자신이 남긴 문의 목록과 상태(PENDING/RESOLVED)를 조회할 수 있고,
단건 조회 시 `inquiry_answer` 스레드(문의 본문 → 답변)를 시간순으로 함께 받습니다.
경작 문의가 `RESOLVED`로 바뀐 것을 보고 재배가 삭제되었음을 확인할 수 있습니다
(재배 목록에서도 사라짐).

```json
{
    "inquiryId": 41,
    "categoryId": 1,
    "categoryName": "일반 문의",
    "title": "정기 점검 일정이 궁금합니다",
    "status": "RESOLVED",
    "thread": [
        { "id": 101, "content": "정기 점검 일정이 궁금합니다.", "answerContent": null, "preId": null, "createdAt": "2026-07-20T09:00:00" },
        { "id": 108, "content": null, "answerContent": "매주 월요일 오전에 점검합니다.", "preId": 101, "createdAt": "2026-07-21T10:00:00" }
    ]
}
```

---

# 사용 Database

## PostgreSQL

```
inquiry_category (Cultivation DB, 조회 전용 시드 데이터)
inquiry (Cultivation DB, 쓰기 + 조회)
inquiry_answer (Cultivation DB, 쓰기 + 조회)
cultivation / harvest / cultivation_photo / cultivation_sensor / environment_setting (Cultivation DB, 경작 문의 처리 시 삭제 대상)
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
- 일반 문의(category_id=1)에 재배 삭제를 시도하거나 경작 문의(category_id=2)에
  텍스트 답변을 시도하는 등 카테고리에 맞지 않는 처리 시도
- 경작 문의인데 `cultivationId`가 없거나, 일반 문의인데 `cultivationId`를 보낸 등
  카테고리-필드 불일치
- 이미 `RESOLVED`된 문의에 대한 재처리 시도
- 다른 사용자의 문의 조회 시도 (본인 문의만 조회 가능, 관리자는 예외)

---

# 고려 사항

- 문의 작성 가능 여부는 JWT `role` 클레임으로만 판단합니다. Cultivation DB에는
  `role` 정보가 없으므로 Auth Service가 발급한 토큰을 신뢰합니다.
- 일반 문의 답변은 사용자에게 알림을 보내지 않습니다. 사용자가 직접 문의 목록을
  조회해 확인해야 합니다.
- 경작 문의는 재배 삭제 없이 종료 처리하는 기능이 아직 없습니다 — 관리자가 조치가
  필요 없다고 판단해도 문의는 `PENDING`으로 남습니다(추후 개발 예정).
- 재배 삭제는 하드 삭제이며 되돌릴 수 없습니다. `harvest`/`cultivation_photo`/
  `cultivation_sensor`/`environment_setting`이 함께 사라지지만, 다른 서비스(AI DB의
  `growth_record`/`daily_feedback`/`insight`, Notification DB의
  `notification_subscription.target_id`가 이 재배를 가리키는 구독 행)에 남아있는 이
  재배 관련 데이터는 자동으로 정리되지 않습니다(소프트 참조라 별도 이벤트 구독이
  필요, 현재는 미구현). `notification_endpoint`는 사용자 단위라 재배 삭제와 무관하게
  그대로 남습니다 — 정리 대상은 그 재배를 가리키던 `notification_subscription` 행뿐입니다.
- `inquiry.category_id`가 "경작 문의"(`id=2`)일 때만 `cultivation_id`가 채워지는
  것은 애플리케이션 레벨 불변식이며, DB CHECK로는 강제하지 않습니다 — `category_id`가
  FK라 특정 값(2)을 하드코딩한 CHECK 제약은 카테고리가 데이터로 관리되는 설계 의도와
  맞지 않다고 판단했습니다.
- `inquiry`에는 더 이상 `admin_id`/`answered_at` 컬럼이 없습니다. "언제"는
  `inquiry_answer.created_at`으로 확인할 수 있고, "누가 답변했는지"는 현재 스키마에
  없습니다 — 필요해지면 `inquiry_answer.admin_id` 추가를 추후 검토할 수 있습니다.
- `inquiry_answer`의 `pre_id` 스레드 체인은 "문의 → 답변 → 추가 질문 → 재답변"까지
  확장 가능하도록 설계되어 있지만, 현재 기능 범위는 기존과 동일하게 "관리자 답변
  1회"만 다룹니다. 과도하게 앞서 구현하지 않습니다.
