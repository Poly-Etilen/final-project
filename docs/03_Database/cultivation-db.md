# Cultivation Database

## 개요

Cultivation Database는 사용자의 버섯 재배 정보를 관리합니다.

하나의 Cultivation은 하나의 재배를 의미하며,
재배 생성 시 추천되는 환경값은 별도로 저장되지 않습니다.

사용자가 추천값을 수정하거나 그대로 적용하여 저장 버튼을 눌렀을 때만
Environment Setting이 생성됩니다. 이때 API로 주고받는 단일 목표값은 Cultivation Service에 의해
범위(min~max)로 변환되어 저장됩니다.

재배 종료 후에는 Harvest 정보를 저장합니다.

> ℹ️ **변경 이력**: 재배 생성 시 보여주는 추천 환경값은 AI Service(Embedding/Vector Search/LLM)를
> 호출해 생성하던 방식에서, 공공데이터 기반 5종 버섯의 최적 범위를 담은 `mushroom_reference`
> 참조 테이블을 Cultivation Service가 직접 조회하는 방식으로 변경했습니다. 버섯 종류가 5가지로
> 고정되어 있어 매번 동일한 값이 나오는 조회에는 벡터 검색/LLM이 필요하지 않다고 판단했습니다.
> `mushroom_reference`는 사용자별 데이터가 아닌 전역 고정 시드 데이터이며, `environment_setting`
> (사용자가 재배별로 직접 설정하는 위험 한계값)과는 별개의 테이블입니다.

---

# ERD

```
mushroom_reference (전역 참조 테이블, cultivation과 FK 관계 없음)
──────────────────────────────────────────────
PK  mushroom_type
    temp_min
    temp_max
    humidity_min
    humidity_max
    co2_min
    co2_max
    light_min
    light_max
    description
    created_at
    updated_at

cultivation
──────────────────────────────────────────────
PK  id
    user_id
    name
    mushroom_type
    status
    started_at
    finished_at
    created_at
    updated_at

          │ 1
          │
          │
          ▼
environment_setting
──────────────────────────────────────────────
PK  id
FK  cultivation_id
    temp_min
    temp_max
    humidity_min
    humidity_max
    co2_min
    co2_max
    light_min
    light_max
    created_at
    updated_at

          │ 1
          │
          │
          ▼
harvest
──────────────────────────────────────────────
PK  id
FK  cultivation_id
    harvest_weight
    memo
    harvested_at

          │ 1
          │
          │
          ▼
photo
──────────────────────────────────────────────
PK  id
FK  cultivation_id
    image_url
    uploaded_at
    created_at
```

---

# Table

## mushroom_reference

공공데이터 기반으로 버섯 종류별 최적 생육 환경 **범위**를 담은 전역 참조 테이블입니다.

특정 cultivation에 속하지 않는 고정 시드 데이터이며, 관리자가 데이터를 갱신하기 전까지는
변하지 않습니다. 재배 생성 시 Cultivation Service가 이 테이블을 직접 조회하여 추천값으로
보여주며, 이 값 자체는 environment_setting에 저장되지 않습니다.

| Column | Type | Description |
|---------|------|-------------|
| mushroom_type | VARCHAR(50) | PK, 버섯 종류 |
| temp_min | DECIMAL(4,1) | 최적 온도 하한 |
| temp_max | DECIMAL(4,1) | 최적 온도 상한 |
| humidity_min | DECIMAL(4,1) | 최적 습도 하한 |
| humidity_max | DECIMAL(4,1) | 최적 습도 상한 |
| co2_min | INT | 최적 CO₂ 하한 |
| co2_max | INT | 최적 CO₂ 상한 |
| light_min | INT | 최적 조도 하한 |
| light_max | INT | 최적 조도 상한 |
| description | VARCHAR(500) | 버섯별 생육 특성 설명 |
| created_at | TIMESTAMP | 생성일 |
| updated_at | TIMESTAMP | 수정일 |

---

## cultivation

재배 정보를 저장합니다.

| Column | Type | Description |
|---------|------|-------------|
| id | BIGSERIAL | PK |
| user_id | BIGINT | 사용자 |
| name | VARCHAR(100) | 재배 이름 |
| mushroom_type | VARCHAR(50) | 버섯 종류 |
| status | VARCHAR(20) | 재배 상태 |
| started_at | TIMESTAMP | 재배 시작일 |
| finished_at | TIMESTAMP | 재배 종료일 |
| created_at | TIMESTAMP | 생성일 |
| updated_at | TIMESTAMP | 수정일 |

---

## environment_setting

사용자가 최종 저장한 환경값을 **범위(min~max)** 로 저장합니다.

단일 목표값이 아닌 범위로 저장하는 이유는 Rule Engine Service의 자동 제어가
값이 범위를 벗어날 때만 장치를 동작시키고, 범위 안에서는 불필요하게 켜고 끄지 않도록(허용 오차/히스테리시스) 하기 위함입니다.

AI 추천값 자체는 저장하지 않습니다. (API/AI 추천 응답은 여전히 단일 목표값을 사용하며,
Cultivation Service가 저장 시점에 단일값을 범위로 변환합니다. 아래 "단일값 → 범위 변환" 참고)

| Column | Type |
|---------|------|
| id | BIGSERIAL |
| cultivation_id | BIGINT |
| temp_min | DECIMAL(4,1) |
| temp_max | DECIMAL(4,1) |
| humidity_min | DECIMAL(4,1) |
| humidity_max | DECIMAL(4,1) |
| co2_min | INT |
| co2_max | INT |
| light_min | INT |
| light_max | INT |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

---

## harvest

재배 종료 후 수확 정보를 저장합니다.

| Column | Type |
|---------|------|
| id | BIGSERIAL |
| cultivation_id | BIGINT |
| harvest_weight | DECIMAL(8,2) |
| memo | TEXT |
| harvested_at | TIMESTAMP |

---

## photo

사용자가 직접 촬영하여 업로드한 생육 사진의 메타데이터입니다.

이미지 원본은 MinIO에 저장하며, 이 테이블에는 URL만 저장합니다.

| Column | Type |
|---------|------|
| id | BIGSERIAL |
| cultivation_id | BIGINT |
| image_url | VARCHAR(500) |
| uploaded_at | TIMESTAMP |
| created_at | TIMESTAMP |

---

# DDL

## mushroom_reference

```sql
CREATE TABLE mushroom_reference (

    mushroom_type VARCHAR(50) PRIMARY KEY,

    temp_min DECIMAL(4,1) NOT NULL,

    temp_max DECIMAL(4,1) NOT NULL,

    humidity_min DECIMAL(4,1) NOT NULL,

    humidity_max DECIMAL(4,1) NOT NULL,

    co2_min INT NOT NULL,

    co2_max INT NOT NULL,

    light_min INT NOT NULL,

    light_max INT NOT NULL,

    description VARCHAR(500),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP

);
```

시드 데이터 예시

```sql
INSERT INTO mushroom_reference
    (mushroom_type, temp_min, temp_max, humidity_min, humidity_max, co2_min, co2_max, light_min, light_max, description)
VALUES
    ('OYSTER', 15.0, 18.0, 85, 95, 700, 900, 300, 400, '느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다.');
```

공공데이터 기준 5종(느타리, 새송이, 표고, 팽이, 양송이 등) 데이터를 시드로 등록합니다.

---

## cultivation

```sql
CREATE TABLE cultivation (

    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT NOT NULL,

    name VARCHAR(100) NOT NULL,

    mushroom_type VARCHAR(50) NOT NULL,

    status VARCHAR(20) NOT NULL DEFAULT 'CREATED',

    started_at TIMESTAMP,

    finished_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP

);
```

---

## environment_setting

```sql
CREATE TABLE environment_setting (

    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL UNIQUE,

    temp_min DECIMAL(4,1) NOT NULL,

    temp_max DECIMAL(4,1) NOT NULL,

    humidity_min DECIMAL(4,1) NOT NULL,

    humidity_max DECIMAL(4,1) NOT NULL,

    co2_min INT NOT NULL,

    co2_max INT NOT NULL,

    light_min INT NOT NULL,

    light_max INT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_environment_cultivation
        FOREIGN KEY (cultivation_id)
        REFERENCES cultivation(id)
        ON DELETE CASCADE

);
```

---

## harvest

```sql
CREATE TABLE harvest (

    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL UNIQUE,

    harvest_weight DECIMAL(8,2),

    memo TEXT,

    harvested_at TIMESTAMP NOT NULL,

    CONSTRAINT fk_harvest_cultivation
        FOREIGN KEY (cultivation_id)
        REFERENCES cultivation(id)
        ON DELETE CASCADE

);
```

---

## photo

```sql
CREATE TABLE photo (

    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    image_url VARCHAR(500) NOT NULL,

    uploaded_at TIMESTAMP NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_photo_cultivation
        FOREIGN KEY (cultivation_id)
        REFERENCES cultivation(id)
        ON DELETE CASCADE

);
```

---

# Index

## cultivation

```sql
CREATE INDEX idx_cultivation_user
ON cultivation(user_id);
```

```sql
CREATE INDEX idx_cultivation_status
ON cultivation(status);
```

---

## environment_setting

```sql
CREATE UNIQUE INDEX uk_environment_cultivation
ON environment_setting(cultivation_id);
```

---

## harvest

```sql
CREATE UNIQUE INDEX uk_harvest_cultivation
ON harvest(cultivation_id);
```

---

## photo

```sql
CREATE INDEX idx_photo_cultivation
ON photo(cultivation_id);
```

```sql
CREATE INDEX idx_photo_uploaded_at
ON photo(uploaded_at);
```

가장 최근 업로드된 사진을 빠르게 조회하기 위해 cultivation_id와 uploaded_at을 함께 사용합니다.

---

# 상태(Status)

| 값 | 설명 |
|-----|------|
| CREATED | 생성 완료 |
| RUNNING | 재배 중 |
| FINISHED | 재배 종료 |

---

# 데이터 생성 흐름

## ① 재배 생성

사용자가

- 재배 이름
- 버섯 종류

를 입력합니다.

↓

Cultivation 생성

↓

mushroom_reference 조회 (mushroom_type 기준)

↓

사용자에게 추천값(범위)으로 반환

※ mushroom_reference는 Cultivation Service 자체 조회이며, AI Service를 호출하지 않습니다.
추천값 자체는 아직 environment_setting에 저장하지 않습니다.

---

## ② 환경 저장

사용자가

온도

습도

CO₂

조도

를 수정합니다. (API 요청/응답은 단일 목표값 그대로 사용)

↓

저장 버튼 클릭

↓

Cultivation Service가 단일 목표값을 허용 오차만큼 확장하여 범위로 변환

↓

environment_setting 생성 (temp_min/max, humidity_min/max, co2_min/max, light_min/max)

### 단일값 → 범위 변환 기준 (기본값)

| 항목 | 허용 오차 | 예시 (목표값 → 저장 범위) |
|------|-----------|---------------------------|
| Temperature | ±1.5℃ | 22℃ → temp_min 20.5 / temp_max 23.5 |
| Humidity | ±5% | 90% → humidity_min 85 / humidity_max 95 |
| CO₂ | ±50ppm | 800ppm → co2_min 750 / co2_max 850 |
| Light | ±30lux | 350lux → light_min 320 / light_max 380 |

허용 오차 값은 재배 환경 저장 API 요청/응답에는 노출되지 않으며, Cultivation Service 내부 저장 로직에만 적용됩니다.
값은 향후 버섯 종류별로 다르게 조정될 수 있습니다.

---

## ③ 재배 종료

재배 종료

↓

Harvest 생성

↓

재배 상태 변경

---

# 관계

```
Auth Service

↓

userId

↓

Cultivation

↓

Environment Setting

↓

Harvest

↓

Photo (RUNNING 기간 중 언제든 업로드 가능)
```

---

# 고려 사항

- 추천 환경(mushroom_reference 조회 결과)은 Database에 별도로 저장하지 않습니다.
- 사용자가 저장한 환경(environment_setting)만 저장합니다.
- mushroom_reference는 "최적 생육 범위"(참고용 추천 데이터), environment_setting은 "위험 한계값"(실제 자동 제어 기준)으로 목적이 다릅니다. Rule Engine Service는 mushroom_reference를 직접 참조하지 않고, 항상 environment_setting(및 그 Redis 캐시)만 사용합니다.
- mushroom_reference는 cultivation_id가 없는 전역 테이블이며, 버섯 종류가 5종으로 고정되어 있어 관리자가 값을 갱신하기 전까지 정적으로 유지됩니다.
- environment_setting은 단일 목표값이 아닌 범위(min~max)로 저장합니다. Rule Engine Service가 범위를 벗어날 때만 장치를 제어하도록 하여 불필요한 On/Off를 줄이기 위함입니다.
- 단일값 → 범위 변환은 Cultivation Service 내부 로직이며, API 요청/응답 스펙에는 영향을 주지 않습니다.
- Environment Setting은 Cultivation당 하나만 존재합니다.
- Harvest는 재배 종료 후에만 생성됩니다.
- Sensor 데이터는 InfluxDB에서 관리하며 PostgreSQL에는 저장하지 않습니다.
- 사진 원본 파일은 PostgreSQL이 아닌 MinIO에 저장하고, image_url만 저장합니다.
- 사진은 카메라 센서가 아닌 사용자가 직접 촬영하여 업로드합니다.
- 하나의 재배(cultivation)에는 여러 장의 photo가 누적될 수 있습니다.