# Database Overview

## 데이터베이스 구성

본 프로젝트는 데이터의 특성에 따라 서로 다른 데이터베이스를 사용합니다.

| Database | 용도 | 사용 서비스 |
|----------|------|------------|
| PostgreSQL | 관계형 데이터 저장 | Auth, Cultivation, DatasourceGenerator |
| Redis | 캐시 및 임시 데이터 | Auth, AI, Rule Engine, Sensor |
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

버섯 재배 정보를 관리합니다. 공공데이터 기반 5종 버섯의 최적 환경 범위(mushroom_reference)도 함께 보관하며,
재배 생성 시 이 테이블을 조회해 AI/Embedding 호출 없이 추천값을 보여줍니다.

### Table

- mushroom_reference (전역 참조 데이터, cultivation과 무관)
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

### AI 챗봇 응답 Cache

```
ai:{hash}
```

TTL 24시간

---

### AI 생육 분석 결과 Cache

```
ai:{cultivationId}:analysis
```

TTL 6시간

---

### AI 리포트 Cache

```
report:{cultivationId}:{period}
```

TTL 24시간

자세한 키/값 구조는 [redis.md](./redis.md), [ai-api.md](../02_API/ai-api.md) 참고.

---

## Rule Engine

### 목표 환경 범위 캐시

```
cultivation:{cultivationId}:range
```

Cultivation Service가 발행하는 EnvironmentRangeUpdatedEvent를 구독해 write-through로 갱신하며,
TTL(24시간) 만료나 서비스 재시작 등으로 캐시가 없을 때만 Cultivation Service를 OpenFeign으로 호출합니다.

---

## Sensor

### Current Environment

```
cultivation:{cultivationId}:current
```

Rule Engine Service가 RabbitMQ(EnvironmentMeasuredEvent)로 전달한 데이터를 Sensor Service가 저장합니다.
이벤트는 매초 발행되며 Redis는 매번 그대로 갱신합니다(값 1건만 유지하는 덮어쓰기 구조라 용량 영향 없음).

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
매초 들어오는 이벤트를 InfluxDB에 그대로 다 기록하지 않고, 재배별로 10초 간격으로 스로틀링하여 저장합니다.
(자세한 내용은 [influxdb.md](./influxdb.md) 참고)

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
| Rule Engine | X | O | X | X | X |
| Sensor | X | O | O | X | X |
| Notification | X | X | X | X | X |
| DatasourceGenerator | O | X | X | X | X |

Auth(구 Auth+User)는 서비스 통합으로 테이블이 하나로 줄었습니다.
Rule Engine Service는 PostgreSQL/InfluxDB 같은 영구 저장소가 없으며, Redis는 목표 환경 범위
캐시 전용으로만 사용합니다(측정값 저장이 아님).
Sensor Service는 Rule Engine Service가 RabbitMQ로 전달한 데이터를 Redis/InfluxDB에 저장합니다.
(한때 Rule Engine과 Sensor를 하나로 통합했었지만, 저장·조회 책임의 크기가 달라 다시 분리했습니다.)

---

# 데이터 흐름

DatasourceGenerator

↓

MQTT

↓

Rule Engine Service (수신·검증 → Redis 캐시에서 목표 환경 범위 조회 → 규칙평가 → 자동 제어)

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

---

# 목표 환경 범위 캐시 흐름

Cultivation Service

↓

environment_setting 생성/수정

↓

RabbitMQ (EnvironmentRangeUpdatedEvent)

↓

Rule Engine Service

↓

Redis 저장 (cultivation:{cultivationId}:range, write-through)

캐시가 없을 때(TTL 만료, 재시작 직후 등)만 Rule Engine Service가 Cultivation Service를
OpenFeign으로 직접 호출해 값을 채웁니다.
