# Database Overview

## 데이터베이스 구성

본 프로젝트는 데이터의 특성에 따라 서로 다른 데이터베이스를 사용합니다.

| Database | 용도 | 사용 서비스 |
|----------|------|------------|
| PostgreSQL | 관계형 데이터 저장 | Auth, Cultivation, DatasourceGenerator |
| Redis | 캐시 및 임시 데이터 | Auth, AI, Rule Engine |
| InfluxDB | 시계열 센서 데이터 | Rule Engine |
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

## Rule Engine

### Current Environment

```
cultivation:{cultivationId}:current
```

(기존 Sensor Service가 담당하던 저장소이며, 서비스 통합으로 Rule Engine Service가 관리합니다.)

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

Rule Engine Service가 저장/조회를 전담합니다. (기존 Sensor Service 역할 포함)

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
| Rule Engine | X | O | O | X | X |
| Notification | X | X | X | X | X |
| DatasourceGenerator | O | X | X | X | X |

Auth(구 Auth+User), Rule Engine(구 Collector+RuleEngine+Sensor/Storage)은 서비스 통합으로 컬럼이 하나로 줄었습니다.

---

# 데이터 흐름

DatasourceGenerator

↓

MQTT

↓

Rule Engine Service

├── 규칙 평가 → 자동 제어
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

센서 데이터 수신부터 저장까지는 Rule Engine Service 하나가 전담하며,
서비스 경계를 넘는 지점(Notification 알림 등)에서만 RabbitMQ를 사용합니다.
