# Sensor Service

## 역할

Sensor Service는 Rule Engine Service로부터 전달받은 센서 측정값을 저장하고,
현재 환경/통계/차트/주간·월간 리포트 데이터를 조회할 수 있도록 제공하는 서비스입니다.

> ℹ️ **변경 이력**: 한때 Rule Engine Service와 하나로 통합하는 방안을 검토했었지만,
> 저장·조회 책임(Redis/InfluxDB, 통계·차트 API, 주간/월간 리포트 집계)의 크기와 변경 주기가
> 규칙 평가/자동 제어 로직과 달라 다시 별도 서비스로 분리했습니다.
> Rule Engine Service와는 RabbitMQ(EnvironmentMeasuredEvent)로만 연결되며, 직접 호출하지 않습니다.

---

# 책임

- 센서 데이터 저장 (Redis 최신값, InfluxDB 이력)
- 실시간 환경 조회
- 환경 통계 / 차트 데이터 제공
- 주간 / 월간 데이터 집계 및 AI Service 전달

센서 수신, 검증, 규칙 평가, 자동 제어는 Rule Engine Service의 책임입니다.

---

# 주요 기능

## 센서 데이터 저장

Rule Engine Service가 RabbitMQ로 발행한 EnvironmentMeasuredEvent를 구독하여 저장합니다.

1. Redis에 최신 센서 데이터 저장
2. InfluxDB에 시계열 데이터 저장

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

TTL은 설정하지 않으며 항상 최신 데이터로 덮어씁니다.

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

Rule Engine Service가 검증을 마친 센서 측정값을 발행하면 구독하여 저장합니다.

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

RabbitMQ Publish (EnvironmentMeasuredEvent)

↓

Sensor Service

├── Redis 저장 (최신 데이터)
└── InfluxDB 저장 (이력)

↓

Dashboard

↓

현재 상태 조회 (Redis) / 차트 조회 (InfluxDB)

---

# 예외 상황

- RabbitMQ 구독 실패
- Redis 저장 실패
- InfluxDB 저장 실패
- 조회 데이터 없음
- 잘못된 period 값
- 데이터 집계 실패
- AI Service 호출 실패

---

# 추후 개발 예정

- 데이터 압축/보관 주기 정책
- Grafana 연동
- 이상 데이터 탐지 고도화
