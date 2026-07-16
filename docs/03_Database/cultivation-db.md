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

> ℹ️ **변경 이력**: 센서 "장치"의 등록/조회/삭제(CRUD)는 원래 DatasourceGenerator가 담당했지만,
> 센서가 항상 특정 재배(cultivation)에 종속되는 정보이고 DatasourceGenerator는 데이터 생성/발행에만
> 집중하는 것이 책임이 명확하다고 판단해 Cultivation Service로 옮겼습니다. 이에 따라 `sensor`
> 테이블도 Cultivation Database로 이전했습니다. DatasourceGenerator는 더 이상 센서 메타데이터를
> 소유하지 않으며, Cultivation Service가 발행하는 이벤트를 구독해 시뮬레이션에 필요한 최소 정보만
> 자체 캐시(`sensor_cache`)로 보관합니다. (자세한 내용은 [datasource-generator-db.md](./datasource-generator-db.md) 참고)

> ℹ️ **변경 이력**: 센서 등록 시 사용자로부터 받는 정보가 `device_eui`(장치 고유 식별자)
> 중심으로 재정의되면서 `sensor` 테이블도 전면 교체했습니다. 대리키(`id BIGSERIAL`) 대신
> `device_eui`를 PK로 사용하고, `place`/`location`/`device_model` 컬럼을 추가했습니다.
> `sensor_uuid`/`name`/`installed_at`은 제거했고, 별도 `datasource` 엔티티에 대한
> `datasource_id` 참조도 없앴습니다. 위치 정보(place/location)는 이제 데이터 소스를 거치지
> 않고 센서 레코드에 직접 저장합니다. 이에 따라 DatasourceGenerator DB의 `datasource`
> 테이블도 함께 폐지되었습니다. (자세한 내용은 [datasource-generator-db.md](./datasource-generator-db.md) 참고)

> ℹ️ **변경 이력**: `mushroom_reference`에 이름(한글/영문/학명), 특성, 효능, 재배 가이드,
> 추가 정보 컬럼이 추가되었습니다. 원래 이 데이터는 별도 `mushroom`이라는 테이블(PK
> `mushroom_id`)로 논의되었지만, 버섯 종류당 정확히 한 행만 존재하는 정적 참조 데이터라는
> 점에서 `mushroom_reference`와 본질적으로 같은 데이터이므로 병합했습니다. PK는 기존과 동일하게
> `mushroom_type`을 유지합니다(이미 재배 생성 API, AI 버섯 가이드 API 등 시스템 전반에서
> 식별자로 쓰이고 있어, 별도 대리키 `mushroom_id`를 새로 도입할 이유가 없습니다). 원본 DDL에
> 있던 `embedding VECTOR(1024)` 컬럼은 PostgreSQL(Cultivation DB)에 두지 않았습니다. 이미
> [elasticSearch.md](./elasticSearch.md)에 "PostgreSQL과 데이터를 중복 저장하지 않는다"는
> 원칙이 있고, 임베딩을 계산/보관하는 책임은 Embedding Service에 있기 때문입니다. 대신
> Cultivation Service가 `MushroomReferenceUpdatedEvent`를 발행하면 Embedding Service가
> 구독해 Elasticsearch의 `mushroom_environment` 인덱스를 갱신합니다. (자세한 내용은
> [elasticSearch.md](./elasticSearch.md), [embedding.md](../01_Domain/embedding.md) 참고)

---

# ERD

```
mushroom_reference (전역 참조 테이블, cultivation과 FK 관계 없음)
──────────────────────────────────────────────
PK  mushroom_type
    mushroom_name_ko
    mushroom_name_en
    mushroom_scientific_name
    temp_min
    temp_max
    humidity_min
    humidity_max
    co2_min
    co2_max
    light_min
    light_max
    description
    characteristics
    health_benefits
    cultivation_guide
    additional_info
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

cultivation (1) ──── (N) sensor  ※ 위 체인과 별도로 cultivation에서 바로 분기
──────────────────────────────────────────────
PK  device_eui
FK  cultivation_id
    place
    location
    device_model
    sensor_type
    status
    created_at
    updated_at
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
| mushroom_type | VARCHAR(50) | PK, 버섯 종류 코드 (예: OYSTER) |
| mushroom_name_ko | VARCHAR(50) | 버섯 한글 이름 (예: 느타리버섯) |
| mushroom_name_en | VARCHAR(50) | 버섯 영문 이름 (예: Oyster Mushroom) |
| mushroom_scientific_name | VARCHAR(50) | 학명 (예: Pleurotus ostreatus) |
| temp_min | DECIMAL(4,1) | 최적 온도 하한 |
| temp_max | DECIMAL(4,1) | 최적 온도 상한 |
| humidity_min | DECIMAL(4,1) | 최적 습도 하한 |
| humidity_max | DECIMAL(4,1) | 최적 습도 상한 |
| co2_min | INT | 최적 CO₂ 하한 |
| co2_max | INT | 최적 CO₂ 상한 |
| light_min | INT | 최적 조도 하한 |
| light_max | INT | 최적 조도 상한 |
| description | VARCHAR(500) | 짧은 한 줄 참고 문구 (재배 생성 응답에 그대로 노출) |
| characteristics | TEXT | 버섯의 특성 설명 (버섯 가이드 RAG 컨텍스트) |
| health_benefits | TEXT | 효능 (버섯 가이드 RAG 컨텍스트) |
| cultivation_guide | TEXT | 재배 시 주의사항/가이드 (버섯 가이드 RAG 컨텍스트) |
| additional_info | TEXT | 기타 추가 정보 (버섯 가이드 RAG 컨텍스트) |
| created_at | TIMESTAMP | 생성일 |
| updated_at | TIMESTAMP | 수정일 |

`description`은 재배 생성 응답에 그대로 노출되는 짧은 참고 문구이고,
`characteristics`/`health_benefits`/`cultivation_guide`/`additional_info`는 AI Service의
"버섯 가이드"(`POST /ai/mushroom-guide`) 기능이 LLM 프롬프트에 RAG 컨텍스트로 넣어 자연스러운
문장으로 재구성하는 원문 데이터입니다. AI Service는 이 값을 그대로 반환하지 않고, LLM으로
다듬어서 `benefits`/`precautions` 형태로 응답합니다.

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

mushroom_reference 조회 결과(추천값, 범위 형태)는 그대로 저장하지 않습니다. 사용자가 이 추천값을
참고해서 **단일 목표값**으로 환경 저장 API(`PATCH /environment`)를 호출하면, 그 시점에
Cultivation Service가 단일값을 범위로 변환해 저장합니다. 아래 "단일값 → 범위 변환" 참고.
조회 응답에서 다시 단일값이 필요하면 저장된 범위의 중간값을 계산합니다. (아래 "범위 → 단일값
역변환" 참고)

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

## sensor

재배에 연결된 센서 장치의 메타데이터를 관리합니다. (기존 DatasourceGenerator DB에서 이전)

센서 등록 시 사용자로부터 받는 정보를 기준으로 설계했습니다. 대리키(BIGSERIAL) 대신
`device_eui`(장치 고유 식별자)를 PK로 사용하며, 위치 정보(place/location)를 별도
데이터 소스 엔티티 없이 센서 레코드에 직접 저장합니다.

| Column | Type | Description |
|---------|------|-------------|
| device_eui | INT | PK, 장치 고유 식별자 (하드웨어 EUI) |
| cultivation_id | BIGINT | 재배 (FK, 같은 DB 내 실제 외래키) |
| place | VARCHAR(50) | 설치 장소 (예: 1동 A구역) |
| location | VARCHAR(50) | 세부 위치 |
| device_model | VARCHAR(100) | 장치 모델명 |
| sensor_type | VARCHAR(30) | 센서 종류 |
| status | VARCHAR(20) | 상태 (ONLINE/OFFLINE/ERROR/MAINTENANCE), 시스템이 관리 (사용자 입력 아님) |
| created_at | TIMESTAMP | 생성일 |
| updated_at | TIMESTAMP | 수정일 |

`place`/`location`/`device_model`/`sensor_type`은 사용자가 센서 등록 시 직접 입력하는 값이며,
`status`는 등록 시 기본값(ONLINE)으로 시작해 이후 SensorErrorEvent로만 갱신됩니다.
`cultivation_id`는 요청 body가 아니라 URL 경로(`/cultivations/{cultivationId}/sensors`)로부터 채워집니다.

---

# DDL

## mushroom_reference

```sql
CREATE TABLE mushroom_reference (

    mushroom_type VARCHAR(50) PRIMARY KEY,

    mushroom_name_ko VARCHAR(50),

    mushroom_name_en VARCHAR(50),

    mushroom_scientific_name VARCHAR(50),

    temp_min DECIMAL(4,1) NOT NULL,

    temp_max DECIMAL(4,1) NOT NULL,

    humidity_min DECIMAL(4,1) NOT NULL,

    humidity_max DECIMAL(4,1) NOT NULL,

    co2_min INT NOT NULL,

    co2_max INT NOT NULL,

    light_min INT NOT NULL,

    light_max INT NOT NULL,

    description VARCHAR(500),

    characteristics TEXT,

    health_benefits TEXT,

    cultivation_guide TEXT,

    additional_info TEXT,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP

);
```

`embedding VECTOR(1024)` 컬럼은 이 테이블에 두지 않습니다. Elasticsearch와의 데이터 중복 저장을
피하기 위해서이며, 임베딩은 Embedding Service가 계산해 Elasticsearch에만 저장합니다. (아래
"관계" 참고)

시드 데이터 예시

```sql
INSERT INTO mushroom_reference
    (mushroom_type, mushroom_name_ko, mushroom_name_en, mushroom_scientific_name,
     temp_min, temp_max, humidity_min, humidity_max, co2_min, co2_max, light_min, light_max,
     description, characteristics, health_benefits, cultivation_guide, additional_info)
VALUES
    ('OYSTER', '느타리버섯', 'Oyster Mushroom', 'Pleurotus ostreatus',
     15.0, 18.0, 85, 95, 700, 900, 300, 400,
     '느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다.',
     '군생하며 갓은 회갈색~담회색을 띠고, 균사 성장 속도가 빠른 편입니다.',
     '식이섬유와 베타글루칸이 풍부해 면역력 강화와 콜레스테롤 감소에 도움을 줍니다.',
     '다습한 환경을 선호하지만 환기가 부족하면 곰팡이가 발생하기 쉬우니 CO₂ 농도 관리에 유의해야 합니다.',
     NULL);
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

## sensor

```sql
CREATE TABLE sensor (

    device_eui INT NOT NULL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    place VARCHAR(50),

    location VARCHAR(50),

    device_model VARCHAR(100),

    sensor_type VARCHAR(30),

    status VARCHAR(20) NOT NULL DEFAULT 'ONLINE',

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_sensor_cultivation
        FOREIGN KEY (cultivation_id)
        REFERENCES cultivation(id)
        ON DELETE CASCADE

);
```

`device_eui`는 BIGSERIAL로 자동 채번하지 않고, 등록 시 사용자가 입력한 장치 고유 식별자를
그대로 PK로 사용합니다. `place`/`location`/`device_model`/`sensor_type`은 사용자 입력 그대로
nullable이며, `status`만 시스템이 관리하는 NOT NULL 컬럼입니다. 더 이상 별도 `datasource`
테이블/FK가 없습니다(폐지됨).

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

# 상태(Status)

## cultivation

| 값 | 설명 |
|-----|------|
| CREATED | 생성 완료 |
| RUNNING | 재배 중 |
| FINISHED | 재배 종료 |

---

## sensor

| 값 | 설명 |
|-----|------|
| ONLINE | 정상 |
| OFFLINE | 연결 끊김 |
| ERROR | 오류 |
| MAINTENANCE | 점검 중 |

sensor.status는 Rule Engine Service가 발행하는 SensorErrorEvent를 Cultivation Service가 구독해 갱신합니다.
(기존에는 DatasourceGenerator가 이 이벤트를 구독했지만, sensor 테이블 이전과 함께 구독 주체도 옮겨졌습니다.)

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

### 범위 → 단일값 역변환 (조회 시)

environment_setting에는 min/max만 저장되며, 사용자가 입력했던 단일 목표값은 별도 컬럼으로 저장하지
않습니다. `GET /cultivations/{id}` 등 조회 API가 단일값을 응답해야 할 때는 저장된 범위의
**중간값**을 계산해서 사용합니다.

```
단일값 = (min + max) / 2
```

허용 오차가 항상 대칭(±고정값)으로 적용되므로, 이 중간값은 사용자가 원래 입력했던 단일 목표값과
정확히 일치합니다. 예시: temp_min 20.5 / temp_max 23.5 → (20.5+23.5)/2 = 22.0℃ (원본 입력값과 동일)

이 계산은 Cultivation Service가 조회 시점에 매번 수행하며, 별도로 캐싱하거나 추가 컬럼에
저장하지 않습니다.

---

## ③ 재배 종료

재배 종료

↓

Harvest 생성

↓

재배 상태 변경

---

## ④ 센서 등록/삭제

사용자가

- device_eui (장치 고유 식별자)
- place (설치 장소)
- location (세부 위치)
- device_model (장치 모델명)
- sensor_type (센서 종류)

를 입력해 재배에 센서를 등록합니다. cultivation_id는 URL 경로에서 채워지며, status는
시스템이 ONLINE으로 초기화합니다.

↓

sensor 생성 (device_eui가 PK, cultivation_id는 실제 FK)

↓

RabbitMQ Publish (SensorRegisteredEvent)

↓

DatasourceGenerator가 구독하여 자체 sensor_cache에 반영 (시뮬레이션 데이터 생성 대상 목록)

삭제 시에는 sensor 레코드를 삭제하고 SensorDeletedEvent를 발행해 동일하게 반영합니다.

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


Cultivation ── (1:N) ── Sensor

Sensor 등록/삭제 이벤트는 DatasourceGenerator(sensor_cache)로,
SensorErrorEvent(Rule Engine Service 발행)는 Cultivation Service(sensor.status)로 전달됩니다.
별도 datasource 엔티티는 더 이상 존재하지 않습니다(place/location을 sensor에 직접 저장).


mushroom_reference (관리자 등록/수정, cultivation과 FK 없음)

↓

RabbitMQ Publish (MushroomReferenceUpdatedEvent)

↓

Embedding Service (characteristics/health_benefits/cultivation_guide/additional_info를
임베딩하여 Elasticsearch의 mushroom_environment 인덱스 갱신)


mushroom_reference (characteristics/health_benefits/cultivation_guide/additional_info)

↓

AI Service가 OpenFeign으로 조회 (`GET /api/v1/mushroom-references/{mushroomType}`)

↓

LLM 프롬프트에 RAG 컨텍스트로 삽입 → "버섯 가이드"(효능/주의사항) 응답 생성
```

---

# 고려 사항

- 추천 환경(mushroom_reference 조회 결과)은 Database에 별도로 저장하지 않습니다.
- 사용자가 저장한 환경(environment_setting)만 저장합니다.
- mushroom_reference는 "최적 생육 범위"(참고용 추천 데이터), environment_setting은 "위험 한계값"(실제 자동 제어 기준)으로 목적이 다릅니다. Rule Engine Service는 mushroom_reference를 직접 참조하지 않고, 항상 environment_setting(및 그 Redis 캐시)만 사용합니다.
- mushroom_reference는 cultivation_id가 없는 전역 테이블이며, 버섯 종류가 5종으로 고정되어 있어 관리자가 값을 갱신하기 전까지 정적으로 유지됩니다.
- mushroom_reference에는 이름(한글/영문/학명), 특성, 효능, 재배 가이드, 추가 정보 컬럼도 함께 있지만, 이 값들의 임베딩(벡터)은 이 테이블에 저장하지 않습니다. Elasticsearch와 데이터를 중복 저장하지 않기 위해서이며, 임베딩 계산과 보관은 Embedding Service/Elasticsearch의 책임입니다.
- mushroom_reference가 생성/수정되면 `MushroomReferenceUpdatedEvent`를 발행해 Embedding Service가 Elasticsearch 인덱스를 갱신하도록 합니다. 데이터가 정적이라 이 이벤트는 관리자가 참조 데이터를 등록/수정할 때만 드물게 발생합니다.
- characteristics/health_benefits/cultivation_guide/additional_info는 AI Service가 "버섯 가이드" 기능에서 LLM 프롬프트의 RAG 컨텍스트로만 사용하며, 그대로 응답에 노출하지 않습니다.
- environment_setting은 단일 목표값이 아닌 범위(min~max)로 저장합니다. Rule Engine Service가 범위를 벗어날 때만 장치를 제어하도록 하여 불필요한 On/Off를 줄이기 위함입니다.
- 단일값 → 범위 변환은 Cultivation Service 내부 로직이며, API 요청/응답 스펙에는 영향을 주지 않습니다.
- 반대로 조회 시 단일값이 필요하면 저장된 범위의 중간값 `(min+max)/2`를 계산합니다. 허용 오차가 대칭이므로 이 값은 사용자가 원래 입력했던 단일값과 정확히 일치하며, 별도 컬럼에 원본값을 중복 저장하지 않습니다.
- Environment Setting은 Cultivation당 하나만 존재합니다.
- Harvest는 재배 종료 후에만 생성됩니다.
- 센서 "장치" 메타데이터(sensor 테이블)는 Cultivation DB(PostgreSQL)에 저장하지만, 센서가 측정한 "값"(시계열)은 Sensor Service의 InfluxDB에서 관리하며 이 DB에는 저장하지 않습니다. 두 "sensor"는 서로 다른 데이터입니다.
- sensor.device_eui는 대리키가 아닌 사용자가 입력하는 장치 고유 식별자를 그대로 PK로 사용합니다.
- 별도 datasource 테이블/엔티티는 존재하지 않습니다. 위치 정보(place/location)는 센서 레코드에 직접 저장하며, 여러 센서가 같은 place/location 값을 자유롭게 공유할 수 있습니다(정규화하지 않음).
- 사진 원본 파일은 PostgreSQL이 아닌 MinIO에 저장하고, image_url만 저장합니다.
- 사진은 카메라 센서가 아닌 사용자가 직접 촬영하여 업로드합니다.
- 하나의 재배(cultivation)에는 여러 장의 photo가 누적될 수 있습니다.