# Cultivation Database

## 개요

Cultivation Database는 사용자의 버섯 재배 정보를 관리합니다.

하나의 Cultivation은 하나의 재배를 의미하며,
재배 생성 시 추천되는 환경값은 별도로 저장되지 않습니다.

사용자가 저장하는 목표 환경값(Environment Setting)과 센서 장치 메타데이터(Sensor)는 더 이상 이
DB에 저장되지 않습니다. Sensor Service 소유의 [sensor-db.md](./sensor-db.md)를 참고하세요.

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

> ℹ️ **변경 이력**: `device_eui` 타입을 `INT`에서 `VARCHAR(32)`로 바꿨습니다. 실제
> 실습실 장비(Milesight AM107)의 MQTT 페이로드를 확인해보니 `device_eui`가
> `"24e124128c067999"`처럼 LoRaWAN 표준 64비트 DevEUI를 16자리 hex 문자열로 표현한
> 값이었습니다. 앞자리 `0`이 의미를 가질 수 있고 산술 연산이 필요 없는 순수 식별자라 숫자
> 타입보다 문자열이 맞습니다. 이 문서 전반의 `deviceEui` 예시도 `4` 같은 숫자 대신 실제
> 형식에 맞는 hex 문자열로 갱신했습니다.

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

> ℹ️ **변경 이력**: `environment_setting`을 "cultivation당 1행(범위 8개 컬럼)" 구조에서
> "항목(type)별로 여러 행이 쌓이는" 구조로 전면 재설계했습니다. 계기는 AI Service의 "일일
> 피드백" 기능(사용자가 환경을 수정했을 때 그 이후 생육이 실제로 어떻게 달라졌는지 매일
> 비교해 알려주는 기능)을 만들려면 "몇 시에 몇 도에서 몇 도로 바꿨는지"에 대한 이력이 필요한데,
> 기존 구조는 `cultivation_id UNIQUE`라 수정할 때마다 이전 값을 덮어써 이력이 전혀 남지
> 않았기 때문입니다. 처음에는 별도 `environment_setting_history` 테이블을 추가하는 방안도
> 검토했지만, 테이블 자체를 항목(TEMPERATURE/HUMIDITY/CO2/LIGHT)별 행으로 나누고 수정 시
> UPDATE 대신 INSERT만 하도록 바꾸면 `environment_setting` 하나로 "현재값"과 "이력"을 동시에
> 표현할 수 있어 이 방식으로 확정했습니다. `cultivation_id`는 더 이상 UNIQUE가 아니며,
> `cultivation`과의 관계도 1:1에서 1:N으로 바뀌었습니다. (자세한 내용은 아래 "environment_setting"
> 섹션, [daily-feedback.md](../04_sequence/daily-feedback.md) 참고)

> ℹ️ **변경 이력**: "인사이트" 기능(같은 버섯 종류 + 유사한 온도로 재배했던 타인의 사례를 바탕으로
> 피드백을 주는 기능) 추가를 위해 `harvest` 테이블에 `is_embedded` 컬럼을 추가했습니다. AI Service가
> 매일 00시 배치로 "아직 임베딩되지 않은 수확 건수"를 조회해야 하는데, AI Service가 별도의 워터마크를
> 관리하는 대신 Cultivation Service가 "임베딩 여부"를 단순 boolean 컬럼으로 소유하고 AI Service는
> 조회만 하는 방식으로 설계했습니다. (자세한 내용은 [insight.md](../04_sequence/insight.md),
> [ai.md](../01_Domain/ai.md) 참고)

> ℹ️ **변경 이력**: 팀 회의 결과, `sensor`와 `environment_setting` 두 테이블을 Cultivation DB에서
> **Sensor Service 소유의 새 DB([sensor-db.md](./sensor-db.md))로 이전**했습니다. Sensor
> Service가 이미 센서 측정값(InfluxDB/Redis)과 통계/차트/리포트를 전담하고 있어, 센서 장치
> 메타데이터와 목표 환경(위험 한계값)까지 함께 소유하는 것이 "database per service" 원칙에
> 더 맞는다고 판단했습니다. API(등록/조회/삭제/환경 저장)도 테이블과 함께 완전히 이관되었고,
> Cultivation Service는 재배 생성 시 `devices`를 함께 등록하는 기존 흐름을 유지하기 위해
> Sensor Service를 OpenFeign으로 호출합니다(더 이상 하나의 로컬 트랜잭션이 아니며, 실패 시
> 보상 삭제로 처리). 이제 Cultivation DB는 `mushroom_reference`/`cultivation`/`harvest`/
> `photo`만 소유합니다. (자세한 내용은 [sensor-db.md](./sensor-db.md),
> [cultivation-api.md](../02_API/cultivation-api.md), [sensor.md](../01_Domain/sensor.md),
> [README.md](../README.md)의 결정 사항 #24 참고)

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
harvest
──────────────────────────────────────────────
PK  id
FK  cultivation_id
    harvest_weight
    memo
    harvested_at
    is_embedded

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

`sensor`/`environment_setting`은 더 이상 이 DB에 없습니다. Sensor Service 소유의
[sensor-db.md](./sensor-db.md)를 참고하세요. `cultivation.id`를 참조하지만 DB가 분리되어
FK는 없습니다.

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

## harvest

재배 종료 후 수확 정보를 저장합니다.

| Column | Type |
|---------|------|
| id | BIGSERIAL |
| cultivation_id | BIGINT |
| harvest_weight | DECIMAL(8,2) |
| memo | TEXT |
| harvested_at | TIMESTAMP |
| is_embedded | BOOLEAN |

`is_embedded`는 "인사이트" 기능을 위해 이 수확 건이 Elasticsearch `cultivation_insight` 인덱스에
임베딩되었는지를 나타내는 플래그입니다. 기본값 `FALSE`로 시작하며, AI Service가 배치 임베딩을
완료한 뒤 `TRUE`로 갱신합니다. 한 번 `TRUE`가 되면 다시 바뀌지 않습니다. (자세한 내용은
[insight.md](../04_sequence/insight.md) 참고)

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

## harvest

```sql
CREATE TABLE harvest (

    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL UNIQUE,

    harvest_weight DECIMAL(8,2),

    memo TEXT,

    harvested_at TIMESTAMP NOT NULL,

    is_embedded BOOLEAN NOT NULL DEFAULT FALSE,

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

## harvest

```sql
CREATE UNIQUE INDEX uk_harvest_cultivation
ON harvest(cultivation_id);
```

```sql
CREATE INDEX idx_harvest_unembedded
ON harvest(is_embedded)
WHERE is_embedded = FALSE;
```

"임베딩되지 않은 수확 건수/목록"만 조회하는 것이 유일한 접근 패턴이라, `is_embedded = FALSE`
조건의 부분 인덱스(Partial Index)로 좁혀 인덱스 크기를 최소화합니다. 임베딩이 끝나 `TRUE`로
바뀐 행은 이 인덱스에서 자동으로 빠집니다.

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

`sensor` 관련 인덱스는 더 이상 이 DB에 없습니다. [sensor-db.md](./sensor-db.md) 참고.

---

# 상태(Status)

## cultivation

| 값 | 설명 |
|-----|------|
| CREATED | 생성 완료 |
| RUNNING | 재배 중 |
| FINISHED | 재배 종료 |

---

`sensor.status`는 더 이상 이 DB에 없습니다. [sensor-db.md](./sensor-db.md) 참고.

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

devices가 있다면 Sensor Service에 OpenFeign 호출 (배치 등록, `POST /api/v1/sensors/cultivations/{cultivationId}/batch`)

↓

mushroom_reference 조회 (mushroom_type 기준)

↓

사용자에게 추천값(범위)으로 반환

※ mushroom_reference는 Cultivation Service 자체 조회이며, AI Service를 호출하지 않습니다.
추천값 자체는 environment_setting(Sensor Service 소유)에 저장되지 않습니다. Sensor Service
배치 등록 호출이 실패하면 Cultivation Service는 방금 생성한 cultivation을 보상 삭제합니다.
(자세한 내용은 [cultivation-api.md](../02_API/cultivation-api.md)의 "재배 생성",
[sensor-db.md](./sensor-db.md) 참고)

환경 설정 저장(단일값 → 범위 변환, 범위 → 단일값 역변환)은 더 이상 Cultivation Service의
책임이 아닙니다. Sensor Service가 담당하며, 변환 기준은 그대로 이전되었습니다. (자세한 내용은
[sensor-db.md](./sensor-db.md), [sensor.md](../01_Domain/sensor.md) 참고)

---

## ② 재배 종료

재배 종료

↓

Harvest 생성 (is_embedded = FALSE로 초기화)

↓

재배 상태 변경

수확 시점에는 임베딩이 즉시 일어나지 않습니다. `is_embedded`는 이후 AI Service의 00시 배치
스케줄러가 미임베딩 건수를 모아 임계치(20개) 이상일 때 일괄 처리한 뒤에만 `TRUE`로 바뀝니다.
(자세한 내용은 [insight.md](../04_sequence/insight.md) 참고)

센서 등록/삭제와 환경 저장 흐름은 더 이상 이 DB의 책임이 아닙니다. [sensor-db.md](./sensor-db.md)
참고.

---

# 관계

```
Auth Service

↓

userId

↓

Cultivation

↓ (1:1)

Harvest

↓

Photo (RUNNING 기간 중 언제든 업로드 가능)


Cultivation ── (cultivation_id, FK 없음) ── Sensor Service (sensor-db.md: sensor, environment_setting)

Cultivation Service는 재배 생성 시 Sensor Service를 OpenFeign으로 호출해 devices를 배치
등록하고(실패 시 cultivation 보상 삭제), 재배 삭제 시 CultivationDeletedEvent를 발행해 Sensor
Service가 해당 cultivation_id의 sensor/environment_setting을 정리하도록 합니다. Sensor Service는
쓰기 요청 시 Cultivation Service의 `GET /api/v1/cultivations/{cultivationId}/owner`로 소유권을
확인합니다. (자세한 내용은 [sensor-db.md](./sensor-db.md) 참고)


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


harvest.is_embedded (인사이트 기능용)

↓

AI Service가 OpenFeign으로 조회
  - GET /api/v1/harvests/unembedded-count (미임베딩 건수)
  - GET /api/v1/harvests/unembedded (미임베딩 목록 + 환경 평균)
  - PATCH /api/v1/harvests/embedded (임베딩 완료 처리)

↓

Embedding Service (Elasticsearch cultivation_insight 인덱스에 저장)

Cultivation Service는 "임베딩되었는지 여부"만 소유하며, 임베딩 자체와 Elasticsearch 저장/검색은
전적으로 Embedding Service의 책임입니다. (자세한 내용은 [insight.md](../04_sequence/insight.md),
[embedding.md](../01_Domain/embedding.md) 참고)
```

---

# 고려 사항

- 추천 환경(mushroom_reference 조회 결과)은 Database에 별도로 저장하지 않습니다.
- 사용자가 저장하는 목표 환경(environment_setting)과 센서 장치 메타데이터(sensor)는 Sensor Service 소유의 DB로 이전되어 더 이상 이 Database에 저장되지 않습니다. 설계 근거(범위 저장, INSERT-only 이력, device_eui PK 등)는 [sensor-db.md](./sensor-db.md)를 참고하세요.
- mushroom_reference는 cultivation_id가 없는 전역 테이블이며, 버섯 종류가 5종으로 고정되어 있어 관리자가 값을 갱신하기 전까지 정적으로 유지됩니다.
- mushroom_reference에는 이름(한글/영문/학명), 특성, 효능, 재배 가이드, 추가 정보 컬럼도 함께 있지만, 이 값들의 임베딩(벡터)은 이 테이블에 저장하지 않습니다. Elasticsearch와 데이터를 중복 저장하지 않기 위해서이며, 임베딩 계산과 보관은 Embedding Service/Elasticsearch의 책임입니다.
- mushroom_reference가 생성/수정되면 `MushroomReferenceUpdatedEvent`를 발행해 Embedding Service가 Elasticsearch 인덱스를 갱신하도록 합니다. 데이터가 정적이라 이 이벤트는 관리자가 참조 데이터를 등록/수정할 때만 드물게 발생합니다.
- characteristics/health_benefits/cultivation_guide/additional_info는 AI Service가 "버섯 가이드" 기능에서 LLM 프롬프트의 RAG 컨텍스트로만 사용하며, 그대로 응답에 노출하지 않습니다.
- Harvest는 재배 종료 후에만 생성됩니다.
- `harvest.is_embedded`는 "인사이트" 기능(타인의 유사 재배 사례 기반 피드백)을 위한 플래그이며, 기본값 `FALSE`로 시작해 AI Service의 배치 임베딩이 끝난 뒤에만 `TRUE`로 바뀝니다. 임베딩 자체(텍스트 요약 생성, 벡터화, Elasticsearch 저장)는 Cultivation Service가 아닌 Embedding Service의 책임이며, Cultivation Service는 "여부"만 소유합니다.
- `is_embedded` 판단을 위한 환경 평균값(기간 가중 평균)은 더 이상 Cultivation Service가 계산하지 않습니다. environment_setting이 Sensor Service로 이전되면서, AI Service의 인사이트 배치 스케줄러가 Cultivation Service(미임베딩 수확 목록)와 Sensor Service(`POST /api/v1/sensors/environment-averages`, 환경 평균 일괄 조회)를 각각 호출해 조합합니다. (자세한 내용은 [insight.md](../04_sequence/insight.md), [sensor-db.md](./sensor-db.md) 참고)
- 사진 원본 파일은 PostgreSQL이 아닌 MinIO에 저장하고, image_url만 저장합니다.
- 사진은 카메라 센서가 아닌 사용자가 직접 촬영하여 업로드합니다.
- 하나의 재배(cultivation)에는 여러 장의 photo가 누적될 수 있습니다.