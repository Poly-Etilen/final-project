# Database Overview

## 데이터베이스 구성

본 프로젝트는 데이터의 특성에 따라 서로 다른 데이터베이스를 사용합니다.

| Database | 용도 | 사용 서비스 |
|----------|------|------------|
| PostgreSQL | 관계형 데이터 저장 | Auth, Cultivation, DatasourceGenerator |
| Redis | 캐시 및 임시 데이터 | Auth, AI, Sensor |
| InfluxDB | 시계열 센서 데이터 | Sensor |
| Elasticsearch | Vector Search | Embedding |
| MinIO | 생육 사진(이미지) 저장 | Cultivation, AI |

---

# PostgreSQL

## Auth DB

### 목적

인증 정보와 회원 프로필 정보를 하나의 `users` 테이블로 관리합니다. (기존 Auth+User 서비스 통합)

### Table

- users

---

## Cultivation DB

### 목적

버섯 재배 정보를 관리합니다.

### Table

- cultivation
- environment_setting
- harvest
- photo

---

## DatasourceGenerator DB

### 목적

센서 및 데이터 소스를 관리합니다. (기존 명칭: Datasource DB)

### Table

- datasource
- sensor

---

# Redis

Redis는 캐시 및 임시 데이터를 저장합니다.

## Auth

### Refresh Token

```
refresh:{userId}
```

---

### Email Verification

```
email:{email}
```

---

## AI

### AI Response Cache

```
ai:{hash}
```

---

## Sensor

### Current Environment

```
cultivation:{cultivationId}:current
```

Rule Engine Service가 RabbitMQ(EnvironmentMeasuredEvent)로 전달한 데이터를 Sensor Service가 저장합니다.

---

# InfluxDB

## Measurement

```
environment
```

### Tags

- cultivationId
- sensorId

### Fields

- temperature
- humidity
- co2
- light

Sensor Service가 저장/조회를 전담합니다. Rule Engine Service는 RabbitMQ로 데이터를 전달만 합니다.

---

# Elasticsearch

## Index

```
mushroom_environment
```

### Document

- mushroomType
- temperature
- humidity
- co2
- light
- description
- embedding(Vector)

---

# MinIO

버섯 생육 사진(이미지)을 저장하는 객체 저장소입니다.

## Bucket

```
mushroom-photos
```

## 사용 서비스

| Service | 역할 |
|----------|------|
| Cultivation Service | 사진 업로드 |
| AI Service | 사진 조회 (Vision 분석용) |

---

# 서비스별 Database

| Service | PostgreSQL | Redis | InfluxDB | Elasticsearch | MinIO |
|----------|------------|--------|-----------|---------------|-------|
| Auth | O | O | X | X | X |
| Cultivation | O | X | X | X | O |
| AI | X | O | X | X | O |
| Embedding | X | X | X | O | X |
| Rule Engine | X | X | X | X | X |
| Sensor | X | O | O | X | X |
| Notification | X | X | X | X | X |
| DatasourceGenerator | O | X | X | X | X |

Auth(구 Auth+User)는 서비스 통합으로 테이블이 하나로 줄었습니다.
Rule Engine Service는 자체 Database가 없으며 MQTT/RabbitMQ로만 동작합니다.
Sensor Service는 Rule Engine Service가 RabbitMQ로 전달한 데이터를 Redis/InfluxDB에 저장합니다.
(한때 Rule Engine과 Sensor를 하나로 통합했었지만, 저장·조회 책임의 크기가 달라 다시 분리했습니다.)

---

# 데이터 흐름

DatasourceGenerator

↓

MQTT

↓

Rule Engine Service (수신·검증·규칙평가 → 자동 제어)

↓

RabbitMQ (EnvironmentMeasuredEvent)

↓

Sensor Service

├── Redis 저장 (최신값)
└── InfluxDB 저장 (이력)

↓

AI Service (주간/월간 리포트 요청 시 조회)

↓

Embedding Service

↓

Elasticsearch

↓

LLM

↓

Client

센서 데이터 수신·규칙평가(Rule Engine Service)와 저장·조회(Sensor Service)는 서로 다른 서비스이며,
RabbitMQ(EnvironmentMeasuredEvent)로만 연결됩니다.
