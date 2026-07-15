# DatasourceGenerator Database

## 개요

DatasourceGenerator Database는 센서 및 데이터 소스를 관리합니다. (기존 명칭: Datasource Database)

실제 센서 데이터는 저장하지 않으며,
센서의 메타데이터와 데이터 소스 정보를 관리합니다.

센서에서 측정된 데이터는 MQTT를 통해 Rule Engine Service로 전달됩니다.

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
sensor
──────────────────────────────────────────────
PK  id
FK  datasource_id
FK  cultivation_id
    sensor_uuid
    sensor_type
    name
    status
    installed_at
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

## sensor

센서 정보를 관리합니다.

| Column | Type | Description |
|---------|------|-------------|
| id | BIGSERIAL | PK |
| datasource_id | BIGINT | 데이터 소스 |
| cultivation_id | BIGINT | 재배 정보 |
| sensor_uuid | UUID | 센서 고유 ID |
| sensor_type | VARCHAR(30) | 센서 종류 |
| name | VARCHAR(100) | 센서 이름 |
| status | VARCHAR(20) | 상태 |
| installed_at | TIMESTAMP | 설치일 |
| created_at | TIMESTAMP | 생성일 |
| updated_at | TIMESTAMP | 수정일 |

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

## sensor

```sql
CREATE TABLE sensor (

    id BIGSERIAL PRIMARY KEY,

    datasource_id BIGINT NOT NULL,

    cultivation_id BIGINT NOT NULL,

    sensor_uuid UUID NOT NULL UNIQUE,

    sensor_type VARCHAR(30) NOT NULL,

    name VARCHAR(100) NOT NULL,

    status VARCHAR(20) NOT NULL DEFAULT 'ONLINE',

    installed_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_sensor_datasource
        FOREIGN KEY (datasource_id)
        REFERENCES datasource(id)
        ON DELETE CASCADE

);
```

---

# Index

## datasource

```sql
CREATE INDEX idx_datasource_type
ON datasource(type);
```

---

## sensor

```sql
CREATE UNIQUE INDEX uk_sensor_uuid
ON sensor(sensor_uuid);
```

```sql
CREATE INDEX idx_sensor_cultivation
ON sensor(cultivation_id);
```

```sql
CREATE INDEX idx_sensor_status
ON sensor(status);
```

---

# Status

## datasource

| 값 | 설명 |
|-----|------|
| ACTIVE | 사용 중 |
| INACTIVE | 비활성 |
| ERROR | 오류 |

---

## sensor

| 값 | 설명 |
|-----|------|
| ONLINE | 정상 |
| OFFLINE | 연결 끊김 |
| ERROR | 오류 |
| MAINTENANCE | 점검 중 |

sensor.status는 Rule Engine Service가 발행하는 SensorErrorEvent를 구독해 갱신합니다.

---

# 데이터 흐름

센서 등록

↓

Datasource 생성

↓

Sensor 생성

↓

MQTT Publish

↓

Rule Engine Service

---

# 관계

```
Datasource

    │

    ▼

Sensor

    │

    ▼

Cultivation (ID 참조)

    │

    ▼

MQTT → Rule Engine Service
```

---

# 고려 사항

- 실제 센서 데이터는 PostgreSQL에 저장하지 않습니다.
- 센서의 메타데이터만 저장합니다.
- 센서값은 MQTT를 통해 Rule Engine Service로 전달됩니다.
- 시계열 데이터는 Sensor Service(InfluxDB)에서 관리합니다. Rule Engine Service는 수신·검증·규칙평가만 담당하고 RabbitMQ로 Sensor Service에 저장을 위임합니다.
- 하나의 Datasource에는 여러 개의 Sensor가 연결될 수 있습니다.
