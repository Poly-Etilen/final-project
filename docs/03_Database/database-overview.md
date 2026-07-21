# Database Overview

## 데이터베이스 구성

본 프로젝트는 데이터의 특성에 따라 서로 다른 데이터베이스를 사용합니다.

| Database | 용도 | 사용 서비스 |
|----------|------|------------|
| PostgreSQL | 관계형 데이터 저장 | Auth, Cultivation, Notification, AI, Sensor |
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

버섯 재배 정보를 관리합니다. 공공데이터 기반 5종 버섯의 최적 환경 범위와 이름/특성/효능/재배
가이드 등 참조 데이터(mushroom_reference)도 함께 보관하며, 재배 생성 시 이 테이블을 조회해
AI/Embedding 호출 없이 추천값을 보여줍니다. 참조 데이터의 텍스트는 AI Service의 "버섯 가이드"
기능(RAG 컨텍스트)과 Embedding Service의 Elasticsearch 인덱스(MushroomReferenceUpdatedEvent로
동기화)에도 활용됩니다.

### Table

- mushroom_reference (전역 참조 데이터, cultivation과 무관)
- cultivation
- harvest (`is_embedded` 컬럼으로 "인사이트" 기능의 임베딩 여부 관리)
- photo

> ℹ️ **변경 이력**: `environment_setting`과 `sensor`는 팀 회의 결과 Sensor Service 소유의
> 별도 DB로 이관되어 더 이상 이 DB에 없습니다. 아래 "Sensor DB" 참고.

---

## Sensor DB

### 목적

센서 "장치" 메타데이터(sensor)와 사용자가 설정한 목표 환경(위험 한계값, environment_setting)을
관리합니다. Sensor Service가 이미 측정값(InfluxDB/Redis)과 통계/차트/리포트를 전담하고 있어,
장치 메타데이터와 목표 환경까지 함께 소유하는 것이 "database per service" 원칙에 더 맞는다고
판단했습니다. Sensor Service가 PostgreSQL을 갖는 것은 이번이 처음입니다.

### Table

- sensor (센서 장치 메타데이터, Cultivation DB에서 이전)
- environment_setting (목표 환경/위험 한계값 이력, Cultivation DB에서 이전)

두 테이블 모두 `cultivation_id`를 갖지만 DB가 분리되어 있어 DB 레벨 외래키(FK)는 걸지
않습니다(순수 값 참조). 자세한 내용은 [sensor-db.md](./sensor-db.md) 참고.

---

## Notification DB

### 목적

발송한 알림의 이력(목록 조회/읽음 처리용)을 관리합니다. 채널 발송(WebSocket/Telegram/
Discord) 자체는 여전히 RabbitMQ 이벤트 기반 비동기 처리이며, DB는 그 결과만 저장합니다.

### Table

- notification

자세한 내용은 [notification-db.md](./notification-db.md) 참고.

---

## AI DB

### 목적

AI 챗봇의 대화 이력(질문/답변), 생육 분석 이력, 일일 피드백 이력을 관리합니다. `ai:{hash}`/
`ai:{cultivationId}:analysis` Redis 캐시(빠른 재조회/중복 호출 방지용)와는 역할이 다릅니다 —
캐시는 성능 최적화용, 이 DB는 이력 조회용입니다.

### Table

- chat_message (챗봇 대화 이력)
- growth_record (생육 분석 결과 이력 — Vision 분석할 때마다 한 행씩 쌓이며, "일일 피드백"의 생육 추이 비교에 사용)
- daily_feedback (일일 피드백 이력 — Daily Scheduler가 매일 재배별로 생성)

자세한 내용은 [ai-db.md](./ai-db.md) 참고.

---

## DatasourceGenerator (DB 없음)

DatasourceGenerator는 별도의 PostgreSQL DB를 사용하지 않습니다. 센서 데이터 생성/발행에
필요한 최소 정보(`sensor_cache`: device_eui/cultivationId/sensorType)는 메모리(In-Memory)에서만
관리하며, Sensor Service가 발행하는 이벤트(평상시) + 서비스 시작 시 OpenFeign 전체 조회로
채워집니다. 별도 "데이터 소스" 엔티티도 없습니다. 위치 정보(place/location)는 Sensor DB의
`sensor` 테이블에 직접 저장되며, DatasourceGenerator는 알 필요가 없습니다. (자세한 내용은
[datasource-generator-db.md](./datasource-generator-db.md) 참고)

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
report:{cultivationId}:weekly
```

TTL 24시간. Weekly Scheduler(Sensor Service)가 매주 먼저 집계 데이터를 전달하면 AI Service가
리포트를 생성해 이 캐시를 채워둡니다(push). 재배 기간이 한 달을 넘지 않아 월간 리포트는
만들지 않습니다.

---

### 버섯 가이드 Cache

```
ai:mushroom:{mushroomType}:guide
```

TTL 7일. cultivationId가 아닌 mushroomType(5종 고정) 기준으로 캐싱합니다.

---

### 인사이트 Cache

```
ai:{cultivationId}:insight
```

TTL 24시간. 다른 AI 캐시(챗봇/리포트/가이드)와 달리 스케줄러가 미리 채워두지 않고, 사용자가
`GET /ai/insight`를 요청한 시점에 Embedding Service 검색 + LLM 요약을 거쳐 채워집니다. 유사
사례가 없어 고정 문구로 응답한 경우는 캐시하지 않습니다.

자세한 키/값 구조는 [redis.md](./redis.md), [ai-api.md](../02_API/ai-api.md) 참고.

---

## Rule Engine

### 목표 환경 범위 캐시

```
cultivation:{cultivationId}:range
```

Sensor Service가 발행하는 EnvironmentRangeUpdatedEvent를 구독해 write-through로 갱신하며,
TTL(24시간) 만료나 서비스 재시작 등으로 캐시가 없을 때만 Sensor Service를 OpenFeign으로 호출합니다.

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
- deviceEui

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
cultivation_insight
```

### Document (mushroom_environment)

- mushroomType (문서 ID로도 사용)
- mushroomNameKo / mushroomNameEn / mushroomScientificName
- tempMin / tempMax / humidityMin / humidityMax / co2Min / co2Max / lightMin / lightMax
- description
- characteristics / healthBenefits / cultivationGuide / additionalInfo
- embedding(Vector) — characteristics/healthBenefits/cultivationGuide/additionalInfo를 결합해 생성

Cultivation DB의 `mushroom_reference`와 1:1로 대응하며, `MushroomReferenceUpdatedEvent`로
동기화됩니다. 임베딩 벡터는 Cultivation DB(PostgreSQL)에는 저장하지 않고 이 인덱스에만
저장합니다.

### Document (cultivation_insight)

- cultivationId (문서 ID로도 사용)
- mushroomType / avgTemperature / avgHumidity / avgCo2 / avgLight
- growthScore / harvestWeight
- summary (자연어 요약 원문)
- embedding(Vector) — summary를 임베딩해 생성

"인사이트" 기능(같은 버섯 종류 + 유사한 온도로 재배했던 타인의 사례 기반 피드백)을 위한
인덱스입니다. `mushroom_environment`와 달리 5종 고정 데이터가 아니라, 완료된 재배가 끝날 때마다
AI Service의 Insight Batch Scheduler(00시, 임계치 20건)가 배치로 채워 넣는 계속 쌓이는
데이터입니다. 검색도 벡터 유사도가 아닌 mushroomType/avgTemperature 필터 기반입니다. (자세한
내용은 [elasticSearch.md](./elasticSearch.md), [insight.md](../04_sequence/insight.md) 참고)

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
| AI | O | O | X | X | O |
| Embedding | X | X | X | O | X |
| Rule Engine | X | O | X | X | X |
| Sensor | X | O | O | X | X |
| Notification | O | X | X | X | X |
| DatasourceGenerator | X | X | X | X | X |

Auth(구 Auth+User)는 서비스 통합으로 테이블이 하나로 줄었습니다.
Rule Engine Service는 PostgreSQL/InfluxDB 같은 영구 저장소가 없으며, Redis는 목표 환경 범위
캐시 전용으로만 사용합니다(측정값 저장이 아님).
Sensor Service는 Rule Engine Service가 RabbitMQ로 전달한 데이터를 Redis/InfluxDB에 저장합니다.
(한때 Rule Engine과 Sensor를 하나로 통합했었지만, 저장·조회 책임의 크기가 달라 다시 분리했습니다.)
DatasourceGenerator는 어떤 영구 저장소도 사용하지 않으며, `sensor_cache`는 메모리(In-Memory)에서만 관리합니다.
AI Service와 Notification Service는 각각 챗봇 대화 이력(`chat_message`)과 알림 이력(`notification`)
조회 기능이 추가되면서 처음으로 PostgreSQL을 갖게 되었습니다.

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

AI Service (Weekly Scheduler가 주간 집계 데이터를 push로 전달, 사용자 요청 시점이 아님)

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

Sensor Service

↓

environment_setting 생성/수정

↓

RabbitMQ (EnvironmentRangeUpdatedEvent)

↓

Rule Engine Service

↓

Redis 저장 (cultivation:{cultivationId}:range, write-through)

캐시가 없을 때(TTL 만료, 재시작 직후 등)만 Rule Engine Service가 Sensor Service를
OpenFeign으로 직접 호출해 값을 채웁니다.

> ℹ️ **변경 이력**: `environment_setting`이 Cultivation Service에서 Sensor Service로
> 이관되면서, 발행 주체와 fallback 호출 대상이 모두 Sensor Service로 바뀌었습니다.

---

# 센서 등록 캐시 동기화 흐름

## 평상시 (이벤트 기반)

Sensor Service

↓

sensor 생성/삭제 (PostgreSQL)

↓

RabbitMQ (SensorRegisteredEvent / SensorDeletedEvent)

↓

DatasourceGenerator

↓

메모리 캐시(sensor_cache) Upsert/삭제

센서 장치 CRUD의 원본(source of truth)은 Sensor DB의 `sensor`입니다. DatasourceGenerator의
`sensor_cache`는 "어떤 센서에 대해 MQTT 데이터를 생성/발행할지" 판단하기 위한 읽기 전용
캐시일 뿐이며, PostgreSQL이 아닌 메모리(In-Memory)에 보관됩니다.

## 서비스 시작 시 (재구성)

DatasourceGenerator 시작

↓

Sensor Service에 OpenFeign 호출 (`GET /api/v1/sensors`, 전체 센서 목록)

↓

메모리 캐시(sensor_cache) 일괄 채움

메모리 캐시이므로 재시작하면 비어 있습니다. 평상시에는 이벤트로만 갱신하지만, 시작 시점에는
전체 목록을 한 번에 받아와 복구합니다.

> ℹ️ **변경 이력**: `sensor` 테이블이 Cultivation Service에서 Sensor Service로 이관되면서,
> 이벤트 발행 주체와 재시작 시 조회 대상이 모두 Sensor Service로 바뀌었습니다.

---

# 버섯 참조 데이터 동기화 흐름 (Elasticsearch)

## 평상시 (이벤트 기반)

Cultivation Service

↓

관리자가 mushroom_reference 등록/수정 (PostgreSQL)

↓

RabbitMQ (MushroomReferenceUpdatedEvent)

↓

Embedding Service

↓

텍스트(characteristics/healthBenefits/cultivationGuide/additionalInfo) 임베딩 → Elasticsearch Upsert

## 전체 재생성 시 (드묾, 예: 임베딩 모델 교체)

Embedding Service

↓

Cultivation Service에 OpenFeign 호출 (`GET /api/v1/mushroom-references`, 전체 목록)

↓

전체 mushroomType 재임베딩 → Elasticsearch 일괄 Upsert

mushroom_reference의 원본(source of truth)은 Cultivation DB(PostgreSQL)이며, 임베딩 벡터는
PostgreSQL에 저장하지 않고 Elasticsearch에만 저장합니다(데이터 중복 방지). 버섯 종류가 5종으로
고정된 정적 데이터라 이 동기화는 매우 드물게 발생합니다.

---

# 인사이트 데이터 흐름 (Elasticsearch)

## 배치 임베딩 적재 (스케줄 기반, 임계치 도달 시에만)

AI Service (Insight Batch Scheduler, 매일 00시)

↓

Cultivation Service에 OpenFeign 호출 (미임베딩 harvest 건수 조회, 20건 미만이면 종료)

↓

Cultivation Service에 OpenFeign 호출 (미임베딩 목록 조회)

↓

Sensor Service에 OpenFeign 호출 (환경 평균 일괄 조회, 기간 가중 평균)

↓

AI Service 자체 growth_record와 병합 (생육 점수)

↓

Embedding Service에 OpenFeign 호출 (배치 임베딩 요청)

↓

Embedding Service: 자연어 요약 → Embedding Model → Elasticsearch 저장 (cultivation_insight)

↓

AI Service → Cultivation Service (harvest.is_embedded = TRUE 처리)

## 조회 (사용자 요청 시점)

Client (GET /ai/insight)

↓

AI Service

↓

Redis 캐시 미스 시 → Cultivation Service(버섯 종류 조회) + Sensor Service(환경 평균 조회) → Embedding Service(cultivation_insight 검색) → LLM 요약 → Redis 캐시 저장

`harvest.is_embedded`(Cultivation DB)의 원본은 Cultivation Service가 소유하며, AI Service는
워터마크를 별도로 관리하지 않고 단순 조회/갱신만 합니다. 배치 적재와 조회는 완전히 독립된
흐름이며, 조회는 배치가 실행되지 않은 날에도 언제든 가능합니다(그 시점까지 쌓인 데이터
기준으로 검색). 자세한 내용은 [insight.md](../04_sequence/insight.md) 참고.
