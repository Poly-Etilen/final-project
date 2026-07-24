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
| MinIO / 로컬 저장소 | 생육 사진(이미지) 저장 | Cultivation, AI |

---

# PostgreSQL

## Auth DB

인증 정보와 회원 프로필을 관리합니다.

### Table

- users (email/nickname UNIQUE, status ACTIVE/DORMANT/DELETED, 탈퇴는 status+deleted_at Soft Delete)
- oauth_user (provider 정규화, UNIQUE(provider, provider_user_id), users에 FK)

자세한 내용은 [auth-db.md](./auth-db.md) 참고.

---

## Cultivation DB

버섯 재배 자체(생성/진행/종료, 수확 기록, 사진)와 센서 장치, 목표 환경(위험
한계값) 이력, 버섯 참조 데이터, 문의(Inquiry)까지 함께 관리합니다(구 Sensor
Service 통합).

### Table

- cultivation (UNIQUE(user_id, name), status CREATED/RUNNING/FINISHED, mushroom_type은 같은 DB의 `mushroom_reference`를 실제 FK로 참조)
- harvest (cultivation당 여러 건, UNIQUE(cultivation_id, flush_no))
- photo (object_key + storage_type으로 저장소 중립적 메타데이터 관리)
- measurement_type (온도/습도/CO2/조도 4종 참조 테이블 — 여러 테이블이 공유하는 항목 도메인을 한 곳에서 정의)
- sensor (PK는 대리키 id, device_eui UNIQUE, is_deleted 소프트 삭제, cultivation_id는 실제 FK)
- sensor_type (센서 1대가 여러 항목을 측정할 수 있는 1:N 하위 테이블, measurement_type FK)
- environment_setting (재배별 목표 환경 이력, INSERT-only, measurement_type FK, cultivation_id는 실제 FK)
- mushroom_reference (버섯 5종 참조 데이터, mushroom_type 코드 UNIQUE, RAG용 텍스트 컬럼 포함)
- mushroom_reference_threshold (버섯별 항목별 추천 범위, mushroom_reference/measurement_type FK)
- inquiry (일반 문의/경작 문의, cultivation_id는 재배 삭제 후 기록 보존을 위해 의도적으로 소프트 참조)

자세한 내용은 [cultivation-db.md](./cultivation-db.md) 참고.

---

## Notification DB

수신한 알림 이벤트, 채널별 발송 이력, 재배별 알림 채널 등록 정보를 관리합니다.

### Table

- notification_event (event_type/message만 기록하는 최소 이벤트 로그)
- notification_delivery (채널별 발송 이력 — rendered_message, attempt_count, notification_event/notification_endpoint FK)
- notification_endpoint (재배 단위로 등록되는 Telegram/Discord 연동 정보)

자세한 내용은 [notification-db.md](./notification-db.md) 참고.

---

## AI DB

AI 챗봇 대화 이력, 생육 분석 이력, 일일 피드백 이력, 인사이트 사례 원본을 관리합니다.

### Table

- chat_log (발화 하나당 한 행. APP은 user_id 필수/cultivation_id 선택, TELEGRAM/DISCORD는 반대)
- growth_record (Vision 분석 결과 이력, object_key/storage_type으로 사진 스냅샷 보관)
- daily_feedback (UNIQUE(cultivation_id, feedback_date))
- insight ("인사이트" 사례, harvest_id UNIQUE — 수확 완료 시점에 즉시 적재, mushroom_type/avg_temperature 인덱스로 1차 필터 + 나머지 환경값 조건으로 SQL 검색)

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
| AI: 버섯 가이드 캐시 | `ai:mushroom:{mushroomType}:guide` | 7일 |
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
`storage_type`)는 Cultivation DB의 `photo`, AI DB의 `growth_record`에서 관리합니다.
완성된 URL이 아닌 저장소 중립적인 키를 저장해, 저장소 전환 시에도 기존 데이터를 다시 쓸
필요가 없도록 했습니다. 자세한 내용은 [minio.md](./minio.md) 참고.

---

# 서비스 간 참조 원칙

- **같은 DB 안**: 실제 FK를 겁니다. (예: `oauth_user.user_id → users.id`, `harvest.cultivation_id → cultivation.id`, `sensor.cultivation_id → cultivation.id`, `cultivation.mushroom_type → mushroom_reference.mushroom_type`)
- **다른 서비스의 DB**: FK 없는 순수 값(BIGINT/VARCHAR)으로만 참조하고, 존재/소유권 검증이
  필요하면 해당 서비스를 OpenFeign으로 호출해 확인합니다. (예: `cultivation.user_id`,
  `notification_endpoint.cultivation_id`)
- **같은 DB인데도 예외적으로 소프트 참조**: `inquiry.cultivation_id`는 같은 Cultivation DB
  안에 있지만, 재배 삭제 후에도 문의 기록을 보존해야 해서 의도적으로 FK를 걸지 않았습니다.
- **삭제 전파**: 같은 DB라면 `ON DELETE CASCADE`로 자동 처리되지만, 다른 DB 간에는
  이벤트(`CultivationDeletedEvent` 등)를 구독해 보상 삭제합니다.
