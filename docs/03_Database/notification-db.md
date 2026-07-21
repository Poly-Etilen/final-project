# Notification Database

## 개요

Notification Database는 사용자에게 발송된 알림 이력을 `notification` 테이블 하나로 관리합니다.

기존에는 Notification Service가 별도 Database 없이 RabbitMQ 이벤트를 받아 채널(WebSocket/
Telegram/Discord)로 전송만 하고 기록을 남기지 않았습니다. 이 때문에 사용자가 지난 알림을
다시 확인할 방법이 없었습니다.

> ℹ️ **변경 이력**: 알림 이력 조회 기능이 추가되면서 Notification Service가 처음으로
> PostgreSQL DB를 갖게 되었습니다. 이 프로젝트는 서비스마다 자기 DB만 소유하는 원칙을
> 따르므로, 기존 Auth DB/Cultivation DB에 얹지 않고 Notification 전용 DB로 신설했습니다.
> (자세한 내용은 [notification.md](../01_Domain/notification.md),
> [notification-api.md](../02_API/notification-api.md) 참고)

> ℹ️ **변경 이력**: `type` CHECK 제약에서 `MONTHLY_REPORT`를 제거하고 `DAILY_FEEDBACK`을
> 추가했습니다. 재배 기간이 한 달을 넘지 않아 월간 리포트를 폐기했고, 대신 AI Service의
> Daily Scheduler가 매일 발행하는 `DailyFeedbackCompletedEvent`를 새로 구독하게 되었습니다.
> (자세한 내용은 [ai.md](../01_Domain/ai.md), [daily-feedback.md](../04_sequence/daily-feedback.md) 참고)

---

# ERD

```
notification
──────────────────────────────────────────────
PK  id
    user_id
    type
    message
    is_read
    created_at
```

---

# Table

## notification

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | X | 알림을 받는 사용자 (Auth Service의 userId, 소프트 참조) |
| type | VARCHAR(30) | X | 알림 종류 |
| message | VARCHAR(500) | X | 알림 문구 (채널로 발송한 것과 동일한 최종 문구) |
| is_read | BOOLEAN | X | 읽음 여부 |
| created_at | TIMESTAMP | X | 발송 시각 |

`user_id`는 Auth Service의 `users.id`를 가리키지만, 이 프로젝트의 다른 서비스 간 참조와
마찬가지로 DB 레벨 FK가 아닌 소프트 참조입니다. Notification Service가 구독하는 각 이벤트
payload에 `userId`가 포함되어 있다고 가정합니다(이벤트 발행 측 스키마와 맞춰야 합니다).

---

# DDL

```sql
CREATE TABLE notification (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT NOT NULL,

    type VARCHAR(30) NOT NULL,

    message VARCHAR(500) NOT NULL,

    is_read BOOLEAN NOT NULL DEFAULT FALSE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_notification_type CHECK (type IN (
        'ENVIRONMENT_CONTROL',
        'SENSOR_ERROR',
        'HARVEST_COMPLETED',
        'CULTIVATION_FINISHED',
        'WEEKLY_REPORT',
        'DAILY_FEEDBACK'
    ))
);
```

`type`은 Notification Service가 구독하는 6개 RabbitMQ 이벤트와 1:1로 대응합니다
(자세한 내용은 [notification.md](../01_Domain/notification.md)의 RabbitMQ 섹션 참고).

---

# Index

```sql
CREATE INDEX idx_notification_user
ON notification(user_id, created_at DESC);
```

알림 목록 조회(`GET /notifications`)가 항상 "특정 사용자의, 최신순" 조건으로 조회되기 때문에
이 복합 인덱스 하나로 충분합니다.

---

# 관계

```
notification (Notification Service)

    │ user_id

    ▼

Auth Service (userId로 알림 수신자 식별)
```

Notification Service는 다른 서비스와 데이터베이스를 공유하지 않으며, `userId`를 통해서만
간접적으로 참조됩니다.

---

# 다른 서비스와의 연관

## Rule Engine / Sensor / Cultivation / AI Service

직접 DB를 공유하지 않습니다. 각 서비스가 발행한 RabbitMQ 이벤트를 구독해 `notification`
행을 생성합니다.

---

# 고려 사항

- 알림 "발송"(WebSocket/Telegram/Discord)과 알림 "저장"(이 테이블)은 별개입니다. 채널 발송이
  실패해도 이력 저장 자체는 시도합니다(사용자가 앱에서 다시 확인할 수 있도록).
- `message`는 발송 시점에 이미 완성된 문구를 그대로 저장합니다. 조회 시점에 다시 조합하지
  않습니다(이벤트 payload가 사라져도 과거 알림 문구가 그대로 남도록).
- 알림 채널 설정(Telegram chat id, Discord webhook 등 사용자별 연동 정보)은 이 테이블의
  범위가 아닙니다. 현재 문서에는 이 설정이 어디에 저장되는지 별도로 정의되어 있지 않으며,
  향후 정리가 필요합니다.
- 데이터가 무한히 쌓이는 이력성 테이블이라, 운영 단계에서는 오래된 알림에 대한 보관 주기
  정책(예: N개월 이상 지난 읽은 알림 삭제)이 필요할 수 있습니다(추후 개발 예정).
