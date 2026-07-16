# DatasourceGenerator Database

## 개요

DatasourceGenerator Database는 데이터 소스를 관리하고, 센서 데이터 생성/발행에 필요한
최소한의 센서 정보 캐시를 보관합니다. (기존 명칭: Datasource Database)

실제 센서 데이터는 저장하지 않으며, 센서가 측정한 값은 MQTT를 통해 Rule Engine Service로
전달됩니다.

> ℹ️ **변경 이력**: 센서 "장치"의 등록/조회/삭제(CRUD)는 원래 이 DB의 `sensor` 테이블이
> 전담했지만, 센서가 항상 특정 재배에 종속되는 정보라는 점 때문에 소유권을 Cultivation
> Service(Cultivation DB)로 옮겼습니다. 이 DB는 더 이상 센서 메타데이터의 원본(source of
> truth)이 아니며, Cultivation Service가 발행하는 `SensorRegisteredEvent`/`SensorDeletedEvent`를
> 구독해 "어떤 센서에 대해 데이터를 생성/발행해야 하는지" 판단하는 데 필요한 최소 정보만
> `sensor_cache` 테이블에 보관합니다. (자세한 내용은 [cultivation-db.md](./cultivation-db.md) 참고)

---

# ERD

```
datasource
──────────────────────────────────────────────
PK  id
    name
    type
    location
    description
    status
    created_at
    updated_at

          │ 1
          │
          │
          ▼
sensor_cache (Cultivation Service 이벤트로 채워지는 읽기 전용 캐시, source of truth 아님)
──────────────────────────────────────────────
PK  sensor_id (Cultivation Service의 sensor.id와 동일)
FK  datasource_id
    cultivation_id
    sensor_type
    created_at
    updated_at
```

---

# Table

## datasource

센서가 연결되는 데이터 소스를 관리합니다.

| Column | Type | Description |
|---------|------|-------------|
| id | BIGSERIAL | PK |
| name | VARCHAR(100) | 데이터 소스 이름 |
| type | VARCHAR(30) | 데이터 소스 타입 |
| location | VARCHAR(255) | 설치 위치 |
| description | TEXT | 설명 |
| status | VARCHAR(20) | 상태 |
| created_at | TIMESTAMP | 생성일 |
| updated_at | TIMESTAMP | 수정일 |

---

## sensor_cache

DatasourceGenerator가 "어떤 센서에 대해 MQTT 데이터를 생성/발행해야 하는지" 알기 위한
읽기 전용 캐시입니다. Cultivation Service의 `sensor` 테이블이 원본이며, 이 테이블은
`SensorRegisteredEvent`(삽입)/`SensorDeletedEvent`(삭제)로만 갱신됩니다. 이름, 상태(ONLINE
등), 설치일 같은 상세 메타데이터는 갖지 않습니다. (필요하면 Cultivation Service API를 조회)

| Column | Type | Description |
|---------|------|-------------|
| sensor_id | BIGINT | PK, Cultivation Service의 sensor.id와 동일한 값 |
| datasource_id | BIGINT | 데이터 소스 (같은 DB 내 실제 FK) |
| cultivation_id | BIGINT | 재배 (Cultivation DB에 대한 소프트 참조) |
| sensor_type | VARCHAR(30) | 센서 종류 (시뮬레이션 데이터 생성에 사용) |
| created_at | TIMESTAMP | 캐시 생성일 |
| updated_at | TIMESTAMP | 캐시 갱신일 |

---

# DDL

## datasource

```sql
CREATE TABLE datasource (

    id BIGSERIAL PRIMARY KEY,

    name VARCHAR(100) NOT NULL,

    type VARCHAR(30) NOT NULL,

    location VARCHAR(255),

    description TEXT,

    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP

);
```

---

## sensor_cache

```sql
CREATE TABLE sensor_cache (

    sensor_id BIGINT PRIMARY KEY,

    datasource_id BIGINT NOT NULL,

    cultivation_id BIGINT NOT NULL,

    sensor_type VARCHAR(30) NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_sensor_cache_datasource
        FOREIGN KEY (datasource_id)
        REFERENCES datasource(id)
        ON DELETE CASCADE

);
```

`sensor_id`는 BIGSERIAL이 아니라 Cultivation Service가 이벤트로 알려주는 값을 그대로 사용합니다
(자체 채번하지 않음). `cultivation_id`는 다른 서비스의 DB를 가리키는 소프트 참조이므로 FK 제약이 없습니다.

---

# Index

## datasource

```sql
CREATE INDEX idx_datasource_type
ON datasource(type);
```

---

## sensor_cache

```sql
CREATE INDEX idx_sensor_cache_cultivation
ON sensor_cache(cultivation_id);
```

---

# Status

## datasource

| 값 | 설명 |
|-----|------|
| ACTIVE | 사용 중 |
| INACTIVE | 비활성 |
| ERROR | 오류 |

sensor_cache는 상태(ONLINE/OFFLINE/ERROR 등)를 갖지 않습니다. 센서 상태는 이제 Cultivation
Service의 `sensor.status`가 원본이며, Rule Engine Service가 발행하는 SensorErrorEvent도
Cultivation Service가 구독합니다. DatasourceGenerator는 상태 정보가 필요 없습니다(데이터
생성/발행 여부는 sensor_cache에 존재하는지 여부로만 판단).

---

# 데이터 흐름

센서 등록 (Cultivation Service)

↓

RabbitMQ Publish (SensorRegisteredEvent)

↓

DatasourceGenerator 구독 → sensor_cache Upsert

↓

MQTT Publish (sensor_cache에 있는 센서만 데이터 생성/발행)

↓

Rule Engine Service

삭제 시에는 SensorDeletedEvent를 구독해 sensor_cache에서 해당 행을 삭제합니다.

---

# 관계

```
Cultivation Service (sensor, 원본)

    │  SensorRegisteredEvent / SensorDeletedEvent (RabbitMQ)

    ▼

DatasourceGenerator (sensor_cache, 읽기 전용 캐시)

    │

    ▼

Datasource (같은 DB, 실제 FK)

    │

    ▼

MQTT → Rule Engine Service
```

---

# 고려 사항

- 실제 센서 데이터(측정값)는 PostgreSQL에 저장하지 않습니다.
- 센서 "장치" 메타데이터의 원본(source of truth)은 더 이상 이 DB가 아니라 Cultivation Service의 `sensor` 테이블입니다. 이 DB는 시뮬레이션/발행에 필요한 최소 정보만 이벤트로 전달받아 `sensor_cache`에 보관합니다.
- sensor_cache가 비어있거나 이벤트 유실로 최신 상태가 아니면, 해당 센서에 대한 데이터 생성/발행이 누락될 수 있습니다. 이벤트만으로 동기화하는 구조라 재동기화(reconciliation) 수단은 아직 없으며, 향후 필요 시 Cultivation Service를 OpenFeign으로 호출해 전체 목록을 다시 받아오는 배치를 추가할 수 있습니다.
- 센서값은 MQTT를 통해 Rule Engine Service로 전달됩니다.
- 시계열 데이터는 Sensor Service(InfluxDB)에서 관리합니다. Rule Engine Service는 수신·검증·규칙평가만 담당하고 RabbitMQ로 Sensor Service에 저장을 위임합니다.
- 하나의 Datasource에는 여러 개의 sensor_cache 행(센서)이 연결될 수 있습니다.
