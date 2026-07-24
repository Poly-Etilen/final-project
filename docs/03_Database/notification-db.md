# Notification Database

## 개요

Notification Database는 사용자가 등록한 알림 채널(`notification_endpoint`)과 그 채널로
받을 구독(`notification_subscription`), 이벤트×채널별 메시지 템플릿
(`notification_template`), 수신한 이벤트(`notification`)와 채널별 발송 이력
(`notification_delivery`)을 관리하는 Notification Service 소유의 PostgreSQL
Database입니다.

알림 채널은 Telegram/Discord만 지원하며, **재배 단위가 아니라 사용자 단위**로
등록합니다. 사용자는 자신의 채널을 등록한 뒤, 그 채널로 어떤 이벤트를 어떤 대상(예:
특정 재배)에 대해 받을지 개별적으로 구독합니다. 같은 재배에 여러 사용자(OWNER/MEMBER)가
접근하더라도, 각자 자신의 채널/구독을 독립적으로 관리합니다 — 이전처럼 재배 단위로
채널을 공유하지 않습니다.

---

# ERD

```
channel_type (전역 참조, 2종 고정 시드: TELEGRAM/DISCORD)
──────────────────────────────────────────────
PK  id
    code
    display_name
    is_deleted


subscription_target_type (전역 참조, 현재 CULTIVATION 1종만 시드)
──────────────────────────────────────────────
PK  id
    target_type

          │ 1
          ▼
notification_event_type (전역 참조, 6종 고정 시드)
──────────────────────────────────────────────
PK  id
    code
    display_name
    description
FK  target_type_id       (subscription_target_type.id)

          │ 1
          ▼
notification_subscription_type (구독 가능 카탈로그)
──────────────────────────────────────────────
PK  id
FK  notification_event_type_id
FK  subscription_target_type_id
    notification_subscription_name
    description
    created_at
    updated_at
    is_deleted

          │ 1
          ▼
subscription_channel (구독종류별 사용 가능 채널)
──────────────────────────────────────────────
PK  id
FK  notification_subscription_type_id
FK  channel_type_id       (channel_type.id)

    UNIQUE(notification_subscription_type_id, channel_type_id)


notification_event_type            channel_type
    │ 1                                 │ 1
    ▼                                   ▼
notification_template (이벤트 × 채널별 메시지 템플릿, 버전 관리)
──────────────────────────────────────────────
PK  id
FK  notification_event_type_id
FK  channel_type_id
    body_template
    version
    created_at
    updated_at

    UNIQUE(notification_event_type_id, channel_type_id, version)


notification_endpoint (사용자 단위 채널 등록)
──────────────────────────────────────────────
PK  id
    user_id            (Auth Service, 소프트 참조)
FK  channel_type_id
    destination
    display_name
    enabled
    created_at
    updated_at

          │ 1
          ▼
notification_subscription_type      notification_endpoint
          │ 1                              │ 1
          └───────────────┬─────────────────┘
                           ▼
              notification_subscription (사용자의 실제 구독)
              ──────────────────────────────────────────────
              PK  id
              FK  notification_subscription_type_id
              FK  notification_endpoint_id
                  target_id       (예: cultivationId, 소프트 참조)
                  enabled
                  created_at
                  updated_at
                  is_deleted

                  UNIQUE(notification_subscription_type_id,
                         notification_endpoint_id, target_id)


notification_template              notification_subscription
    │ 1                                       │ 1
    ▼                                         │
notification (수신 이벤트 + 렌더링 메시지)         │
──────────────────────────────────────────────  │
PK  id                                           │
FK  notification_template_id                     │
    source_event_id   (UUID)                      │
    event_payload     (JSONB)                      │
    message                                         │
    created_at                                       │
                                                       │
          │ 1                                         │
          ▼                                           ▼
notification_delivery (구독별 발송 이력)
──────────────────────────────────────────────
PK  id
FK  notification_id
FK  notification_subscription_id
    status              (BOOLEAN, 발송 성공 여부)
    provider_message_id
    rendered_message
    attempt_count
    error
    sent_at
    created_at
    updated_at
```

`notification_delivery`는 `notification`과 `notification_subscription` 양쪽을
참조하는 같은 DB 내 실제 FK입니다. 어떤 채널로 보낼지는
`notification_subscription.notification_endpoint_id` → `notification_endpoint`로
결정됩니다.

---

# Table

## channel_type

지원하는 알림 채널 유형의 고정 시드 데이터입니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| code | VARCHAR(20) | X | 채널 코드 (TELEGRAM, DISCORD) |
| display_name | VARCHAR(100) | X | 표시용 이름 |
| is_deleted | BOOLEAN | X | 소프트 삭제 플래그, 기본값 FALSE |

시드 데이터: `TELEGRAM`, `DISCORD` 2건 고정.

---

## subscription_target_type

구독 대상의 유형(현재는 재배 하나뿐)을 정의하는 참조 테이블입니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| target_type | VARCHAR(20) | X | 대상 유형 (현재 CULTIVATION만 사용) |

시드 데이터: `CULTIVATION` 1건. 추후 문의(Inquiry) 등으로 확장될 수 있어 별도
테이블로 분리해 두었습니다.

---

## notification_event_type

기존 `notification_event.event_type` CHECK 제약(enum)을 테이블로 승격한 참조
테이블입니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| code | VARCHAR(50) | X | 이벤트 코드 |
| display_name | VARCHAR(100) | X | 표시용 이름 |
| description | VARCHAR(500) | X | 이벤트 설명 |
| target_type_id | INT | X | FK, `subscription_target_type.id` |

시드 데이터: `ENVIRONMENT_CONTROL`, `SENSOR_ERROR`, `HARVEST_COMPLETED`,
`CULTIVATION_FINISHED`, `CULTIVATION_MODE_CHANGED`, `DAILY_FEEDBACK` 6건, 전부
`target_type_id`는 `CULTIVATION`으로 시드합니다.

CHECK 제약 대신 테이블로 승격한 이유는 각 이벤트가 `target_type_id`(어떤 대상에 대한
이벤트인지)와 `description` 같은 부가 속성을 함께 가져야 하고, 이후
`notification_subscription_type`/`notification_template`이 이 값을 FK로 참조해야
하기 때문입니다.

---

## notification_subscription_type

"구독 가능한 항목"의 카탈로그입니다. 예: "수확 완료 알림(CULTIVATION 대상)".

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| notification_event_type_id | INT | X | FK, `notification_event_type.id` |
| subscription_target_type_id | INT | X | FK, `subscription_target_type.id` |
| notification_subscription_name | VARCHAR(20) | X | 구독 항목 표시명 |
| description | VARCHAR(500) | O | 설명 |
| created_at | DATETIME | X | 생성 일시 |
| updated_at | DATETIME | X | 수정 일시 |
| is_deleted | BOOLEAN | X | 소프트 삭제 플래그, 기본값 FALSE |

현재는 이벤트 6종 × 대상 유형(CULTIVATION 1종)이라 사실상 이벤트 타입마다 구독 항목이
하나씩 있지만, 같은 이벤트 타입이라도 대상 유형이 늘어나면(예: 문의 관련 이벤트가
INQUIRY 대상으로 추가) 조합이 늘어날 수 있어 별도 카탈로그 테이블로 둡니다. 하드
삭제 대신 `is_deleted`를 쓰는 이유는, 이 구독 종류로 이미 생성된
`notification_subscription`/`notification_delivery` 기록을 보존하기 위해서입니다.

---

## subscription_channel

구독 종류별로 실제 발송 가능한 채널을 정의합니다(모든 구독 종류가 모든 채널을 지원한다고
가정하지 않습니다).

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| notification_subscription_type_id | BIGINT | X | FK, `notification_subscription_type.id` |
| channel_type_id | INT | X | FK, `channel_type.id` |

`UNIQUE(notification_subscription_type_id, channel_type_id)`로 중복 등록을 막습니다.

> 외부 ERD는 이 테이블의 PK 컬럼명을 `subscription_channel_id`로 명명했지만, 이
> 프로젝트는 모든 테이블이 PK 컬럼명을 `id`로 통일하는 컨벤션을 쓰고 있어 동일하게
> `id`로 맞췄습니다. 의미/제약은 외부 ERD와 동일합니다.

---

## notification_template

이벤트 × 채널 조합별 메시지 템플릿입니다. Telegram/Discord마다 메시지 포맷이 다를 수
있어, 이벤트 타입별로 채널에 맞는 템플릿을 미리 정의해 두고 발송 시 변수만 치환합니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| notification_event_type_id | INT | X | FK, `notification_event_type.id` |
| channel_type_id | INT | X | FK, `channel_type.id` |
| body_template | TEXT | X | 메시지 템플릿 (예: `🍄 수확 기록 완료\n{{cultivationName}} 수확이 기록되었습니다. 수확량: {{harvestWeight}}kg`) |
| version | INT | X | 템플릿 버전 |
| created_at | DATETIME | X | 생성 일시 |
| updated_at | DATETIME | X | 수정 일시 |

`UNIQUE(notification_event_type_id, channel_type_id, version)`. 같은 이벤트×채널
조합이라도 문구 개선을 위해 버전을 올려 새 행을 추가할 수 있으며, 발송 시점에는 항상
최신 버전을 사용합니다. 과거 버전은 이력으로만 남습니다.

---

## notification_endpoint

사용자 단위로 등록된 알림 채널(Telegram/Discord) 연동 정보입니다. 기존에는
`cultivation_id`를 갖는 재배 단위 테이블이었으나, 사용자 단위로 재설계되었습니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | X | 채널 소유자 (Auth Service, 소프트 참조) |
| channel_type_id | INT | X | FK, `channel_type.id` |
| destination | VARCHAR(500) | X | Telegram Chat ID 또는 Discord Webhook URL |
| display_name | VARCHAR(100) | O | 채널 표시명 |
| enabled | BOOLEAN | X | 채널 활성화 여부, 기본값 TRUE |
| created_at | DATETIME | X | 생성 일시 |
| updated_at | DATETIME | X | 수정 일시 |

한 사용자가 같은 채널 유형을 여러 개 등록할 수 있습니다(예: Telegram 채널 2개). 재배와
무관하게 사용자 프로필처럼 관리되며, `enabled = false`인 endpoint는 조회는 되지만
발송 대상에서 제외됩니다.

---

## notification_subscription

사용자가 실제로 구독한 항목입니다. "내 텔레그램 채널로, 재배 12번의 수확완료 알림을
받겠다" 같은 개별 구독 레코드입니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| notification_subscription_type_id | BIGINT | X | FK, `notification_subscription_type.id` |
| notification_endpoint_id | BIGINT | X | FK, `notification_endpoint.id` |
| target_id | BIGINT | X | 구독 대상 ID (예: cultivationId, 소프트 참조) |
| enabled | BOOLEAN | X | 구독 활성화 여부, 기본값 TRUE |
| created_at | DATETIME | X | 생성 일시 |
| updated_at | DATETIME | X | 수정 일시 |
| is_deleted | BOOLEAN | X | 소프트 삭제 플래그, 기본값 FALSE |

`UNIQUE(notification_subscription_type_id, notification_endpoint_id, target_id)`로
같은 채널이 같은 대상에 같은 구독 종류를 중복 등록하지 못하게 막습니다. `target_id`는
FK가 아니라 소프트 참조입니다 — `notification_subscription_type.subscription_target_type_id`가
가리키는 유형(현재는 전부 CULTIVATION)에 따라 의미가 결정되는 다형성 참조이기
때문입니다. `enabled`(사용자가 잠시 끄고 켜는 스위치)와 `is_deleted`(구독 취소, 이력
보존을 위한 소프트 삭제)는 별개 상태입니다.

---

## notification

기존 `notification_event`를 리네이밍하고, 템플릿 기반으로 재구성한 테이블입니다.
수신한 RabbitMQ 이벤트와 그 이벤트를 공통 템플릿으로 렌더링한 메시지를 기록합니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| notification_template_id | BIGINT | X | FK, `notification_template.id` |
| source_event_id | UUID | X | RabbitMQ 이벤트 ID (중복 처리 방지용), UNIQUE |
| event_payload | JSONB | X | 원본 이벤트 데이터 (예: cultivationId, harvestWeight 등) |
| message | TEXT | X | 템플릿에 변수를 채운 렌더링 결과 (채널 무관, 공통 메시지) |
| created_at | DATETIME | X | 생성 일시 |

`source_event_id`에 `UNIQUE` 제약을 두어, RabbitMQ 재전송으로 같은 이벤트가 중복
수신되더라도 `notification` 행이 중복 생성되지 않도록(멱등 처리) 합니다.
`notification_template_id`는 발송 시점에 사용한 템플릿(대표로 하나) 기준이며, 실제
채널별 최종 문구는 `notification_delivery.rendered_message`에 별도로 남습니다.

---

## notification_delivery

`notification` 하나가 특정 구독(`notification_subscription`)으로 발송된 이력입니다.
같은 이벤트도 매칭된 구독 수만큼 여러 행이 생깁니다. 기존에는 `endpoint_id`를 직접
참조했지만, 재설계 이후에는 `notification_subscription_id`를 참조합니다 — 같은
endpoint라도 구독마다(대상마다) 별도로 발송 이력이 남아야 하기 때문입니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| notification_id | BIGINT | X | FK, `notification.id` (기존 `notification_event_id`) |
| notification_subscription_id | BIGINT | X | FK, `notification_subscription.id` (기존 `endpoint_id` 대체) |
| status | BOOLEAN | X | 발송 성공 여부 |
| provider_message_id | VARCHAR(100) | O | Telegram/Discord가 반환한 메시지 ID |
| rendered_message | TEXT | X | 해당 구독(채널)으로 실제 발송된 문구 |
| attempt_count | SMALLINT | X | 발송 시도 횟수, 기본값 0 |
| error | TEXT | O | 마지막 발송 실패 원인 (기존 `last_error_message` 리네이밍) |
| sent_at | DATETIME | O | 발송 성공 일시 |
| created_at | DATETIME | X | 생성 일시 |
| updated_at | DATETIME | X | 수정 일시 |

기존에는 `sent_at` 유무로 발송 성공 여부를 판단했지만, 외부 ERD를 따라 `status`
BOOLEAN 컬럼으로 명시했습니다. `attempt_count`는 외부 ERD에 `TINYINT`로 정의되어
있으나 PostgreSQL에는 `TINYINT`가 없어 `SMALLINT`로 매핑했습니다(값 범위상 문제 없음).

---

# DDL

```sql
CREATE TABLE channel_type (
    id BIGSERIAL PRIMARY KEY,

    code VARCHAR(20) NOT NULL,

    display_name VARCHAR(100) NOT NULL,

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    CONSTRAINT uk_channel_type_code UNIQUE (code)
);
```

```sql
CREATE TABLE subscription_target_type (
    id BIGSERIAL PRIMARY KEY,

    target_type VARCHAR(20) NOT NULL,

    CONSTRAINT uk_subscription_target_type UNIQUE (target_type)
);
```

```sql
CREATE TABLE notification_event_type (
    id BIGSERIAL PRIMARY KEY,

    code VARCHAR(50) NOT NULL,

    display_name VARCHAR(100) NOT NULL,

    description VARCHAR(500) NOT NULL,

    target_type_id INT NOT NULL,

    CONSTRAINT uk_notification_event_type_code UNIQUE (code),

    CONSTRAINT fk_notification_event_type_target
        FOREIGN KEY (target_type_id) REFERENCES subscription_target_type(id)
);
```

```sql
CREATE TABLE notification_subscription_type (
    id BIGSERIAL PRIMARY KEY,

    notification_event_type_id INT NOT NULL,

    subscription_target_type_id INT NOT NULL,

    notification_subscription_name VARCHAR(20) NOT NULL,

    description VARCHAR(500),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    CONSTRAINT fk_notification_subscription_type_event
        FOREIGN KEY (notification_event_type_id) REFERENCES notification_event_type(id),

    CONSTRAINT fk_notification_subscription_type_target
        FOREIGN KEY (subscription_target_type_id) REFERENCES subscription_target_type(id)
);
```

```sql
CREATE TABLE subscription_channel (
    id BIGSERIAL PRIMARY KEY,

    notification_subscription_type_id BIGINT NOT NULL,

    channel_type_id INT NOT NULL,

    CONSTRAINT fk_subscription_channel_type
        FOREIGN KEY (notification_subscription_type_id) REFERENCES notification_subscription_type(id),

    CONSTRAINT fk_subscription_channel_channel
        FOREIGN KEY (channel_type_id) REFERENCES channel_type(id),

    CONSTRAINT uk_subscription_channel UNIQUE (notification_subscription_type_id, channel_type_id)
);
```

```sql
CREATE TABLE notification_template (
    id BIGSERIAL PRIMARY KEY,

    notification_event_type_id INT NOT NULL,

    channel_type_id INT NOT NULL,

    body_template TEXT NOT NULL,

    version INT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_notification_template_event
        FOREIGN KEY (notification_event_type_id) REFERENCES notification_event_type(id),

    CONSTRAINT fk_notification_template_channel
        FOREIGN KEY (channel_type_id) REFERENCES channel_type(id),

    CONSTRAINT uk_notification_template UNIQUE (notification_event_type_id, channel_type_id, version)
);
```

```sql
CREATE TABLE notification_endpoint (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT NOT NULL,

    channel_type_id INT NOT NULL,

    destination VARCHAR(500) NOT NULL,

    display_name VARCHAR(100),

    enabled BOOLEAN NOT NULL DEFAULT TRUE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_notification_endpoint_channel
        FOREIGN KEY (channel_type_id) REFERENCES channel_type(id)
);
```

```sql
CREATE TABLE notification_subscription (
    id BIGSERIAL PRIMARY KEY,

    notification_subscription_type_id BIGINT NOT NULL,

    notification_endpoint_id BIGINT NOT NULL,

    target_id BIGINT NOT NULL,

    enabled BOOLEAN NOT NULL DEFAULT TRUE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    CONSTRAINT fk_notification_subscription_type
        FOREIGN KEY (notification_subscription_type_id) REFERENCES notification_subscription_type(id),

    CONSTRAINT fk_notification_subscription_endpoint
        FOREIGN KEY (notification_endpoint_id) REFERENCES notification_endpoint(id),

    CONSTRAINT uk_notification_subscription
        UNIQUE (notification_subscription_type_id, notification_endpoint_id, target_id)
);
```

```sql
CREATE TABLE notification (
    id BIGSERIAL PRIMARY KEY,

    notification_template_id BIGINT NOT NULL,

    source_event_id UUID NOT NULL,

    event_payload JSONB NOT NULL,

    message TEXT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_notification_template
        FOREIGN KEY (notification_template_id) REFERENCES notification_template(id),

    CONSTRAINT uk_notification_source_event UNIQUE (source_event_id)
);
```

```sql
CREATE TABLE notification_delivery (
    id BIGSERIAL PRIMARY KEY,

    notification_id BIGINT NOT NULL,

    notification_subscription_id BIGINT NOT NULL,

    status BOOLEAN NOT NULL,

    provider_message_id VARCHAR(100),

    rendered_message TEXT NOT NULL,

    attempt_count SMALLINT NOT NULL DEFAULT 0,

    error TEXT,

    sent_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_notification_delivery_notification
        FOREIGN KEY (notification_id) REFERENCES notification(id),

    CONSTRAINT fk_notification_delivery_subscription
        FOREIGN KEY (notification_subscription_id) REFERENCES notification_subscription(id)
);
```

`notification_endpoint.user_id`와 `notification_subscription.target_id`는 각각 Auth
Service, Cultivation Service의 데이터를 가리키는 소프트 참조라 FK를 걸지 않습니다. 그
외 이 문서의 모든 FK는 Notification Service가 소유한 같은 DB 안의 테이블을 가리키므로
실제 FK로 강제합니다.

---

# Index

```sql
CREATE INDEX idx_notification_endpoint_user
ON notification_endpoint(user_id);
```

사용자별로 등록된 채널 목록 조회에 사용합니다.

```sql
CREATE INDEX idx_notification_subscription_endpoint
ON notification_subscription(notification_endpoint_id);
```

특정 채널이 구독 중인 목록(내 구독 목록 조회 API)을 가져올 때 사용합니다.

```sql
CREATE INDEX idx_notification_subscription_type_target
ON notification_subscription(notification_subscription_type_id, target_id)
WHERE enabled = TRUE AND is_deleted = FALSE;
```

이벤트 수신 시 "이 구독 종류 + 이 대상(target_id)"을 구독 중인 활성 구독을 조회하는
핵심 인덱스입니다. 알림 발송 경로에서 가장 자주 조회되는 패턴이라 활성 구독만 남기는
부분 인덱스(partial index)로 구성했습니다.

```sql
CREATE INDEX idx_notification_template_event_channel_version
ON notification_template(notification_event_type_id, channel_type_id, version DESC);
```

이벤트×채널에 대한 최신 버전 템플릿을 조회할 때 사용합니다.

```sql
CREATE INDEX idx_notification_delivery_notification
ON notification_delivery(notification_id);
```

특정 이벤트(`notification`)가 어떤 구독들로 발송됐는지 조회할 때 사용합니다.

```sql
CREATE INDEX idx_notification_delivery_subscription_created
ON notification_delivery(notification_subscription_id, created_at DESC);
```

특정 구독의 알림 발송 이력을 최신순으로 조회할 때 사용합니다.

---

# 관계

```
notification_endpoint (Notification Service)

    │ user_id (소프트 참조)
    ▼
Auth Service


notification_subscription (Notification Service)

    │ target_id (소프트 참조, notification_subscription_type이 가리키는
    │            subscription_target_type에 따라 의미 결정 — 현재는 전부 cultivationId)
    ▼
Cultivation Service


channel_type, subscription_target_type, notification_event_type,
notification_subscription_type, subscription_channel, notification_template,
notification_endpoint, notification_subscription, notification, notification_delivery
(Notification Service)

    │ 서로 같은 DB 내 실제 FK로 연결
    ▼
위 10개 테이블 간 참조는 모두 이 서비스 안에서 해결됩니다.
```

Notification Service는 다른 서비스와 데이터베이스를 공유하지 않으며, `user_id`와
`target_id`를 통해서만 간접적으로 참조됩니다.

---

# 다른 서비스와의 연관

Rule Engine / Sensor / Cultivation / AI Service와 직접 DB를 공유하지 않습니다. 각
서비스가 발행한 RabbitMQ 이벤트를 구독해 `notification_event_type`에 매칭한 뒤
`notification` 행을 생성합니다.

---

# 고려 사항

- 알림 "발송"과 "기록"은 별개입니다. 채널 발송이 실패해도 `notification` 기록 자체는
  시도합니다.
- `notification_delivery`는 이벤트 × 매칭된 활성 구독 수만큼 생성됩니다. 특정
  대상(target_id)에 대해 등록된 구독이 없으면 `notification`만 남고
  `notification_delivery`는 생성되지 않습니다.
- 재시도 정책(최대 횟수, 주기)은 `attempt_count`/`error`로 이력만 남기며, 구체적인
  재시도 스케줄링은 애플리케이션 레벨에서 처리합니다.
- `notification_endpoint`가 사용자 단위로 바뀌면서, 같은 재배에 여러 사용자(OWNER/
  MEMBER)가 접근하더라도 각자 독립적으로 채널을 등록하고 구독을 관리합니다 — 더 이상
  채널이 재배 단위로 공유되지 않습니다.
- 기존 `notification_event.event_type` CHECK enum을 `notification_event_type` 테이블로
  승격했습니다. 새 이벤트 타입을 추가할 때 애플리케이션 배포 없이 데이터만 추가할 수
  있지만, 실제로 발송되려면 대응하는 `notification_template`(이벤트×채널)과
  `notification_subscription_type`(구독 카탈로그)도 함께 등록해야 합니다.
- `notification_subscription_type.is_deleted`는 하드 삭제 대신 사용합니다 — 이
  구독 종류로 이미 생성된 `notification_subscription`/`notification_delivery` 기록을
  보존하기 위해서입니다.
- `notification.source_event_id`에 `UNIQUE` 제약을 두어 RabbitMQ 재전송에 의한
  이벤트 중복 처리를 막습니다(멱등 처리).
- `subscription_channel` 테이블의 PK 컬럼명은 외부 ERD의 `subscription_channel_id`
  대신 이 프로젝트의 "모든 테이블은 PK를 `id`로 명명" 컨벤션을 따라 `id`로
  통일했습니다.
- 외부 ERD의 `notification_delivery.attempt_count` 타입(`TINYINT`)은 PostgreSQL에
  없는 타입이라 `SMALLINT`로 매핑했습니다.
- 재배 생성/멤버 참여 시 주요 이벤트를 자동으로 구독시켜 줄지는 아직 정해지지
  않았습니다. 현재는 사용자가 채널 등록 후 직접 구독을 설정해야 알림을 받습니다.
