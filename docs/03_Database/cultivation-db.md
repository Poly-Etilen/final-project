# Cultivation Database

## 개요

Cultivation Database는 사용자의 버섯 재배 정보를 관리합니다.

하나의 Cultivation은 하나의 재배를 의미하며,
재배 생성 시 AI가 추천한 환경은 저장되지 않습니다.

사용자가 추천값을 수정하거나 그대로 적용하여 저장 버튼을 눌렀을 때만
Environment Setting이 생성됩니다.

재배 종료 후에는 Harvest 정보를 저장합니다.

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

          │ 1
          │
          │
          ▼
environment_setting
──────────────────────────────────────────────
PK  id
FK  cultivation_id
    target_temperature
    target_humidity
    target_co2
    target_light
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
```

---

# Table

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

사용자가 최종 저장한 환경값입니다.

AI 추천값은 저장하지 않습니다.

| Column | Type |
|---------|------|
| id | BIGSERIAL |
| cultivation_id | BIGINT |
| target_temperature | DECIMAL(4,1) |
| target_humidity | DECIMAL(4,1) |
| target_co2 | INT |
| target_light | INT |
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

# DDL

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

    target_temperature DECIMAL(4,1) NOT NULL,

    target_humidity DECIMAL(4,1) NOT NULL,

    target_co2 INT NOT NULL,

    target_light INT NOT NULL,

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

AI Service 환경 추천

↓

사용자에게 반환

※ 아직 Database에는 저장하지 않습니다.

---

## ② 환경 저장

사용자가

온도

습도

CO₂

조도

를 수정합니다.

↓

저장 버튼 클릭

↓

environment_setting 생성

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
User Service

↓

userId

↓

Cultivation

↓

Environment Setting

↓

Harvest
```

---

# 고려 사항

- AI 추천 환경은 Database에 저장하지 않습니다.
- 사용자가 저장한 환경만 저장합니다.
- Environment Setting은 Cultivation당 하나만 존재합니다.
- Harvest는 재배 종료 후에만 생성됩니다.
- Sensor 데이터는 InfluxDB에서 관리하며 PostgreSQL에는 저장하지 않습니다.