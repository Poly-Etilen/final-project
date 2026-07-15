# Database Overview

## 데이터베이스 구성

본 프로젝트는 데이터의 특성에 따라 서로 다른 데이터베이스를 사용합니다.

| Database | 용도 | 사용 서비스 |
|----------|------|------------|
| PostgreSQL | 관계형 데이터 저장 | Auth, User, Cultivation, Datasource |
| Redis | 캐시 및 임시 데이터 | Auth, AI, Sensor |
| InfluxDB | 시계열 센서 데이터 | Sensor |
| Elasticsearch | Vector Search | Embedding |
| MinIO | 생육 사진(이미지) 저장 | Cultivation, AI |

---

# PostgreSQL

## Auth DB

### 목적

사용자 인증 정보를 관리합니다.

### Table

- auth_user

---

## User DB

### 목적

사용자 프로필 정보를 관리합니다.

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

## Datasource DB

### 목적

센서 및 데이터 소스를 관리합니다.

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
| User | O | X | X | X | X |
| Cultivation | O | X | X | X | O |
| AI | X | O | X | X | O |
| Embedding | X | X | X | O | X |
| Datasource | O | X | X | X | X |
| Rule Engine | X | X | X | X | X |
| Sensor | X | O | O | X | X |
| Notification | X | X | X | X | X |

---

# 데이터 흐름

Datasource

↓

MQTT

↓

Rule Engine

↓

RabbitMQ

↓

Sensor

├── Redis

└── InfluxDB

↓

AI

↓

Embedding

↓

Elasticsearch

↓

LLM

↓

Client