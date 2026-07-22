# Cultivation Database

## 개요

Cultivation Database는 사용자의 버섯 재배 자체를 관리하는 Cultivation Service 소유의
PostgreSQL Database입니다. 재배 생성/진행/종료, 재배당 여러 번 발생하는 수확(flush) 기록,
생육 사진 메타데이터를 `cultivation`/`harvest`/`photo` 세 테이블로 관리합니다.

센서 장치·목표 환경값·버섯 참조 데이터는 Sensor Service 소유의 별도 DB에서 관리합니다.
Cultivation Service는 재배를 생성할 때 Sensor Service를 OpenFeign으로 호출해 센서를
등록하고, 재배 소유권 확인이 필요한 다른 서비스에게 내부용 API(`GET
/cultivations/{id}/owner`)를 제공합니다.

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
          │
          ▼
harvest
──────────────────────────────────────────────
PK  id
FK  cultivation_id
    flush_no
    harvest_weight
    memo
    harvested_at

    UNIQUE(cultivation_id, flush_no)

          │ 1
          │
          │
          ▼
photo
──────────────────────────────────────────────
PK  id
FK  cultivation_id
    object_key
    storage_type
    uploaded_at
    created_at
```

---

# Table

## cultivation

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | X | 소유자 (Auth Service의 userId, 소프트 참조) |
| name | VARCHAR(100) | X | 재배 이름 (사용자 지정), `UNIQUE(user_id, name)` |
| mushroom_type | VARCHAR(50) | X | 버섯 종류 코드 (예: `OYSTER`). Sensor Service `mushroom_reference.mushroom_type`을 가리키는 소프트 참조 |
| status | VARCHAR(20) | X | 재배 상태 (CREATED/RUNNING/FINISHED), 기본값 CREATED |
| started_at | DATETIME | O | 첫 목표 환경 설정 저장 시점에 자동으로 채워짐 |
| finished_at | DATETIME | O | 재배 종료 시점 |
| created_at | DATETIME | X | 생성 일시 |
| updated_at | DATETIME | X | 수정 일시 |

재배 이름은 전역이 아니라 **같은 사용자 안에서만 유일**합니다(다른 사용자는 같은 이름을 쓸
수 있음). `status`는 재배 생성 시 `CREATED`로 시작해, Sensor Service에 목표 환경을 처음
저장하는 시점에 `RUNNING`으로 자동 전환되고, 사용자가 명시적으로 종료하면 `FINISHED`가
됩니다.

---

## harvest

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id` |
| flush_no | SMALLINT | X | 이 재배의 몇 번째 수확인지 (서버 자동 채번) |
| harvest_weight | NUMERIC(6,1) | O | 수확량(g) |
| memo | TEXT | O | 수확 메모 |
| harvested_at | DATETIME | X | 수확 일시 |

한 재배는 `RUNNING` 상태인 동안 여러 번 수확을 기록할 수 있습니다(흔히 "플러시"라고 부름).
`UNIQUE(cultivation_id, flush_no)`로 같은 재배 안에서 순번이 중복되지 않도록 합니다. 재배
종료(`status → FINISHED`)와 수확 기록 저장은 서로 다른 시점에 호출되는 별개의 동작입니다.

---

## photo

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id` |
| object_key | VARCHAR(500) | X | 저장소 내 상대 경로 (예: `3/20260815-090000.jpg`) |
| storage_type | VARCHAR(10) | X | 저장 위치 (MINIO/LOCAL), 기본값 MINIO |
| uploaded_at | DATETIME | X | 사용자가 촬영/업로드한 시각 |
| created_at | DATETIME | X | DB에 저장된 시각 |

실제 이미지 파일은 `storage_type`이 가리키는 저장소(MinIO 또는 로컬 파일시스템)에 있고,
이 테이블은 메타데이터만 관리합니다. `object_key`는 저장소에 무관하게 파일을 가리키는
상대 경로이며, 실제로 접근 가능한 URL/파일 경로로 바꾸는 조합 로직은 애플리케이션 설정
(저장소별 base path/endpoint)에 있습니다 — 완성된 URL을 DB에 저장하지 않는 이유는, 저장소
자체가 바뀌어도(MinIO ↔ 로컬) 기존에 저장된 행을 수정할 필요가 없도록 하기 위함입니다.
자세한 내용은 [minio.md](./minio.md) 참고.

---

# DDL

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

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uk_cultivation_user_name UNIQUE (user_id, name),

    CONSTRAINT chk_cultivation_status CHECK (status IN ('CREATED', 'RUNNING', 'FINISHED'))
);
```

```sql
CREATE TABLE harvest (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    flush_no SMALLINT NOT NULL,

    harvest_weight NUMERIC(6,1),

    memo TEXT,

    harvested_at TIMESTAMP NOT NULL,

    CONSTRAINT fk_harvest_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id),

    CONSTRAINT uk_harvest_cultivation_flush UNIQUE (cultivation_id, flush_no)
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
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id),

    CONSTRAINT chk_photo_storage_type CHECK (storage_type IN ('MINIO', 'LOCAL'))
);
```

---

# Index

```sql
CREATE INDEX idx_cultivation_user
ON cultivation(user_id, created_at DESC);
```

사용자별 재배 목록 조회에 사용합니다.

```sql
CREATE INDEX idx_harvest_cultivation
ON harvest(cultivation_id, flush_no);
```

재배별 수확 이력 조회(`GET /cultivations/{id}/harvests`)에 사용합니다.

```sql
CREATE INDEX idx_photo_cultivation
ON photo(cultivation_id, uploaded_at DESC);
```

재배별 최신 사진 조회에 사용합니다.

---

# 관계

```
harvest, photo (Cultivation Service)

    │ cultivation_id (같은 DB 내 실제 FK)
    ▼
cultivation
```

Cultivation Service는 다른 서비스와 데이터베이스를 공유하지 않습니다. `user_id`(Auth
Service), `mushroom_type`(Sensor Service의 `mushroom_reference` 코드)은 모두 소프트
참조입니다.

---

# 고려 사항

- 재배 이름 유일성은 전역이 아니라 사용자 단위입니다. 서로 다른 사용자가 같은 이름의
  재배를 가질 수 있습니다.
- `CREATED → RUNNING` 전환은 Cultivation Service가 직접 트리거하지 않고, Sensor Service가
  발행하는 `EnvironmentRangeUpdatedEvent`를 구독해 해당 재배에 대해 이 이벤트를 처음 받는
  시점에 처리합니다(이미 RUNNING/FINISHED면 아무 동작 없음 — 멱등).
- 수확이 기록되면 `HarvestCompletedEvent`가 발행되어, AI Service가 이를 구독해
  "인사이트" 사례를 즉시 적재합니다(자세한 내용은 [ai-db.md](./ai-db.md) 참고).
- 사진은 여러 장 등록될 수 있으며, 생육 분석(Vision)에는 가장 최근 업로드본을 사용합니다.
  과거 사진도 삭제하지 않고 보관합니다.
- 사진 저장소(`storage_type`)를 MinIO에서 로컬로(또는 그 반대로) 전환해도 기존 행을 다시
  쓸 필요가 없습니다 — `object_key`는 저장소 중립적인 값이고, 실제 접근 경로를 조합하는
  로직만 설정을 바꾸면 되기 때문입니다.
