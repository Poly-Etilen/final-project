# Sensor Database

## 개요

Sensor Database는 센서 "장치"의 메타데이터와, 사용자가 저장한 목표 환경(위험 한계값) 이력을
관리하는 Sensor Service 소유의 PostgreSQL Database입니다.

> ℹ️ **변경 이력**: `sensor`, `environment_setting` 두 테이블은 원래 Cultivation DB에
> 있었습니다. 팀 회의 결과, 센서 장치 CRUD와 목표 환경(위험 한계값) 저장/조회가 본질적으로
> "센서" 도메인에 속하고, 이미 Sensor Service가 센서 측정값(InfluxDB/Redis)과 통계/차트/
> 리포트를 전담하고 있어 센서 관련 책임을 한 서비스로 모으는 것이 "database per service"
> 원칙에 더 맞는다고 판단해 Sensor Service로 이전했습니다. Cultivation Service는 이제
> `mushroom_reference`/`cultivation`/`harvest`/`photo`만 소유합니다. (자세한 내용은
> [cultivation-db.md](./cultivation-db.md), [README.md](../README.md)의 결정 사항 #24 참고)

두 테이블 모두 `cultivation_id`를 갖지만, 이제 Cultivation DB와는 별도의 데이터베이스이므로
**DB 레벨 외래키(FK)를 걸지 않습니다.** `cultivation_id`는 Cultivation Service의 `cultivation.id`를
가리키는 순수 참조값(정수)일 뿐이며, 존재/소유권 검증은 필요 시 Cultivation Service를
OpenFeign으로 호출해 확인합니다. (이 프로젝트에서 이미 `cultivation.user_id`가 Auth Service의
`users.id`를 FK 없이 참조하던 것과 같은 패턴입니다.)

기존에는 같은 DB 안에 있어 `cultivation` 삭제 시 `ON DELETE CASCADE`로 `sensor`/
`environment_setting`이 자동으로 함께 삭제되었지만, DB가 분리되며 이 자동 정리가 더 이상
불가능해졌습니다. 대신 Cultivation Service가 재배 삭제 시 `CultivationDeletedEvent`를
발행하고, Sensor Service가 이를 구독해 해당 `cultivation_id`의 `sensor`/`environment_setting`
행을 직접 삭제합니다(보상 정리, eventual consistency). (자세한 내용은 아래 "관계" 참고)

---

# ERD

```
sensor  (cultivation_id는 다른 DB의 cultivation.id를 참조하는 순수 값, FK 없음)
──────────────────────────────────────────────
PK  device_eui
    cultivation_id
    place
    location
    device_model
    sensor_type
    status
    created_at
    updated_at

environment_setting  (1:N — cultivation당 여러 row, 항목별 × 이력별, FK 없음)
──────────────────────────────────────────────
PK  id
    cultivation_id
    type
    min
    max
    unit
    created_at
    updated_at
```

---

# Table

## sensor

재배에 연결된 센서 장치의 메타데이터를 관리합니다. (기존 Cultivation DB에서 이전, 그 이전에는
DatasourceGenerator DB에 있었습니다.)

| Column | Type | Description |
|---------|------|-------------|
| device_eui | VARCHAR(32) | PK, 장치 고유 식별자 (LoRaWAN DevEUI, 16자리 hex 문자열) |
| cultivation_id | BIGINT | 재배 참조값 (다른 서비스 소유 데이터, FK 아님) |
| place | VARCHAR(50) | 설치 장소 (예: 1동 A구역) |
| location | VARCHAR(50) | 세부 위치 |
| device_model | VARCHAR(100) | 장치 모델명 |
| sensor_type | VARCHAR(30) | 센서 종류 |
| status | VARCHAR(20) | 상태 (ONLINE/OFFLINE/ERROR/MAINTENANCE), 시스템이 관리 (사용자 입력 아님) |
| created_at | TIMESTAMP | 생성일 |
| updated_at | TIMESTAMP | 수정일 |

`place`/`location`/`device_model`/`sensor_type`은 사용자가 센서 등록 시 직접 입력하는 값이며,
`status`는 등록 시 기본값(ONLINE)으로 시작해 이후 Rule Engine Service가 발행하는
`SensorErrorEvent`로만 갱신됩니다. `cultivation_id`는 요청 body가 아니라 URL 경로로부터
채워집니다.

---

## environment_setting

사용자가 최종 저장한 환경값을 **범위(min~max)** 로, 항목(type)별 행으로 저장합니다. (기존
Cultivation DB에서 이전)

단일 목표값이 아닌 범위로 저장하는 이유는 Rule Engine Service의 자동 제어가 값이 범위를
벗어날 때만 장치를 동작시키고, 범위 안에서는 불필요하게 켜고 끄지 않도록(허용 오차/
히스테리시스) 하기 위함입니다. `type`(TEMPERATURE/HUMIDITY/CO2/LIGHT)별로 별도 행을 가지며,
수정할 때도 UPDATE가 아니라 새 행을 INSERT합니다 — 이 테이블 하나가 "현재값"과 "이력"을
동시에 표현합니다. "현재값"이 필요하면 `(cultivation_id, type)` 기준으로 가장 최신 행을
조회합니다.

| Column | Type | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 재배 참조값 (다른 서비스 소유 데이터, FK 아님) |
| type | VARCHAR(20) | X | 환경 항목 (TEMPERATURE/HUMIDITY/CO2/LIGHT) |
| min | DECIMAL(4,1) | X | 하한 |
| max | DECIMAL(4,1) | X | 상한 |
| unit | VARCHAR(10) | O | 단위 (예: ℃, %, ppm, lux) |
| created_at | TIMESTAMP | X | 이 값이 저장된 시각 |
| updated_at | TIMESTAMP | X | 생성 시각과 동일하게 유지됨 (UPDATE 경로 없음) |

---

# DDL

## sensor

```sql
CREATE TABLE sensor (

    device_eui VARCHAR(32) NOT NULL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    place VARCHAR(50),

    location VARCHAR(50),

    device_model VARCHAR(100),

    sensor_type VARCHAR(30),

    status VARCHAR(20) NOT NULL DEFAULT 'ONLINE',

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP

);
```

`cultivation_id`에는 FK를 걸지 않습니다(다른 서비스/DB의 데이터). 대신
`idx_sensor_cultivation` 인덱스로 조회 성능을 확보합니다.

---

## environment_setting

```sql
CREATE TABLE environment_setting (

    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    type VARCHAR(20) NOT NULL,

    min DECIMAL(4,1) NOT NULL,

    max DECIMAL(4,1) NOT NULL,

    unit VARCHAR(10),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_environment_type CHECK (type IN (
        'TEMPERATURE', 'HUMIDITY', 'CO2', 'LIGHT'
    ))

);
```

Cultivation DB에 있던 `fk_environment_cultivation FOREIGN KEY ... REFERENCES cultivation(id)
ON DELETE CASCADE` 제약은 유지할 수 없습니다(교차 서비스). CASCADE 삭제는 아래 "관계"의
`CultivationDeletedEvent` 구독으로 대체합니다.

---

# Index

## sensor

```sql
CREATE INDEX idx_sensor_cultivation
ON sensor(cultivation_id);
```

```sql
CREATE INDEX idx_sensor_status
ON sensor(status);
```

device_eui가 PK이므로 별도 UNIQUE 인덱스는 필요하지 않습니다.

---

## environment_setting

```sql
CREATE INDEX idx_environment_setting_cultivation_type
ON environment_setting(cultivation_id, type, created_at DESC);
```

"재배의 각 항목별 현재값(최신 행)"을 조회하는 것이 가장 흔한 접근 패턴이라, `cultivation_id`
+ `type` + `created_at DESC` 복합 인덱스 하나로 커버합니다.

```sql
SELECT DISTINCT ON (type) *
FROM environment_setting
WHERE cultivation_id = ?
ORDER BY type, created_at DESC;
```

---

# 상태(Status)

## sensor

| 값 | 설명 |
|-----|------|
| ONLINE | 정상 |
| OFFLINE | 연결 끊김 |
| ERROR | 오류 |
| MAINTENANCE | 점검 중 |

`sensor.status`는 Rule Engine Service가 발행하는 `SensorErrorEvent`를 Sensor Service가
구독해 갱신합니다. (기존에는 Cultivation Service가 구독했지만, `sensor` 테이블 소유권 이전과
함께 구독 주체도 옮겨졌습니다.)

---

# 데이터 생성 흐름

## ① 센서 등록 (단건)

Client가 재배 생성 이후 개별적으로 센서를 추가할 때 직접 Sensor Service를 호출합니다.

```
Client → Sensor Service

↓

cultivation_id 소유권 검증 (Cultivation Service OpenFeign 호출, GET /api/v1/cultivations/{cultivationId}/owner)

↓

sensor 생성

↓

RabbitMQ Publish (SensorRegisteredEvent)
```

---

## ② 센서 일괄 등록 (재배 생성과 함께, 내부용)

Cultivation Service가 재배 생성 직후 `devices`를 한 번에 등록하기 위해 호출하는 내부용
흐름입니다. 이 경우 소유권 검증은 생략합니다(호출 주체 자체가 이미 그 재배를 막 생성한
Cultivation Service이므로).

```
Cultivation Service → Sensor Service (POST /api/v1/sensors/cultivations/{cultivationId}/batch)

↓

devices 각 항목으로 sensor 레코드 생성 (Sensor DB 자체 트랜잭션, all-or-nothing)

↓

RabbitMQ Publish (SensorRegisteredEvent, 디바이스별)

↓

Cultivation Service에 등록 결과 반환
```

Cultivation Service와 Sensor Service에 걸친 진짜 분산 트랜잭션은 아닙니다. 이 배치 등록이
실패하면 Cultivation Service가 방금 생성한 cultivation을 보상 삭제(compensating delete)해
사용자 관점에서는 여전히 all-or-nothing처럼 보이도록 합니다. (자세한 내용은
[cultivation-api.md](../02_API/cultivation-api.md)의 "재배 생성" 참고)

---

## ③ 환경 저장

```
Client → Sensor Service (PATCH /api/v1/sensors/cultivations/{cultivationId}/environment)

↓

cultivation_id 소유권 검증 (Cultivation Service OpenFeign 호출)

↓

수정된 항목마다 단일 목표값을 허용 오차만큼 확장하여 범위로 변환

↓

environment_setting에 항목별로 새 행 INSERT (수정되지 않은 항목은 기존 최신 행 유지)

↓

RabbitMQ Publish (EnvironmentRangeUpdatedEvent) → Rule Engine Service
```

단일값 → 범위 변환 기준(허용 오차)과 범위 → 단일값 역변환(중간값) 로직은 기존 Cultivation
Service에 있던 것을 그대로 Sensor Service로 옮겼습니다. (자세한 내용은
[sensor.md](../01_Domain/sensor.md) 참고)

---

## ④ 재배 삭제에 따른 정리 (보상 삭제)

```
Cultivation Service

↓

재배 삭제 (cultivation 행 삭제, PostgreSQL 자체 CASCADE로 harvest/photo도 함께 삭제됨)

↓

RabbitMQ Publish (CultivationDeletedEvent)

↓

Sensor Service 구독

↓

sensor / environment_setting에서 해당 cultivation_id 행 전체 삭제
```

같은 DB 안에서의 `ON DELETE CASCADE`를 더 이상 쓸 수 없어 생긴 대체 경로입니다. 이벤트가
유실되면 정리가 지연될 수 있습니다(추후 개발 예정: 주기적 재동기화 배치).

---

# 관계

```
Cultivation Service (cultivation, 원본)

    │  GET /api/v1/cultivations/{cultivationId}/owner (OpenFeign, 쓰기 요청마다 소유권 검증)
    │  CultivationDeletedEvent (RabbitMQ, 재배 삭제 시 정리용)

    ▼

Sensor Service (sensor, environment_setting)

    │  SensorRegisteredEvent / SensorDeletedEvent (RabbitMQ)

    ▼

DatasourceGenerator (sensor_cache, 메모리, 읽기 전용)


Sensor Service (environment_setting)

    │  EnvironmentRangeUpdatedEvent (RabbitMQ)

    ▼

Rule Engine Service (Redis 캐시 write-through)


Sensor Service (sensor.status)

    ▲

    │  SensorErrorEvent (RabbitMQ, Rule Engine Service 발행)
```

---

# 고려 사항

- `sensor`/`environment_setting`은 원래 Cultivation DB에 있었지만, 센서 관련 책임(측정값 저장/조회 + 장치 메타데이터 + 목표 환경)을 Sensor Service 하나로 모으기 위해 이전했습니다.
- `cultivation_id`는 두 테이블 모두에서 다른 서비스(Cultivation Service)의 데이터를 가리키는 순수 참조값이며, DB 레벨 FK를 걸지 않습니다. 이는 이 프로젝트에서 이미 `cultivation.user_id`(Auth Service 참조)가 따르던 패턴과 동일합니다.
- 같은 DB 안에 있을 때 가능했던 `ON DELETE CASCADE`는 더 이상 불가능하며, `CultivationDeletedEvent` 구독을 통한 보상 삭제로 대체했습니다. 이벤트 기반이라 즉시성은 CASCADE보다 약간 떨어질 수 있습니다.
- 쓰기 작업(센서 등록/삭제, 환경 저장)마다 Cultivation Service에 소유권을 확인하는 OpenFeign 호출이 추가로 필요합니다. 다만 "재배 생성과 함께 센서 일괄 등록"하는 경우는 호출 주체가 Cultivation Service 자신이므로 이 검증을 생략합니다.
- environment_setting은 단일 목표값이 아닌 범위(min~max)로 저장합니다. Rule Engine Service가 범위를 벗어날 때만 장치를 제어하도록 하여 불필요한 On/Off를 줄이기 위함입니다.
- Environment Setting은 항목(TEMPERATURE/HUMIDITY/CO2/LIGHT)별로 여러 행이 쌓이며, UPDATE 없이 항상 INSERT만 합니다. 데이터가 무한히 쌓이는 이력성 테이블이라, 운영 단계에서는 오래된 이력에 대한 보관 주기 정책이 필요할 수 있습니다(추후 개발 예정).
- sensor.device_eui는 대리키가 아닌 사용자가 입력하는 장치 고유 식별자를 그대로 PK로 사용합니다.
- 별도 datasource 테이블/엔티티는 존재하지 않습니다. 위치 정보(place/location)는 센서 레코드에 직접 저장합니다.
- 센서가 측정한 "값"(시계열)은 이 DB가 아닌 InfluxDB에서 관리합니다. 이 DB는 장치 메타데이터와 목표 환경(위험 한계값)만 다룹니다.
