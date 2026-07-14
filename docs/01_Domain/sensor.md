# Sensor Service

## 역할

Sensor Service는 Rule Engine으로부터 전달받은 센서 데이터를 저장하고,
실시간 환경 조회 및 통계 데이터를 제공하는 서비스입니다.

실시간 대시보드를 위해 최신 센서 데이터는 Redis에 저장하고,
장기 분석 및 통계 처리를 위해 모든 센서 데이터는 InfluxDB에 저장합니다.

또한 주간 및 월간 데이터를 집계하여 AI Service에 제공합니다.

---

# 책임

- 센서 데이터 저장
- 최신 환경 데이터 캐싱
- 실시간 환경 조회
- 환경 통계 조회
- 차트 데이터 제공
- 주간 데이터 집계
- 월간 데이터 집계
- AI 리포트 데이터 제공

---

# 주요 기능

## 센서 데이터 저장

Rule Engine으로부터 전달받은 센서 데이터를 저장합니다.

### 저장 순서

1. Redis에 최신 센서 데이터 저장
2. InfluxDB에 시계열 데이터 저장

---

## 현재 환경 조회

Redis에 저장된 최신 센서 데이터를 조회합니다.

조회 정보

- Temperature
- Humidity
- CO₂
- Light

실시간 대시보드는 Redis 데이터를 사용합니다.

---

## 환경 통계 조회

기간별 환경 통계를 제공합니다.

예시

- 평균 온도
- 평균 습도
- 평균 CO₂
- 평균 조도
- 최대값
- 최소값

통계 데이터는 InfluxDB에서 조회합니다.

---

## 차트 데이터 조회

센서 데이터를 그래프로 제공합니다.

지원 차트

- 온도
- 습도
- CO₂
- 조도

차트 데이터는 InfluxDB에서 조회합니다.

---

## 주간 데이터 집계

Scheduler를 통해

- 평균 온도
- 평균 습도
- 평균 CO₂
- 평균 조도
- 환경 유지율
- 자동 제어 횟수

를 집계하여 AI Service에 전달합니다.

---

## 월간 데이터 집계

월간 센서 데이터를 집계하여 AI 리포트 생성을 지원합니다.

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

---

# Database

## InfluxDB

### Measurement

environment

### Tag

- cultivationId
- sensorId

### Field

- temperature
- humidity
- co2
- light

### Timestamp

- measuredAt

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

TTL은 설정하지 않습니다.

항상 최신 데이터로 덮어쓰기 합니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### AI Service

- 주간 리포트 생성 요청
- 월간 리포트 생성 요청

---

## 호출받는 서비스

### Rule Engine Service

환경 데이터 저장

### API Gateway

현재 환경 조회

### Cultivation Service

현재 환경 조회

---

# RabbitMQ

## Subscribe Event

### EnvironmentControlEvent

Rule Engine에서 발행한 환경 이벤트를 구독합니다.

수신 내용

- 센서 데이터
- 자동 제어 결과

---

# Scheduler

## Weekly Scheduler

매주 환경 데이터를 집계하여 AI Service로 전달합니다.

---

## Monthly Scheduler

매월 환경 데이터를 집계하여 AI Service로 전달합니다.

---

# Sequence

Datasource Service

↓

MQTT Broker

↓

Rule Engine

↓

RabbitMQ

↓

Sensor Service

├── Redis
│     └── 최신 센서 데이터 저장
│
└── InfluxDB
└── 시계열 데이터 저장

↓

API Gateway

↓

Dashboard

(현재 상태 조회는 Redis 사용)

↓

AI Service

(주간/월간 분석은 InfluxDB 사용)

---

# 예외 상황

- Redis 저장 실패
- InfluxDB 저장 실패
- RabbitMQ 연결 실패
- 데이터 집계 실패
- AI Service 호출 실패

---

# 추후 개발 예정

- 이상 데이터 탐지
- 환경 예측
- Grafana 연동
- Redis Pub/Sub 기반 실시간 대시보드