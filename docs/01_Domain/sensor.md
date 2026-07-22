# Sensor Service

## 역할

Sensor Service는 센서 측정값 저장/조회/통계, 센서 "장치" 메타데이터 관리, 사용자가
설정한 목표 환경(위험 한계값) 관리, 공공데이터 기반 버섯 참조 데이터 관리를 담당하는
서비스입니다. Rule Engine Service로부터 RabbitMQ로 측정값을 전달받아 Redis(실시간)와
InfluxDB(이력)에 저장합니다.

---

# 책임

- 센서 측정값 저장(Redis/InfluxDB)·조회·통계 (일일 피드백용 일간 통계 집계 포함)
- 센서 장치 등록/조회/삭제 (`sensor`, `sensor_type`)
- 목표 환경 범위 저장/조회/평균 계산 (`environment_setting`)
- 버섯 참조 데이터 관리 (`mushroom_reference`, `mushroom_reference_threshold`)

---

# 주요 기능

## 센서 데이터 저장

MQTT로 발행된 값을 Rule Engine Service가 수신·검증한 뒤 RabbitMQ
(`EnvironmentMeasuredEvent`, 매초)로 전달하면, Sensor Service가 구독해 저장합니다. Redis는
매초 그대로 갱신하고(현재값), InfluxDB는 재배별로 10초 간격으로 스로틀링해 저장합니다
(이력/통계용).

---

## 센서 장치 등록/조회/삭제

재배에 연결된 센서를 관리합니다. PK는 대리키 `id`이고 `device_eui`는 UNIQUE 제약을 가진
일반 컬럼입니다. 장치 하나가 여러 측정 항목(온도/습도/CO₂/조도)을 동시에 가질 수 있어
`sensor_type` 하위 테이블로 관리합니다. 삭제는 하드 삭제가 아니라 `is_deleted` 플래그로
처리해, 삭제된 센서라도 과거 `environment_setting` 이력이 계속 의미를 가질 수 있게
합니다.

---

## 목표 환경 저장/조회

사용자가 설정한 목표 환경을 범위(threshold_min~threshold_max)로, 측정 항목별 행으로
저장합니다. 부분 수정(예: 온도만)을 지원하며, 수정 시 UPDATE가 아니라 새 행을 INSERT해
이력을 함께 표현합니다. 저장할 때마다 `EnvironmentRangeUpdatedEvent`를 발행합니다 —
Rule Engine Service는 이 이벤트로 Redis 캐시를 갱신하고, Cultivation Service는 이
이벤트를 받아 재배를 `CREATED → RUNNING`으로 전환합니다.

---

## 버섯 참조 데이터 관리

공공데이터 기준 5종 버섯의 참조 데이터입니다. `mushroom_type` 코드(예: `OYSTER`)가
`cultivation.mushroom_type`과 연결되는 자연키 역할을 합니다. 이름/특성/효능/재배 가이드
같은 텍스트 컬럼은 AI Service의 챗봇/버섯 가이드 기능이 `mushroomType`으로 정확히
일치하는 한 건을 조회해 LLM 컨텍스트(RAG 원문)로 그대로 사용합니다. 버섯 종류가 5종
고정이라 별도의 검색 색인 없이 직접 조회로 충분합니다. 온도/습도/CO₂/조도 추천 범위는
`mushroom_reference_threshold`에 항목별로 저장됩니다.

---

# API

## 센서 등록/조회/삭제 (배치 포함)

POST /api/v1/sensors/cultivations/{cultivationId}/batch (내부용)

POST/GET/DELETE /api/v1/sensors/cultivations/{cultivationId}

---

## 목표 환경 저장/조회

PATCH /api/v1/sensors/cultivations/{cultivationId}/environment

GET /api/v1/sensors/cultivations/{cultivationId}/environment-history

GET /api/v1/sensors/cultivations/{cultivationId}/environment-average

POST /api/v1/sensors/environment-averages (내부용, 배치 조회)

---

## 센서 통계

GET /api/v1/sensors/cultivations/{cultivationId}/current

GET /api/v1/sensors/cultivations/{cultivationId}/stats (기간별 집계, AI Service의
일일 피드백은 최근 24시간 기준으로 호출)

---

## 버섯 참조 데이터 조회 (내부용)

GET /api/v1/mushroom-references/{mushroomType}

GET /api/v1/mushroom-references

---

# Database

Sensor Service는 하나의 PostgreSQL Database와 Redis/InfluxDB를 사용합니다.

## Table

- measurement_type (온도/습도/CO2/조도 4종 참조 테이블)
- sensor / sensor_type
- environment_setting
- mushroom_reference / mushroom_reference_threshold

자세한 내용은 [sensor-db.md](../03_Database/sensor-db.md) 참고.

---

# Redis

- 최신 센서 데이터 (`cultivation:{cultivationId}:current`, TTL 없음)

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Cultivation Service

- 쓰기 작업(센서 등록/삭제, 환경 설정 저장) 전 소유권 확인 (`GET /cultivations/{cultivationId}/owner`)

---

## 호출받는 서비스

### Cultivation Service

- 재배 생성 시 센서 배치 등록, 버섯 참조 데이터 조회

### Rule Engine Service

- 목표 환경 범위 캐시 미스 시 fallback 조회

### AI Service

- 환경 변경 이력/평균/일간 통계 조회(일일 피드백용), 버섯 참조 데이터 조회

### API Gateway

- 센서/환경/통계 관련 REST API 요청

---

# Event

## Publish

### SensorRegisteredEvent / SensorDeletedEvent

DatasourceGenerator가 구독해 `sensor_cache`(메모리)를 갱신합니다.

### EnvironmentRangeUpdatedEvent

Rule Engine Service(캐시 갱신), Cultivation Service(CREATED→RUNNING 전환)가 구독합니다.

---

## Subscribe

### EnvironmentMeasuredEvent

Rule Engine Service가 발행. Redis/InfluxDB에 저장합니다.

### SensorErrorEvent

Rule Engine Service가 발행. 센서 `status`를 갱신합니다.

### CultivationDeletedEvent

Cultivation Service가 발행. 해당 재배의 `sensor`/`environment_setting`을 정리합니다.

---

# Sequence

관련 시퀀스는 [sensor-data.md](../04_sequence/sensor-data.md),
[environment-control.md](../04_sequence/environment-control.md),
[sensor-error.md](../04_sequence/sensor-error.md) 참고.

---

# 예외 상황

- 중복된 device_eui 등록 시도
- 존재하지 않는 센서/재배
- 소유권 확인 실패 (Cultivation Service 호출 실패 포함)
- MQTT/RabbitMQ 수신 실패

---

# 추후 개발 예정

- 버섯 참조 데이터 관리 UI/관리자 API
