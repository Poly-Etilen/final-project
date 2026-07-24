# Cultivation Database

## 개요

Cultivation Database는 사용자의 버섯 재배 자체와 그 재배에 필요한 센서/환경 데이터를
함께 관리하는 Cultivation Service 소유의 PostgreSQL Database입니다(구 Sensor
Service 통합). 재배 생성/진행/종료, 수확 기록, 생육 사진 메타데이터, 센서
장치, 목표 환경(위험 한계값) 이력, 공공데이터 기반 버섯 참조 데이터, 문의(Inquiry)를
모두 이 DB에서 관리합니다.

이전에는 Cultivation DB와 Sensor DB가 분리되어 있어 `cultivation_id`/`mushroom_type`
같은 참조가 소프트 참조(FK 없음)였지만, 병합 이후에는 같은 DB 안이므로 실제 FK로
바꿀 수 있는 관계는 FK로 강제합니다(`sensor.cultivation_id`,
`environment_setting.cultivation_id`, `cultivation.mushroom_type`).

---

# ERD

```
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

    UNIQUE(user_id, name)

          │ 1
          │
          ├──────────────┬──────────────┬──────────────┐
          ▼              ▼              ▼              ▼
harvest              photo          sensor      environment_setting
──────────────       ──────────     ──────────  ──────────────────
PK  id                PK  id         PK  id       PK  id
FK  cultivation_id    FK  cultivation_id  FK  cultivation_id  FK  cultivation_id
(UNIQUE)                  object_key      device_eui (UNIQUE) FK  measurement_type_code
    harvest_weight        storage_type    location            threshold_min
    memo                  uploaded_at     location_detail     threshold_max
    harvested_at          created_at      device_model        threshold_unit
    product_score                        status              created_at
    product_grade                        is_deleted          updated_at
                                          created_at
                                              │ 1
                                              │
                                              ▼
                                          sensor_type
                                          ──────────────
                                          PK  id
                                          FK  sensor_id
                                          FK  measurement_type_code

                                          UNIQUE(sensor_id, measurement_type_code)


measurement_type (전역 참조, 4종 고정 시드 데이터)
──────────────────────────────────────────────
PK  code
    name_ko
    unit


mushroom_reference (전역 참조 테이블)
──────────────────────────────────────────────
PK  id
    mushroom_type     (UNIQUE — cultivation.mushroom_type이 FK로 참조)
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
          ▼
mushroom_reference_threshold
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


inquiry  (cultivation_id는 의도적으로 소프트 참조 — 재배 삭제 후에도 문의 기록 보존)
──────────────────────────────────────────────
PK  id
    user_id           (Auth Service, 소프트 참조)
    type
    cultivation_id    (nullable, 소프트 참조)
    title
    content
    answer
    status
    admin_id          (Auth Service, 소프트 참조)
    answered_at
    created_at
    updated_at
```

---

# Table

## cultivation

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | X | 소유자 (Auth Service의 userId, 소프트 참조) |
| name | VARCHAR(100) | X | 재배 이름 (사용자 지정), `UNIQUE(user_id, name)` |
| mushroom_type | VARCHAR(20) | X | 버섯 종류 코드 (예: `OYSTER`), FK `mushroom_reference.mushroom_type` |
| status | VARCHAR(20) | X | 재배 상태 (CREATED/RUNNING/FINISHED), 기본값 CREATED |
| started_at | DATETIME | O | 첫 목표 환경 설정 저장 시점에 자동으로 채워짐 |
| finished_at | DATETIME | O | 재배 종료 시점 |
| created_at | DATETIME | X | 생성 일시 |
| updated_at | DATETIME | X | 수정 일시 |

재배 이름은 전역이 아니라 **같은 사용자 안에서만 유일**합니다. `status`는 재배 생성
시 `CREATED`로 시작해, 목표 환경을 처음 저장하는 시점에 같은 트랜잭션에서
`RUNNING`으로 자동 전환되고, 사용자가 명시적으로 종료하면 `FINISHED`가 됩니다.
`mushroom_type`은 이제 같은 DB 안의 `mushroom_reference.mushroom_type`을 실제 FK로
참조합니다(병합 이전에는 소프트 참조였음).

---

## harvest

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id`, UNIQUE |
| harvest_weight | NUMERIC(6,1) | O | 수확량(g) |
| memo | TEXT | O | 수확 메모 |
| harvested_at | DATETIME | X | 수확 일시 |
| product_score | NUMERIC(5,2) | O | 상품 등급 원점수(0~100). AI Service가 생육 점수 평균과 환경 유지 점수를 합산해 계산 후 전달 |
| product_grade | VARCHAR(10) | O | 상품 등급 (TOP/HIGH/MID/LOW). Cultivation Service가 `product_score`를 구간별로 매핑 |

재배 하나당 수확은 한 건만 기록됩니다(`UNIQUE(cultivation_id)`). 병 재배를 전제로
하나의 재배 단위(병)에서 자란 버섯은 한 번에 수확하며, 재배마다 여러 번 나눠
수확하지 않습니다.

`product_score`/`product_grade`는 수확 기록 시점에는 비어 있다가, `HarvestCompletedEvent`를
구독한 AI Service가 생육 점수 평균(자체 DB의 `growth_record`)과 환경 유지 점수(Cultivation
Service에 새로 조회)를 합산해 원점수를 계산해 돌려주면 그때 채워집니다. Cultivation
Service는 전달받은 원점수를 아래 구간으로 매핑해 `product_grade`에 저장합니다.

| product_score | product_grade |
|----------------|----------------|
| 95점 이상 | TOP (최상) |
| 80점 이상 | HIGH (상) |
| 60점 이상 | MID (중) |
| 60점 미만 | LOW (하) |

자세한 흐름은 [product-grade.md](../04_sequence/product-grade.md) 참고.

---

## photo

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id` |
| object_key | VARCHAR(500) | X | 저장소 내 상대 경로 |
| storage_type | VARCHAR(10) | X | 저장 위치 (MINIO/LOCAL), 기본값 MINIO |
| uploaded_at | DATETIME | X | 사용자가 촬영/업로드한 시각 |
| created_at | DATETIME | X | DB에 저장된 시각 |

자세한 내용은 [minio.md](./minio.md) 참고.

---

## measurement_type

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| code | VARCHAR(20) | X | PK (자연키, 예: TEMPERATURE/HUMIDITY/CO2/LIGHT) |
| name_ko | VARCHAR(20) | X | 표시용 한글 이름 (예: 온도) |
| unit | VARCHAR(10) | X | 기본 단위 (예: ℃, %, ppm, lux) |

측정 항목 4종의 고정 시드 데이터입니다. `sensor_type`/`environment_setting`/
`mushroom_reference_threshold` 세 곳이 이 값을 자유 문자열로 반복하지 않고 이
테이블을 FK로 참조합니다.

---

## sensor

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id` |
| device_eui | VARCHAR(32) | X | 장치 고유 식별자 (LoRaWAN DevEUI), UNIQUE |
| location | VARCHAR(10) | O | 설치 위치 (짧은 코드, 예: A동) |
| location_detail | VARCHAR(50) | O | 세부 위치 |
| device_model | VARCHAR(100) | O | 장치 모델명 |
| status | VARCHAR(10) | X | 상태 (ONLINE/OFFLINE/ERROR), 기본값 OFFLINE, 시스템이 관리 |
| is_deleted | BOOLEAN | X | 소프트 삭제 플래그, 기본값 FALSE |
| created_at | DATETIME | X | 등록 일시 |

PK는 대리키 `id`이고 `device_eui`는 UNIQUE 제약을 가진 일반 컬럼입니다. 삭제는
`is_deleted` 플래그로 처리합니다. `cultivation_id`는 병합 이후 같은 DB 안의 실제
FK입니다(이전에는 소프트 참조).

---

## sensor_type

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| sensor_id | BIGINT | X | FK, `sensor.id` |
| measurement_type_code | VARCHAR(20) | X | FK, `measurement_type.code` |

장치 하나가 여러 항목을 동시에 측정할 수 있어 `sensor`와 1:N 관계입니다.
`UNIQUE(sensor_id, measurement_type_code)`로 중복 등록을 막습니다.

---

## environment_setting

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id` |
| measurement_type_code | VARCHAR(20) | X | FK, `measurement_type.code` |
| threshold_min | NUMERIC(10,4) | X | 하한 |
| threshold_max | NUMERIC(10,4) | X | 상한 |
| threshold_unit | VARCHAR(10) | X | 단위 |
| created_at | DATETIME | X | 이 값이 저장된 시각 |
| updated_at | DATETIME | O | 생성 시각과 동일하게 유지됨 (UPDATE 경로 없음) |

사용자가 최종 저장한 환경값을 **범위**로, 항목별 행으로 저장합니다. 수정할 때도
UPDATE가 아니라 새 행을 INSERT합니다 — 이 테이블 하나가 "현재값"과 "이력"을 동시에
표현합니다. `cultivation_id`는 병합 이후 같은 DB 안의 실제 FK입니다.

---

## mushroom_reference

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| mushroom_type | VARCHAR(20) | X | 버섯 종류 코드 (예: OYSTER), UNIQUE. `cultivation.mushroom_type`이 FK로 참조 |
| mushroom_name_ko | VARCHAR(50) | X | 버섯 한글 이름 |
| mushroom_name_en | VARCHAR(50) | X | 버섯 영문 이름 |
| mushroom_scientific_name | VARCHAR(50) | O | 학명 |
| characteristics | TEXT | O | 특성 설명 |
| health_benefits | TEXT | O | 효능 |
| cultivation_guide | TEXT | O | 재배 가이드 |
| additional_info | TEXT | O | 추가 정보 |
| created_at | DATETIME | X | 생성일 |
| updated_at | DATETIME | O | 수정일 |

공공데이터 기반 5종 버섯의 고정 시드 데이터입니다. 텍스트 컬럼들은 AI Service의
챗봇/버섯 가이드 기능이 `mushroomType`으로 정확히 일치하는 한 건을 그대로 조회해 LLM
컨텍스트(RAG 원문)로 사용합니다. 5종 고정이라 별도의 임베딩·검색 색인 없이 직접
조회로 충분합니다.

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
`UNIQUE(mushroom_reference_id, measurement_type_code)`로 중복 등록을 막습니다.

---

## inquiry

시스템 관리자를 제외한 모든 사용자가 남길 수 있는 문의입니다. 일반 문의(GENERAL)와
경작 문의(CULTIVATION) 두 유형으로 나뉩니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | X | 작성자 (Auth Service, 소프트 참조) |
| type | VARCHAR(20) | X | 문의 유형 (GENERAL/CULTIVATION) |
| cultivation_id | BIGINT | O | CULTIVATION 유형일 때만 채워짐. 의도적으로 소프트 참조(FK 없음) — 재배가 삭제된 뒤에도 문의 기록은 남아야 하기 때문 |
| title | VARCHAR(200) | X | 제목 |
| content | TEXT | X | 문의 내용 |
| answer | TEXT | O | 관리자 답변 (GENERAL 유형에서만 사용) |
| status | VARCHAR(20) | X | 처리 상태 (OPEN/ANSWERED/CLOSED), 기본값 OPEN |
| admin_id | BIGINT | O | 처리한 관리자 (Auth Service, 소프트 참조) |
| answered_at | DATETIME | O | 답변/처리 시각 |
| created_at | DATETIME | X | 작성 일시 |
| updated_at | DATETIME | X | 수정 일시 |

GENERAL 유형은 `cultivation_id`가 항상 NULL이고, 관리자가 `answer`를 작성하면
`status`가 `ANSWERED`로 바뀝니다. CULTIVATION 유형은 `cultivation_id`가 항상 채워져
있어야 하며, `answer`는 사용하지 않습니다 — 관리자는 문의를 읽고 해당
`cultivation_id`가 가리키는 재배를 삭제하는 것으로 처리하며, 이때 `status`가
`CLOSED`로 바뀝니다. `cultivation_id`는 같은 DB 안의 테이블을 가리키지만 의도적으로
FK를 걸지 않았습니다 — FK를 걸면 재배 삭제 시 `ON DELETE CASCADE`/`SET NULL` 중
하나를 선택해야 하는데, 전자는 처리 이력(문의) 자체가 함께 사라지고 후자는 "어떤
재배였는지"를 잃어버리기 때문입니다.

---

# DDL

```sql
CREATE TABLE cultivation (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT NOT NULL,

    name VARCHAR(100) NOT NULL,

    mushroom_type VARCHAR(20) NOT NULL,

    status VARCHAR(20) NOT NULL DEFAULT 'CREATED',

    started_at TIMESTAMP,

    finished_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uk_cultivation_user_name UNIQUE (user_id, name),

    CONSTRAINT chk_cultivation_status CHECK (status IN ('CREATED', 'RUNNING', 'FINISHED'))
);
```

```sql
CREATE TABLE harvest (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    harvest_weight NUMERIC(6,1),

    memo TEXT,

    harvested_at TIMESTAMP NOT NULL,

    product_score NUMERIC(5,2),

    product_grade VARCHAR(10),

    CONSTRAINT fk_harvest_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id) ON DELETE CASCADE,

    CONSTRAINT uk_harvest_cultivation UNIQUE (cultivation_id),

    CONSTRAINT chk_harvest_product_grade CHECK (product_grade IN ('TOP', 'HIGH', 'MID', 'LOW'))
);
```

```sql
CREATE TABLE photo (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    object_key VARCHAR(500) NOT NULL,

    storage_type VARCHAR(10) NOT NULL DEFAULT 'MINIO',

    uploaded_at TIMESTAMP NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_photo_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id) ON DELETE CASCADE,

    CONSTRAINT chk_photo_storage_type CHECK (storage_type IN ('MINIO', 'LOCAL'))
);
```

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

    CONSTRAINT fk_sensor_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id) ON DELETE CASCADE,

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
        FOREIGN KEY (sensor_id) REFERENCES sensor(id) ON DELETE CASCADE,

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

    CONSTRAINT fk_environment_setting_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id) ON DELETE CASCADE,

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

```sql
ALTER TABLE cultivation
    ADD CONSTRAINT fk_cultivation_mushroom_type
    FOREIGN KEY (mushroom_type) REFERENCES mushroom_reference(mushroom_type);
```

```sql
CREATE TABLE inquiry (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT NOT NULL,

    type VARCHAR(20) NOT NULL,

    cultivation_id BIGINT,

    title VARCHAR(200) NOT NULL,

    content TEXT NOT NULL,

    answer TEXT,

    status VARCHAR(20) NOT NULL DEFAULT 'OPEN',

    admin_id BIGINT,

    answered_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_inquiry_type CHECK (type IN ('GENERAL', 'CULTIVATION')),

    CONSTRAINT chk_inquiry_status CHECK (status IN ('OPEN', 'ANSWERED', 'CLOSED')),

    CONSTRAINT chk_inquiry_cultivation_id CHECK (
        (type = 'GENERAL' AND cultivation_id IS NULL) OR
        (type = 'CULTIVATION' AND cultivation_id IS NOT NULL)
    )
);
```

`mushroom_reference`가 `mushroom_reference_threshold`보다 먼저 생성되어야 하고,
`cultivation`이 `mushroom_reference` 생성 이후에 FK를 걸 수 있어 `cultivation.
mushroom_type` FK는 `ALTER TABLE`로 별도 추가합니다(시드 데이터 적재 순서 때문).
`inquiry.cultivation_id`/`inquiry.user_id`/`inquiry.admin_id`는 의도적으로 FK를
걸지 않습니다.

---

# Index

```sql
CREATE INDEX idx_cultivation_user
ON cultivation(user_id, created_at DESC);
```

사용자별 재배 목록 조회에 사용합니다.

재배당 수확이 한 건뿐이라 `harvest.cultivation_id`의 `UNIQUE` 제약이 만드는 인덱스로
조회가 충분해, 별도 인덱스는 두지 않습니다.

```sql
CREATE INDEX idx_photo_cultivation
ON photo(cultivation_id, uploaded_at DESC);
```

재배별 최신 사진 조회에 사용합니다.

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

```sql
CREATE INDEX idx_inquiry_user
ON inquiry(user_id, created_at DESC);
```

사용자 본인의 문의 목록 조회에 사용합니다.

```sql
CREATE INDEX idx_inquiry_status
ON inquiry(status, created_at DESC);
```

관리자의 미답변/미처리 문의 큐 조회에 사용합니다.

```sql
CREATE INDEX idx_inquiry_cultivation
ON inquiry(cultivation_id)
WHERE cultivation_id IS NOT NULL;
```

특정 재배에 대한 경작 문의 조회에 사용합니다.

---

# 관계

```
harvest, photo, sensor, environment_setting
    │ cultivation_id (같은 DB 내 실제 FK, ON DELETE CASCADE)
    ▼
cultivation
    │ mushroom_type (같은 DB 내 실제 FK)
    ▼
mushroom_reference

sensor_type
    │ sensor_id (실제 FK), measurement_type_code (실제 FK)
    ▼
sensor, measurement_type

environment_setting
    │ measurement_type_code (실제 FK)
    ▼
measurement_type

mushroom_reference_threshold
    │ mushroom_reference_id, measurement_type_code (실제 FK)
    ▼
mushroom_reference, measurement_type

inquiry
    │ user_id, admin_id (Auth Service, 소프트 참조)
    │ cultivation_id (같은 DB지만 의도적 소프트 참조)
    ▼
cultivation (참조는 하되 FK 없음)
```

Cultivation Service는 병합 이전 Sensor Service와 데이터베이스를 공유하게 되어,
`cultivation_id`/`mushroom_type` 참조가 소프트 참조에서 실제 FK로 바뀌었습니다.
`user_id`(Auth Service)는 여전히 다른 서비스 DB를 가리키는 소프트 참조입니다.

---

# 고려 사항

- 재배 이름 유일성은 전역이 아니라 사용자 단위입니다.
- `CREATED → RUNNING` 전환은 목표 환경 저장과 같은 트랜잭션 안에서 처리됩니다(병합
  이전에는 `EnvironmentRangeUpdatedEvent`를 스스로 구독하는 방식이었으나, 같은 서비스
  안이 되면서 불필요해졌습니다). Rule Engine Service를 위한 이벤트 발행 자체는
  계속됩니다.
- 수확이 기록되면 `HarvestCompletedEvent`가 발행되어, AI Service가 이를 구독해
  "인사이트" 사례를 즉시 적재하고, 상품 등급 원점수(`product_score`)를 계산해
  Cultivation Service에 돌려줍니다. `product_grade`는 그 원점수를 받은 Cultivation
  Service가 매깁니다. 이 콜백 호출이 실패하면 `product_score`/`product_grade`는
  NULL로 남으며, 재시도 정책은 추후 정합니다.
- 사진 저장소(`storage_type`)를 MinIO에서 로컬로(또는 그 반대로) 전환해도 기존 행을
  다시 쓸 필요가 없습니다.
- `sensor`/`environment_setting`은 병합 이전 소프트 참조였던 `cultivation_id`가 실제
  FK(`ON DELETE CASCADE`)로 바뀌어, 재배 삭제 시 별도의 보상 이벤트 없이 DB
  트랜잭션만으로 함께 정리됩니다.
- `measurement_type`을 별도 참조 테이블로 뺀 이유는 온도/습도/CO₂/조도라는 같은 값
  집합이 여러 테이블에서 자유 문자열로 반복되지 않도록 하기 위함입니다.
- `inquiry.cultivation_id`만 예외적으로 FK를 걸지 않습니다. 재배가 삭제된 뒤에도
  "어떤 재배에 대한 문의였는지" 기록이 남아야 하기 때문입니다.
