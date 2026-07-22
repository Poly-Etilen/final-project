# Sensor Database

## 개요

Sensor Database는 센서 "장치"의 메타데이터, 사용자가 저장한 목표 환경(위험 한계값) 이력,
공공데이터 기반 버섯 참조 데이터를 관리하는 Sensor Service 소유의 PostgreSQL Database입니다.
측정값(InfluxDB/Redis)과 통계/리포트를 이미 전담하고 있는 Sensor Service가 장치 메타데이터와
목표 환경까지 함께 소유하는 것이 "database per service" 원칙에 맞다고 판단해 이 DB로
모았습니다.

온도/습도/CO₂/조도라는 같은 측정 항목 도메인이 `sensor_type`(장치가 실제 측정하는 항목),
`environment_setting`(사용자가 설정한 목표 범위), `mushroom_reference_threshold`(버섯별
추천 범위) 세 곳에서 공통으로 쓰이기 때문에, 이 값을 매번 자유 문자열로 반복하지 않고
`measurement_type` 참조 테이블로 한 번만 정의합니다. 세 테이블은 이 테이블을 같은 DB 내
실제 FK로 참조합니다.

---

# ERD

```
measurement_type (전역 참조, 4종 고정 시드 데이터)
──────────────────────────────────────────────
PK  code
    name_ko
    unit


sensor  (cultivation_id는 다른 DB의 cultivation.id를 참조하는 순수 값, FK 없음)
──────────────────────────────────────────────
PK  id
    cultivation_id
    device_eui       (UNIQUE)
    location
    location_detail
    device_model
    status
    is_deleted
    created_at

          │ 1
          │
          │
          ▼
sensor_type  (기기 하나가 여러 측정 항목을 가질 수 있음)
──────────────────────────────────────────────
PK  id
FK  sensor_id
FK  measurement_type_code
    UNIQUE(sensor_id, measurement_type_code)


environment_setting  (1:N — cultivation당 여러 row, 항목별 × 이력별)
──────────────────────────────────────────────
PK  id
    cultivation_id
FK  measurement_type_code
    threshold_min
    threshold_max
    threshold_unit
    created_at


mushroom_reference (전역 참조 테이블, cultivation.mushroom_type 코드와 매칭되는 자연키)
──────────────────────────────────────────────
PK  id
    mushroom_type     (UNIQUE — cultivation.mushroom_type이 참조하는 코드)
    mushroom_name_ko
    mushroom_name_en
    mushroom_scientific_name
    characteristics
    health_benefits
    cultivation_guide
    additional_info
    created_at
    updated_at

          │ 1
          │
          │
          ▼
mushroom_reference_threshold  (항목별 여러 row)
──────────────────────────────────────────────
PK  id
FK  mushroom_reference_id
FK  measurement_type_code
    threshold_min
    threshold_max
    threshold_unit
    created_at
    updated_at

    UNIQUE(mushroom_reference_id, measurement_type_code)
```

`sensor`/`environment_setting`은 `cultivation_id`를 갖지만 Cultivation DB와는 별도의
데이터베이스이므로 DB 레벨 FK를 걸지 않습니다. 존재/소유권 검증은 필요 시 Cultivation
Service를 OpenFeign으로 호출해 확인합니다.

---

# Table

## measurement_type

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| code | VARCHAR(20) | X | PK (자연키, 예: TEMPERATURE/HUMIDITY/CO2/LIGHT) |
| name_ko | VARCHAR(20) | X | 표시용 한글 이름 (예: 온도) |
| unit | VARCHAR(10) | X | 기본 단위 (예: ℃, %, ppm, lux) |

측정 항목 4종의 고정 시드 데이터입니다. 새 측정 항목이 추가될 때만 여기에 행이 늘어나며,
운영 중 자주 바뀌지 않습니다.

---

## sensor

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 재배 참조값 (Cultivation Service, 소프트 참조) |
| device_eui | VARCHAR(32) | X | 장치 고유 식별자 (LoRaWAN DevEUI), UNIQUE |
| location | VARCHAR(10) | O | 설치 위치 (짧은 코드, 예: A동) |
| location_detail | VARCHAR(50) | O | 세부 위치 |
| device_model | VARCHAR(100) | O | 장치 모델명 |
| status | VARCHAR(10) | X | 상태 (ONLINE/OFFLINE/ERROR), 기본값 OFFLINE, 시스템이 관리 |
| is_deleted | BOOLEAN | X | 소프트 삭제 플래그, 기본값 FALSE |
| created_at | DATETIME | X | 등록 일시 |

PK는 대리키 `id`이고 `device_eui`는 UNIQUE 제약을 가진 일반 컬럼입니다. 삭제는 하드
삭제가 아니라 `is_deleted` 플래그로 처리합니다 — 삭제된 센서라도 과거
`environment_setting` 이력이 계속 의미를 가질 수 있게 하기 위함입니다.

---

## sensor_type

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| sensor_id | BIGINT | X | FK, `sensor.id` |
| measurement_type_code | VARCHAR(20) | X | FK, `measurement_type.code` |

장치 하나가 여러 항목을 동시에 측정할 수 있어 `sensor`와 1:N 관계입니다.
`UNIQUE(sensor_id, measurement_type_code)`로 같은 장치에 같은 항목이 중복 등록되지 않게
합니다.

---

## environment_setting

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 재배 참조값 (소프트 참조) |
| measurement_type_code | VARCHAR(20) | X | FK, `measurement_type.code` |
| threshold_min | NUMERIC(10,4) | X | 하한 |
| threshold_max | NUMERIC(10,4) | X | 상한 |
| threshold_unit | VARCHAR(10) | X | 단위 |
| created_at | DATETIME | X | 이 값이 저장된 시각 |
| updated_at | DATETIME | O | 생성 시각과 동일하게 유지됨 (UPDATE 경로 없음) |

사용자가 최종 저장한 환경값을 **범위**로, 항목별 행으로 저장합니다. 수정할 때도 UPDATE가
아니라 새 행을 INSERT합니다 — 이 테이블 하나가 "현재값"과 "이력"을 동시에 표현합니다.
"현재값"은 `(cultivation_id, measurement_type_code)` 기준 최신 행으로 조회합니다.

---

## mushroom_reference

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| mushroom_type | VARCHAR(20) | X | 버섯 종류 코드 (예: OYSTER), UNIQUE. `cultivation.mushroom_type`이 참조 |
| mushroom_name_ko | VARCHAR(50) | X | 버섯 한글 이름 |
| mushroom_name_en | VARCHAR(50) | X | 버섯 영문 이름 |
| mushroom_scientific_name | VARCHAR(50) | O | 학명 |
| characteristics | TEXT | O | 특성 설명 |
| health_benefits | TEXT | O | 효능 |
| cultivation_guide | TEXT | O | 재배 가이드 |
| additional_info | TEXT | O | 추가 정보 |
| created_at | DATETIME | X | 생성일 |
| updated_at | DATETIME | O | 수정일 |

공공데이터 기반 5종 버섯의 고정 시드 데이터입니다. `mushroom_type`이 `cultivation.mushroom_type`과
연결되는 안정적인 코드 역할을 하고, `characteristics`/`health_benefits`/`cultivation_guide`/
`additional_info`는 AI Service의 챗봇/버섯 가이드 기능이 `mushroomType`으로 정확히
일치하는 한 건을 그대로 조회해 LLM 컨텍스트(RAG 원문)로 사용하는 텍스트입니다. 버섯
종류가 5종 고정이라 별도의 임베딩·검색 색인 없이 직접 조회로 충분합니다.

---

## mushroom_reference_threshold

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| mushroom_reference_id | BIGINT | X | FK, `mushroom_reference.id` |
| measurement_type_code | VARCHAR(20) | X | FK, `measurement_type.code` |
| threshold_min | NUMERIC(10,4) | X | 하한 |
| threshold_max | NUMERIC(10,4) | X | 상한 |
| threshold_unit | VARCHAR(10) | X | 단위 |
| created_at | DATETIME | X | 생성일 |
| updated_at | DATETIME | O | 수정일 |

버섯 종류별 최적 생육 환경 범위를 항목별 행으로 저장합니다.
`UNIQUE(mushroom_reference_id, measurement_type_code)`로 같은 버섯의 같은 항목이 중복
등록되지 않게 합니다.

---

# DDL

```sql
CREATE TABLE measurement_type (
    code VARCHAR(20) PRIMARY KEY,

    name_ko VARCHAR(20) NOT NULL,

    unit VARCHAR(10) NOT NULL
);
```

```sql
CREATE TABLE sensor (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    device_eui VARCHAR(32) NOT NULL,

    location VARCHAR(10),

    location_detail VARCHAR(50),

    device_model VARCHAR(100),

    status VARCHAR(10) NOT NULL DEFAULT 'OFFLINE',

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uk_sensor_device_eui UNIQUE (device_eui),

    CONSTRAINT chk_sensor_status CHECK (status IN ('ONLINE', 'OFFLINE', 'ERROR'))
);
```

```sql
CREATE TABLE sensor_type (
    id BIGSERIAL PRIMARY KEY,

    sensor_id BIGINT NOT NULL,

    measurement_type_code VARCHAR(20) NOT NULL,

    CONSTRAINT fk_sensor_type_sensor
        FOREIGN KEY (sensor_id) REFERENCES sensor(id),

    CONSTRAINT fk_sensor_type_measurement
        FOREIGN KEY (measurement_type_code) REFERENCES measurement_type(code),

    CONSTRAINT uk_sensor_type UNIQUE (sensor_id, measurement_type_code)
);
```

```sql
CREATE TABLE environment_setting (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    measurement_type_code VARCHAR(20) NOT NULL,

    threshold_min NUMERIC(10,4) NOT NULL,

    threshold_max NUMERIC(10,4) NOT NULL,

    threshold_unit VARCHAR(10) NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP,

    CONSTRAINT fk_environment_setting_measurement
        FOREIGN KEY (measurement_type_code) REFERENCES measurement_type(code)
);
```

```sql
CREATE TABLE mushroom_reference (
    id BIGSERIAL PRIMARY KEY,

    mushroom_type VARCHAR(20) NOT NULL,

    mushroom_name_ko VARCHAR(50) NOT NULL,

    mushroom_name_en VARCHAR(50) NOT NULL,

    mushroom_scientific_name VARCHAR(50),

    characteristics TEXT,

    health_benefits TEXT,

    cultivation_guide TEXT,

    additional_info TEXT,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP,

    CONSTRAINT uk_mushroom_reference_type UNIQUE (mushroom_type)
);
```

```sql
CREATE TABLE mushroom_reference_threshold (
    id BIGSERIAL PRIMARY KEY,

    mushroom_reference_id BIGINT NOT NULL,

    measurement_type_code VARCHAR(20) NOT NULL,

    threshold_min NUMERIC(10,4) NOT NULL,

    threshold_max NUMERIC(10,4) NOT NULL,

    threshold_unit VARCHAR(10) NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP,

    CONSTRAINT fk_mushroom_reference_threshold_reference
        FOREIGN KEY (mushroom_reference_id) REFERENCES mushroom_reference(id),

    CONSTRAINT fk_mushroom_reference_threshold_measurement
        FOREIGN KEY (measurement_type_code) REFERENCES measurement_type(code),

    CONSTRAINT uk_mushroom_reference_threshold UNIQUE (mushroom_reference_id, measurement_type_code)
);
```

---

# Index

```sql
CREATE INDEX idx_sensor_cultivation
ON sensor(cultivation_id)
WHERE is_deleted = FALSE;
```

재배별 활성 센서 목록 조회에 사용하는 부분 인덱스입니다.

```sql
CREATE INDEX idx_environment_setting_lookup
ON environment_setting(cultivation_id, measurement_type_code, created_at DESC);
```

"현재값" 조회(재배+항목 기준 최신 행)와 이력 조회 모두에 사용합니다.

```sql
CREATE INDEX idx_mushroom_reference_threshold_reference
ON mushroom_reference_threshold(mushroom_reference_id);
```

버섯 하나의 전체 임계값 조회에 사용합니다.

---

# 관계

```
sensor_type, environment_setting  (Sensor Service)

    │ measurement_type_code (같은 DB 내 실제 FK)
    ▼
measurement_type


mushroom_reference_threshold (Sensor Service)

    │ mushroom_reference_id, measurement_type_code (같은 DB 내 실제 FK)
    ▼
mushroom_reference, measurement_type


sensor, environment_setting (Sensor Service)

    │ cultivation_id (소프트 참조)
    ▼
Cultivation Service
```

Sensor Service는 다른 서비스와 데이터베이스를 공유하지 않습니다. `cultivation_id`를 통해서만
Cultivation Service를 간접 참조합니다. `mushroom_reference`/`measurement_type`은 다른
서비스 데이터를 참조하지 않는 전역 시드 데이터입니다.

---

# 다른 서비스와의 연관

`cultivation` 삭제 시 같은 DB 안이었다면 `ON DELETE CASCADE`로 자동 정리됐겠지만, DB가
분리되어 있어 Cultivation Service가 발행하는 `CultivationDeletedEvent`를 Sensor Service가
구독해 해당 `cultivation_id`의 `sensor`/`environment_setting` 행을 직접 삭제합니다(보상
정리).

---

# 고려 사항

- `measurement_type`을 별도 참조 테이블로 뺀 이유는 온도/습도/CO₂/조도라는 같은 값 집합이
  `sensor_type`/`environment_setting`/`mushroom_reference_threshold` 세 곳에서 자유
  문자열로 반복되면 값이 서로 어긋날 위험(예: 한쪽은 "온도", 다른 쪽은 "TEMPERATURE")이
  있기 때문입니다. 이제 새 측정 항목을 추가하려면 `measurement_type`에 행 하나만 추가하면
  됩니다.
- `cultivation.mushroom_type` ↔ `mushroom_reference.mushroom_type`은 둘 다 같은 코드
  문자열(예: `OYSTER`)을 쓰는 것으로 연결 방식을 확정했습니다. 버섯 종류가 공공데이터
  기준 5종으로 고정되어 있어, 코드가 곧 자연키 역할을 합니다.
- "재배 생성 시 환경 추천"은 Cultivation Service가 이 `mushroom_type` 코드로 Sensor
  Service를 OpenFeign 호출해 `mushroom_reference`/`mushroom_reference_threshold`를
  조회하는 방식으로 처리합니다.
