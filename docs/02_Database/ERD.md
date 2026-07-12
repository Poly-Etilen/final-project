# Database ERD

## 개요

EcoSphere는 MSA(Database Per Service)를 적용하여 각 서비스가 자신의 데이터베이스만 관리한다.

사용되는 데이터베이스는 다음과 같다.

| Database | Service |
|----------|---------|
| PostgreSQL(User) | Auth, User |
| PostgreSQL(Workspace) | Workspace |
| InfluxDB | Storage |
| Elasticsearch | Embedding, AI |
| Redis | Auth, Workspace, Notification |

---

# Workspace ERD

```mermaid
erDiagram

WORKSPACE ||--o{ WORKSPACE_MEMBER : has
WORKSPACE ||--|| ENVIRONMENT_SETTING : has
WORKSPACE ||--o{ DEVICE : owns
WORKSPACE ||--o{ AI_REPORT : has
WORKSPACE ||--o{ WORKSPACE_INVITE : invite

WORKSPACE {
    bigint workspace_id PK
    bigint owner_user_id
    string name
    string plant_name
    string description
    string image_url
}

WORKSPACE_MEMBER {
    bigint workspace_member_id PK
    bigint workspace_id FK
    bigint user_id
    string role
}

ENVIRONMENT_SETTING {
    bigint environment_setting_id PK
    bigint workspace_id FK
    decimal target_temperature
    decimal target_humidity
    int target_co2
    decimal target_ph
}

DEVICE {
    bigint device_id PK
    bigint workspace_id FK
    string mqtt_client_id
    string topic
}

AI_REPORT {
    bigint report_id PK
    bigint workspace_id FK
    string report_type
}

WORKSPACE_INVITE {
    bigint invite_id PK
    bigint workspace_id FK
    string invite_code
}
```

---

# Database 역할

## PostgreSQL

비즈니스 데이터 저장

- 사용자
- Workspace
- 환경 설정
- Device
- AI Report

---

## InfluxDB

시계열 데이터 저장

- Temperature
- Humidity
- CO₂
- pH
- Device Status

---

## Elasticsearch

Vector Search

- 식물 환경 데이터
- Embedding Vector

---

## Redis

- JWT
- Refresh Token
- Dashboard Cache
- AI Cache
- Invite Cache