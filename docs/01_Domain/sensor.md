# Sensor Service

## 역할

Sensor Service는 Rule Engine Service로부터 전달받은 센서 측정값을 저장하고,
현재 환경/통계/차트/주간·월간 리포트 데이터를 조회할 수 있도록 제공하는 서비스입니다.

> ℹ️ **변경 이력**: 한때 Rule Engine Service와 하나로 통합하는 방안을 검토했었지만,
> 저장·조회 책임(Redis/InfluxDB, 통계·차트 API, 주간/월간 리포트 집계)의 크기와 변경 주기가
> 규칙 평가/자동 제어 로직과 달라 다시 별도 서비스로 분리했습니다.
> Rule Engine Service와는 RabbitMQ(EnvironmentMeasuredEvent)로만 연결되며, 직접 호출하지 않습니다.

> ℹ️ **변경 이력**: 센서가 1초 주기로 값을 보내는 경우, EnvironmentMeasuredEvent도 매초 발행되어
> InfluxDB에 그대로 다 기록하면 재배 1건 기준 한 달에 수백 MB, 2년 누적 시 상당한 용량이
> 필요해집니다. 반면 재배실 환경(온도/습도/CO₂/조도)은 물리적으로 초 단위로 급변하지 않으므로,
> **Redis(실시간 대시보드용)는 매초 그대로 갱신하고, InfluxDB(이력 저장용)만 10초 간격으로
> 스로틀링**하는 방식을 도입했습니다. 자동 제어 반응 속도(Rule Engine Service)에는 영향이 없습니다.

---

# 책임

- 센서 데이터 저장 (Redis 최신값 매초, InfluxDB 이력은 10초 간격 스로틀링)
- 실시간 환경 조회
- 환경 통계 / 차트 데이터 제공
- 주간 / 월간 데이터 집계 및 AI Service 전달

센서 수신, 검증, 규칙 평가, 자동 제어는 Rule Engine Service의 책임입니다.
Rule Engine Service는 매초 EnvironmentMeasuredEvent를 발행하며, 저장 빈도 조절(스로틀링)은
전적으로 Sensor Service의 책임입니다.

---

# 주요 기능

## 센서 데이터 저장 (Redis 매초 / InfluxDB 10초 스로틀링)

Rule Engine Service가 RabbitMQ로 발행한 EnvironmentMeasuredEvent를 구독하여 저장합니다.
이벤트는 매초 들어오지만, 두 저장소를 다른 주기로 처리합니다.

1. **Redis** — 이벤트를 받을 때마다 매번 최신 데이터로 덮어씁니다. (실시간 대시보드용, 매초 갱신)
2. **InfluxDB** — 재배별로 마지막 기록 시각을 확인하여, 10초 이상 지났을 때만 이번 값을 기록합니다.
   10초가 지나지 않았다면 이번 이벤트는 InfluxDB에는 기록하지 않고 건너뜁니다. (이력 저장용, 10초 간격)

```
EnvironmentMeasuredEvent 수신

↓

Redis 저장 (항상)

↓

마지막 InfluxDB 기록 시각 확인 (재배별)

↓

10초 이상 경과? ─ No → InfluxDB 저장 건너뜀
              │
              Yes
              ↓
         InfluxDB 저장 + 마지막 기록 시각 갱신
```

마지막 InfluxDB 기록 시각은 재배별로 애플리케이션 메모리에 관리합니다.
서비스 재시작 시 초기화되지만, 이 경우 최초 1회 정도만 스로틀링 없이 기록되는 정도라 문제되지 않습니다.

10초라는 값은 초기 기준이며, 필요 시 재배/센서 타입별로 다르게 조정할 수 있습니다.

---

## 현재 환경 조회

Redis에 저장된 최신 센서 데이터를 조회합니다. 실시간 대시보드는 이 데이터를 사용합니다.

---

## 환경 통계 / 차트 조회

InfluxDB에서 기간별 통계와 차트 데이터를 조회합니다.

- 평균/최대/최소 온도·습도·CO₂·조도
- 시간별 변화 추이
- 목표 환경 범위 대비 유지율 (Cultivation Service의 environment_setting 참고)

---

## 주간 / 월간 데이터 집계

Scheduler를 통해 주간/월간 환경 데이터를 집계하여 AI Service에 제공합니다.

---

# API

## 현재 환경 조회

GET /sensors/current

---

## 환경 통계 조회

GET /sensors/statistics

---

## 차트 데이터 조회

GET /sensors/chart

---

## 주간 데이터 조회

GET /sensors/report/weekly

---

## 월간 데이터 조회

GET /sensors/report/monthly

센서 데이터 수신, 규칙 평가, 자동 제어는 이 서비스가 아닌 Rule Engine Service가 담당하며 REST API로 노출하지 않습니다.

---

# Database

## InfluxDB

### Measurement

```
environment
```

### Tag

- cultivationId
- sensorId

### Field

- temperature
- humidity
- co2
- light

---

## Redis

최신 센서 데이터를 저장합니다.

### Key

```
cultivation:{cultivationId}:current
```

### Value

```json
{
  "temperature": 22.5,
  "humidity": 91.2,
  "co2": 820,
  "light": 430,
  "updatedAt": "2026-08-15T10:20:30"
}
```

TTL은 설정하지 않으며 EnvironmentMeasuredEvent를 받을 때마다(매초) 항상 최신 데이터로 덮어씁니다.
InfluxDB와 달리 스로틀링을 적용하지 않습니다 — Redis 덮어쓰기는 값이 늘어나는 게 아니라 항상 1건만
유지되므로 매초 갱신해도 저장 용량에 영향이 없습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### AI Service

- 주간 리포트 생성 요청
- 월간 리포트 생성 요청

---

## 호출받는 서비스

### API Gateway

- 현재 환경 / 통계 / 차트 조회

### AI Service

- 센서 데이터 / 통계 조회

### Cultivation Service

- 현재 센서 상태 조회
- 환경 통계 조회

Rule Engine Service와는 REST/OpenFeign으로 직접 통신하지 않으며, RabbitMQ 이벤트 구독으로만 연결됩니다.

---

# RabbitMQ

## Subscribe Event

### EnvironmentMeasuredEvent

Rule Engine Service가 검증을 마친 센서 측정값을 매초 발행하면 구독합니다.
Redis에는 매번 저장하고, InfluxDB에는 재배별 10초 스로틀링을 적용해 저장합니다.

```json
{
  "cultivationId": 3,
  "sensorId": 1,
  "temperature": 22.4,
  "humidity": 88.1,
  "co2": 1050,
  "light": 420,
  "measuredAt": "2026-08-15T12:30:00"
}
```

---

# Scheduler

## Weekly Scheduler

매주 InfluxDB 데이터를 집계하여 AI Service로 전달합니다.

---

## Monthly Scheduler

매월 InfluxDB 데이터를 집계하여 AI Service로 전달합니다.

---

# Sequence

## 센서 데이터 저장 및 조회

Rule Engine Service

↓

RabbitMQ Publish (EnvironmentMeasuredEvent, 매초)

↓

Sensor Service

├── Redis 저장 (매초, 항상)
└── InfluxDB 저장 (10초 이상 경과했을 때만)

↓

Dashboard

↓

현재 상태 조회 (Redis, 매초 최신) / 차트 조회 (InfluxDB, 10초 간격 이력)

---

# 예외 상황

- RabbitMQ 구독 실패
- Redis 저장 실패
- InfluxDB 저장 실패
- 서비스 재시작 직후 스로틀링 기준 시각 초기화로 인한 일시적 중복 기록 (최초 1회 정도, 영향 미미)
- 조회 데이터 없음
- 잘못된 period 값
- 데이터 집계 실패
- AI Service 호출 실패

---

# 추후 개발 예정

- InfluxDB 장기 보관 데이터에 대한 단계별 Downsampling (예: 최근 7일 원본 → 이후 1분/1시간 평균)
- 재배/센서 타입별 스로틀링 주기 차등 적용
- Grafana 연동
- 이상 데이터 탐지 고도화
