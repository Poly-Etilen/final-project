# Notification Database

## 개요

Notification Database는 수신한 알림 이벤트(`notification_event`), 채널별 발송 이력
(`notification_delivery`), 재배별 알림 채널 연동 정보(`notification_endpoint`) 3개
테이블로 알림을 관리하는 Notification Service 소유의 PostgreSQL Database입니다.

알림 채널은 Telegram/Discord만 지원하며, **사용자 계정이 아니라 재배 단위**로
등록합니다. 같은 재배에 접근하는 사용자라면 등록된 채널을 함께 씁니다.

---

# ERD

```
notification_event
──────────────────────────────────────────────
PK  id
    event_type
    message
    created_at

          │ 1
          │
          │
          ▼
notification_delivery
──────────────────────────────────────────────
PK  id
FK  notification_event_id
FK  endpoint_id
    rendered_message
    provider_message_id
    attempt_count
    last_error_message
    sent_at
    created_at

notification_endpoint
──────────────────────────────────────────────
PK  id
    cultivation_id
    channel_type
    destination
    display_name
    enabled
    created_at
    updated_at
```

`notification_delivery`는 `notification_event`와 `notification_endpoint` 양쪽을 참조하는
같은 DB 내 실제 FK입니다. 어떤 재배로 보낼지는 `notification_delivery.endpoint_id` →
`notification_endpoint.cultivation_id`로 결정됩니다.

---

# Table

## notification_event

수신한 RabbitMQ 이벤트를 기록하는 테이블입니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| event_type | VARCHAR(50) | X | 이벤트 유형 (ENVIRONMENT_CONTROL/SENSOR_ERROR/HARVEST_COMPLETED/CULTIVATION_FINISHED/WEEKLY_REPORT/DAILY_FEEDBACK) |
| message | TEXT | O | 사용자에게 보여줄 기본 메시지 |
| created_at | DATETIME | X | 수신 일시 |

어떤 재배에서 발생한 이벤트인지는 이 테이블이 직접 담지 않습니다. 발행 서비스가 RabbitMQ
메시지에 `cultivationId`를 함께 실어 보내면, Notification Service가 이 값으로
`notification_endpoint`를 조회해 발송 대상만 결정하고 이벤트 자체는 최소한으로 기록합니다.

---

## notification_delivery

`notification_event` 하나가 특정 채널(`notification_endpoint`)로 발송된 이력입니다.
같은 이벤트도 재배가 등록한 채널 수만큼 여러 행이 생깁니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| notification_event_id | BIGINT | X | FK, `notification_event.id` |
| endpoint_id | BIGINT | X | FK, `notification_endpoint.id` |
| rendered_message | TEXT | X | 해당 채널로 실제 발송된 문구(렌더링 완료본) |
| provider_message_id | VARCHAR(255) | O | Telegram/Discord가 반환한 메시지 ID |
| attempt_count | INT | X | 발송 시도 횟수, 기본값 0 |
| last_error_message | TEXT | O | 마지막 발송 실패 원인 |
| sent_at | DATETIME | O | 발송 성공 일시. NULL이면 아직 발송되지 않았거나 실패한 상태 |
| created_at | DATETIME | X | 생성 일시 |

`rendered_message`는 발송 시점에 채널별로 완성된 문구를 저장합니다 — 이벤트가 사라져도
과거 발송 문구가 그대로 남도록 하기 위함입니다. 발송 성공/실패는 상태 컬럼 없이
`sent_at`(성공 시각) 유무와 `attempt_count`/`last_error_message`(재시도 이력)로 판단합니다.

---

## notification_endpoint

재배 단위로 등록된 알림 채널(Telegram/Discord) 연동 정보입니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 채널이 연결된 재배 (Cultivation Service, 소프트 참조) |
| channel_type | VARCHAR(20) | X | 채널 유형 (TELEGRAM, DISCORD) |
| destination | VARCHAR(500) | X | Telegram Chat ID 또는 Discord Webhook URL |
| display_name | VARCHAR(100) | O | 채널 표시명 |
| enabled | BOOLEAN | X | 채널 활성화 여부, 기본값 TRUE |
| created_at | DATETIME | X | 생성 일시 |
| updated_at | DATETIME | X | 수정 일시 |

한 재배가 같은 채널 유형을 여러 개 등록할 수 있습니다. `enabled = false`인 endpoint는
조회는 되지만 발송 대상에서 제외됩니다.

---

# DDL

```sql
CREATE TABLE notification_event (
    id BIGSERIAL PRIMARY KEY,

    event_type VARCHAR(50) NOT NULL,

    message TEXT,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_notification_event_type CHECK (event_type IN (
        'ENVIRONMENT_CONTROL',
        'SENSOR_ERROR',
        'HARVEST_COMPLETED',
        'CULTIVATION_FINISHED',
        'WEEKLY_REPORT',
        'DAILY_FEEDBACK'
    ))
);
```

```sql
CREATE TABLE notification_endpoint (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    channel_type VARCHAR(20) NOT NULL,

    destination VARCHAR(500) NOT NULL,

    display_name VARCHAR(100),

    enabled BOOLEAN NOT NULL DEFAULT TRUE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_notification_endpoint_channel CHECK (channel_type IN ('TELEGRAM', 'DISCORD'))
);
```

```sql
CREATE TABLE notification_delivery (
    id BIGSERIAL PRIMARY KEY,

    notification_event_id BIGINT NOT NULL,

    endpoint_id BIGINT NOT NULL,

    rendered_message TEXT NOT NULL,

    provider_message_id VARCHAR(255),

    attempt_count INT NOT NULL DEFAULT 0,

    last_error_message TEXT,

    sent_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_notification_delivery_event
        FOREIGN KEY (notification_event_id) REFERENCES notification_event(id),

    CONSTRAINT fk_notification_delivery_endpoint
        FOREIGN KEY (endpoint_id) REFERENCES notification_endpoint(id)
);
```

`notification_endpoint.cultivation_id`는 Cultivation Service의 데이터를 가리키는 소프트
참조라 FK를 걸지 않습니다. `notification_delivery`가 참조하는 `notification_event`/
`notification_endpoint`는 모두 이 서비스가 소유한 같은 DB 안의 테이블이므로 실제 FK를
겁니다.

---

# Index

```sql
CREATE INDEX idx_notification_endpoint_cultivation
ON notification_endpoint(cultivation_id);
```

재배별로 등록된 채널 목록 조회에 사용합니다.

```sql
CREATE INDEX idx_notification_delivery_event
ON notification_delivery(notification_event_id);
```

특정 이벤트가 어떤 채널들로 발송됐는지 조회할 때 사용합니다.

```sql
CREATE INDEX idx_notification_delivery_endpoint_created
ON notification_delivery(endpoint_id, created_at DESC);
```

특정 재배(채널)의 알림 발송 이력을 최신순으로 조회할 때 사용합니다.

---

# 관계

```
notification_endpoint (Notification Service)

    │ cultivation_id (소프트 참조)
    ▼
Cultivation Service


notification_delivery (Notification Service)

    │ notification_event_id, endpoint_id (같은 DB 내 실제 FK)
    ▼
notification_event, notification_endpoint
```

Notification Service는 다른 서비스와 데이터베이스를 공유하지 않으며, `cultivationId`를
통해서만 간접적으로 참조됩니다.

---

# 다른 서비스와의 연관

Rule Engine / Sensor / Cultivation / AI Service와 직접 DB를 공유하지 않습니다. 각
서비스가 발행한 RabbitMQ 이벤트를 구독해 `notification_event` 행을 생성합니다.

---

# 고려 사항

- 알림 "발송"과 "기록"은 별개입니다. 채널 발송이 실패해도 이벤트 기록 자체는 시도합니다.
- `notification_delivery`는 이벤트 × 등록된 채널 수만큼 생성됩니다. 등록된 채널이 없는
  재배는 `notification_event`만 남고 `notification_delivery`는 생성되지 않습니다.
- 재시도 정책(최대 횟수, 주기)은 `attempt_count`/`last_error_message`로 이력만 남기며,
  구체적인 재시도 스케줄링은 애플리케이션 레벨에서 처리합니다.
- `notification_endpoint`가 재배 단위이므로, 같은 재배에 여러 사용자가 접근할 수 있다면
  등록된 채널도 함께 공유됩니다.
