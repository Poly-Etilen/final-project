# Database Overview

## 데이터베이스 구성

본 프로젝트는 데이터의 특성에 따라 서로 다른 데이터베이스를 사용하며, "서비스마다 자기 DB만
소유한다"는 원칙(database per service)을 따릅니다. 같은 DB 안의 테이블끼리는 실제 FK를
걸고, 서로 다른 서비스가 소유한 데이터는 FK 없는 순수 값(소프트 참조)으로만 연결합니다.

| Database | 용도 | 사용 서비스 |
|----------|------|------------|
| PostgreSQL | 관계형 데이터 저장 | Auth, Cultivation, Notification, AI |
| Redis | 캐시 및 임시 데이터 | Auth, AI, Rule Engine, Cultivation |
| InfluxDB | 시계열 센서 데이터 | Cultivation |
| MinIO / 로컬 저장소 | 생육 사진 / 프로필 이미지 저장 | Auth, Cultivation, AI |

---

# PostgreSQL

## Auth DB

인증 정보와 회원 프로필을 관리합니다.

### Table

- users (email/nickname UNIQUE, status ACTIVE/DORMANT/DELETED, 탈퇴는 status+deleted_at Soft Delete, `profile_image_url` 컬럼 제거)
- oauth_user (provider 정규화, UNIQUE(provider, provider_user_id), users에 FK)
- profile_image (신규. user_id UNIQUE, object_key/storage_type 패턴으로 프로필 이미지 저장 — Auth Service가 Photo Storage를 사용하는 첫 사례)

자세한 내용은 [auth-db.md](./auth-db.md) 참고.

---

## Cultivation DB

버섯 재배 자체(생성/진행/종료, 수확 기록, 사진)와 센서 장치, 목표 환경(위험
한계값) 이력, 버섯 참조 데이터, 문의(Inquiry)까지 함께 관리합니다(구 Sensor
Service 통합).

### Table

- cultivation (UNIQUE(user_id, name), status CREATED/RUNNING/FINISHED, mode GROWTH/HARVEST(생육 분석 결과로 자동 전환), mushroom_id는 같은 DB의 `mushroom_reference.id`를 실제 FK로 참조)
- cultivation_member (재배 공유, UNIQUE(cultivation_id, user_id), role OWNER/MEMBER, 재배 생성 시 생성자가 OWNER로 자동 등록)
- harvest (cultivation당 한 건, UNIQUE(cultivation_id). harvest_weight NUMERIC(6,2)/product_score NUMERIC(4,1). product_score/product_grade는 AI Service가 계산해 전달, 등급 매핑은 Cultivation Service가 수행)
- cultivation_photo (구 `photo`. object_key + storage_type으로 저장소 중립적 메타데이터 관리)
- sensor_type (구 `measurement_type`. 온도/습도/CO2/조도 4종 참조 테이블, 자연키 code → 대리키 id로 전환, 컬럼: sensor_type/value_unit — 여러 테이블이 공유하는 항목 도메인을 한 곳에서 정의)
- cultivation_sensor (구 `sensor`. PK는 대리키 id, device_eui UNIQUE, is_deleted 소프트 삭제, cultivation_id는 실제 FK)
- cultivation_sensor_type (구 `sensor_type` 브릿지 테이블. 센서 1대가 여러 항목을 측정할 수 있는 1:N 하위 테이블, cultivation_sensor/sensor_type FK)
- environment_setting (재배별 목표 환경 이력, INSERT-only, sensor_type_id FK, threshold_unit 컬럼 제거(sensor_type.value_unit 조회), cultivation_id는 실제 FK)
- mushroom_reference (버섯 5종 참조 데이터, 자연키 코드 없이 대리키 id로만 식별, RAG용 텍스트 컬럼 포함)
- mushroom_reference_threshold (버섯별 항목별 추천 범위, mushroom_reference/sensor_type_id FK, threshold_unit 컬럼 제거)
- inquiry_category (신규 참조 테이블. 시드: 일반 문의/경작 문의 — 기존 `inquiry.type` enum을 FK 방식으로 전환)
- inquiry (category_id FK로 문의 유형 지정, status는 PENDING/RESOLVED 2단계로 축소(기존 OPEN/ANSWERED/CLOSED), cultivation_id는 재배 삭제 후 기록 보존을 위해 의도적으로 소프트 참조)
- inquiry_answer (신규. 스레드형 답변 테이블, pre_id self-FK로 문의 본문/답변 체인 구성 — 기존 `inquiry.answer` 단일 컬럼 대체)

자세한 내용은 [cultivation-db.md](./cultivation-db.md) 참고.

---

## Notification DB

사용자 단위로 등록한 알림 채널과 그 채널로 받을 구독, 이벤트×채널별 메시지 템플릿,
수신한 이벤트와 채널별 발송 이력을 관리합니다. 기존 3테이블(재배 단위 채널 공유
모델)에서 10테이블(사용자 단위 구독 모델)로 전면 재설계되었습니다 — 사용자가 채널을
등록한 뒤 "어떤 재배의 어떤 이벤트를 받을지" 개별적으로 구독해야 하는 구조로
바뀌었습니다.

### Table

- channel_type (신규 참조 테이블. TELEGRAM/DISCORD 2종 고정 시드)
- subscription_target_type (신규 참조 테이블. 구독 대상 유형, 현재 CULTIVATION 1종만 시드)
- notification_event_type (신규 참조 테이블. 기존 `notification_event.event_type` enum을 테이블로 승격, target_type_id FK)
- notification_subscription_type (신규 카탈로그. 구독 가능한 이벤트×대상 조합, notification_event_type/subscription_target_type FK)
- subscription_channel (신규 카탈로그. 구독 종류별 사용 가능 채널, notification_subscription_type/channel_type FK)
- notification_template (신규. 이벤트×채널별 메시지 템플릿, version으로 이력 관리)
- notification_endpoint (구 재배 단위 → 사용자 단위로 전환. user_id 기반 Telegram/Discord 연동 정보)
- notification_subscription (신규. 사용자가 특정 채널로 특정 재배의 특정 이벤트를 구독하는 실제 레코드, target_id로 대상 지정)
- notification (구 `notification_event` 리네이밍. source_event_id(UUID)/event_payload(JSONB)/notification_template_id 추가)
- notification_delivery (채널별 발송 이력 — 기존 endpoint_id 참조 → notification_subscription_id 참조로 변경, notification/notification_subscription FK)

재배 멤버(cultivation_member) 여러 명이 각자 독립적으로 채널/구독을 관리할 수 있게
된 것이 이번 재설계의 핵심 장점입니다. 재배 생성/멤버 참여 시 기본 구독을 자동으로
만들어줄지는 아직 미정이며 추후 개발 예정입니다.

자세한 내용은 [notification-db.md](./notification-db.md) 참고.

---

## AI DB

AI 챗봇 대화 이력, 생육 분석 이력, 일일 피드백 이력, 인사이트 사례 원본을 관리합니다.

### Table

- chat_conversation (신규. 대화방 단위 — 기존 `chat_log` 단일 테이블을 대화-메시지로 분리하며 생김. user_id/cultivation_id 소프트 참조, channel_type)
- chat_message (구 `chat_log`의 후신. 발화 하나당 한 행, chat_conversation_id는 같은 DB 내 실제 FK, sequence_number로 순서 관리)
- growth_record (Vision 분석 결과 이력. 정규화 컬럼들 → `analysis_data` JSONB 하나로 통합, object_key/storage_type 스냅샷 → `cultivation_photo_id` 소프트 참조로 단순화)
- daily_feedback (UNIQUE(cultivation_id, feedback_date), 변경 없음 — 환경 통계 컬럼 그대로 유지)
- insight ("인사이트" 사례, `harvest_id` 컬럼 제거하고 `cultivation_id` UNIQUE로 중복 적재 방지 키 이관, `harvest_weight` → `harvest_weight_grams`로 리네이밍 — 수확 완료 시점에 즉시 적재, mushroom_id/avg_temperature 인덱스로 1차 필터 + 나머지 환경값 조건으로 SQL 검색)

외부 ERD의 `mushroom_embedding` 벡터 검색 기능은 이 프로젝트에서 채택하지 않았습니다
(버섯 5종 고정이라 임베딩 없이 직접 조회로 충분합니다).

자세한 내용은 [ai-db.md](./ai-db.md) 참고.

---

## DatasourceGenerator (DB 없음)

DatasourceGenerator는 별도의 PostgreSQL DB를 사용하지 않습니다. 센서 데이터 생성/발행에
필요한 최소 정보(device_eui/cultivationId/sensorTypes)는 메모리(In-Memory)에서만 관리하며,
Cultivation Service가 발행하는 이벤트(평상시) + 서비스 시작 시 OpenFeign 전체 조회로 채워집니다.
자세한 내용은 [datasource-generator-db.md](./datasource-generator-db.md) 참고.

---

# Redis

| 사용처 | Key | TTL |
|--------|-----|-----|
| Auth: Refresh Token | `refresh:{userId}` | 14일 |
| Auth: 이메일 인증 | `email:{email}` | 5분 |
| AI: 챗봇 응답 캐시 | `ai:{hash}` | 24시간 |
| AI: 생육 분석 결과 캐시 | `ai:{cultivationId}:analysis` | 6시간 |
| AI: 버섯 가이드 캐시 | `ai:mushroom:{mushroomId}:guide` | 7일 |
| AI: 인사이트 후보 캐시 | `ai:{cultivationId}:insight:candidates` | 24시간 (요청 시 채워짐) |
| Rule Engine: 목표 환경 범위 캐시 | `cultivation:{cultivationId}:range` | 24시간 (write-through) |
| Cultivation: 최신 센서 데이터 | `cultivation:{cultivationId}:current` | 없음 (매초 덮어씀) |

자세한 내용은 [redis.md](./redis.md) 참고.

---

# InfluxDB

## Measurement

```
environment
```

### Tags

- cultivationId
- deviceEui

### Fields

- temperature
- humidity
- co2
- light

Cultivation Service가 저장/조회를 전담합니다. 매초 들어오는 이벤트를 그대로 다 기록하지 않고,
재배별로 10초 간격으로 스로틀링하여 저장합니다. (자세한 내용은 [influxdb.md](./influxdb.md) 참고)

---

# Photo Storage (MinIO / 로컬)

사진 원본 파일은 MinIO 또는 로컬 파일시스템에 저장하고, 메타데이터(`object_key`,
`storage_type`)는 Cultivation DB의 `cultivation_photo`, Auth DB의 `profile_image`(신규,
프로필 이미지 업로드)에서 관리합니다. AI DB의 `growth_record`는 더 이상 사진 메타데이터를
스냅샷으로 복사하지 않고, `cultivation_photo_id`로 `cultivation_photo`를 소프트
참조합니다. 완성된 URL이 아닌 저장소 중립적인 키를 저장해, 저장소 전환 시에도 기존
데이터를 다시 쓸 필요가 없도록 했습니다. 자세한 내용은 [minio.md](./minio.md) 참고.

---

# 서비스 간 참조 원칙

- **같은 DB 안**: 실제 FK를 겁니다. (예: `oauth_user.user_id → users.id`, `profile_image.user_id → users.id`, `harvest.cultivation_id → cultivation.id`, `cultivation_sensor.cultivation_id → cultivation.id`, `cultivation.mushroom_id → mushroom_reference.id`, `notification_subscription.notification_endpoint_id → notification_endpoint.id`)
- **다른 서비스의 DB**: FK 없는 순수 값(BIGINT/VARCHAR)으로만 참조하고, 존재/소유권 검증이
  필요하면 해당 서비스를 OpenFeign으로 호출해 확인합니다. (예: `cultivation.user_id`,
  `notification_endpoint.user_id`, `notification_subscription.target_id`)
- **같은 DB인데도 예외적으로 소프트 참조**: `inquiry.cultivation_id`는 같은 Cultivation DB
  안에 있지만, 재배 삭제 후에도 문의 기록을 보존해야 해서 의도적으로 FK를 걸지 않았습니다.
- **삭제 전파**: 같은 DB라면 `ON DELETE CASCADE`로 자동 처리되지만, 다른 DB 간에는
  이벤트(`CultivationDeletedEvent` 등)를 구독해 보상 삭제합니다.
