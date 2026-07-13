# Workspace Database

## 개요

Workspace는 하나의 식물을 관리하는 공간이다.

Workspace 생성 시 생성자는 ADMIN 권한을 가진다.

---

# workspace

```sql
CREATE TABLE workspace (
    workspace_id BIGSERIAL PRIMARY KEY,

    owner_user_id BIGINT NOT NULL,

    name VARCHAR(100) NOT NULL,

    description TEXT,

    thumbnail_url VARCHAR(500),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

# workspace_member

```sql
CREATE TABLE workspace_member (
    workspace_member_id BIGSERIAL PRIMARY KEY,

    workspace_id BIGINT NOT NULL,

    user_id BIGINT NOT NULL,

    role VARCHAR(20) NOT NULL,

    joined_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

# workspace_invite

```sql
CREATE TABLE workspace_invite (
    invite_id BIGSERIAL PRIMARY KEY,

    workspace_id BIGINT NOT NULL,

    invite_code VARCHAR(100) UNIQUE NOT NULL,

    expired_at TIMESTAMP NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```
# environment_setting

사용자가 최종 저장한 환경 설정

AI 추천값은 저장하지 않는다.

저장 대상

- 목표 온도
- 목표 습도
- 목표 CO₂
- 목표 pH

허용 범위

- Temperature Min
- Temperature Max

- Humidity Min
- Humidity Max

- CO₂ Min
- CO₂ Max

- pH Min
- pH Max

```sql
CREATE TABLE environment_setting (
    environment_setting_id BIGSERIAL PRIMARY KEY,

    workspace_id BIGINT NOT NULL UNIQUE,

    target_temperature DECIMAL(4,1) NOT NULL,

    target_humidity DECIMAL(4,1) NOT NULL,

    target_co2 INTEGER NOT NULL,

    target_ph DECIMAL(3,1) NOT NULL,

    temperature_min DECIMAL(4,1),

    temperature_max DECIMAL(4,1),

    humidity_min DECIMAL(4,1),

    humidity_max DECIMAL(4,1),

    co2_min INTEGER,

    co2_max INTEGER,

    ph_min DECIMAL(3,1),

    ph_max DECIMAL(3,1),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

# device

MQTT 센서

저장

- MQTT Client ID
- Topic
- Sensor Type

```sql
CREATE TABLE device (
    device_id BIGSERIAL PRIMARY KEY,

    workspace_id BIGINT NOT NULL,

    device_name VARCHAR(100),

    mqtt_client_id VARCHAR(100),

    topic VARCHAR(255),

    installed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---
# Redis
* workspace List
* workspace detail
* invite