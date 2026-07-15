# Rule Engine Service

## 역할

Rule Engine Service는 MQTT로 수신한 센서 데이터를 검증·저장하고, 규칙(Rule) 기반으로 자동 제어를 수행하며,
실시간/통계성 환경 데이터 조회까지 제공하는 서비스입니다.

기존에는 "센서 수신(Collector)", "규칙 판단·자동 제어(Rule Engine)", "저장·조회(Sensor Service)"가
각각 별도 서비스/역할로 나뉘어 있었으나, MQTT 수신부터 규칙 평가, Redis/InfluxDB 저장까지가
사실상 하나의 파이프라인이라 서비스 간 호출(RabbitMQ Publish/Subscribe) 없이 하나의 서비스 내부 로직으로 처리하도록 통합했습니다.

Notification Service처럼 완전히 분리된 다른 서비스에 이상 상황을 알릴 때만 RabbitMQ를 사용합니다.

---

# 책임

- MQTT 센서 데이터 수신 (Collector)
- 센서 데이터 검증
- 환경 상태 분석 / 규칙(Rule) 평가
- 장치 자동 제어
- 센서 데이터 저장 (Redis 최신값, InfluxDB 이력)
- 실시간 환경 조회
- 환경 통계 / 차트 데이터 제공
- 주간 / 월간 데이터 집계
- 센서 오류 / 연결 해제 감지
- 이상 상황 이벤트 발행 (Notification Service용)

---

# 주요 기능

## 센서 데이터 수신 및 검증

MQTT Broker로부터 센서 데이터를 수신합니다.

수신 항목

- Temperature
- Humidity
- CO₂
- Light

---

## 규칙 평가 및 자동 제어

현재 센서 값과 목표 환경(Cultivation Service에 저장된 환경 설정)을 비교하여 규칙을 평가하고,
필요 시 장치를 자동으로 제어합니다.

제어 대상

- 가습기
- 환풍기
- 냉각기
- 히터
- LED 조명

예시

```
현재 습도 < 목표 습도

↓

가습기 ON
```

---

## 센서 데이터 저장

수신/제어 결과를 저장합니다.

1. Redis에 최신 센서 데이터 저장
2. InfluxDB에 시계열 데이터 저장

이전에는 RabbitMQ(EnvironmentControlEvent)를 통해 별도 서비스(Sensor Service)로 전달한 뒤 저장했지만,
이제는 같은 서비스 내부 로직이라 즉시 저장합니다.

---

## 현재 환경 조회

Redis에 저장된 최신 센서 데이터를 조회합니다. 실시간 대시보드는 이 데이터를 사용합니다.

---

## 환경 통계 / 차트 조회

InfluxDB에서 기간별 통계와 차트 데이터를 조회합니다.

- 평균/최대/최소 온도·습도·CO₂·조도
- 시간별 변화 추이

---

## 주간 / 월간 데이터 집계

Scheduler를 통해 주간/월간 환경 데이터를 집계하여 AI Service에 제공합니다.

---

## 센서 오류 감지

수신 주기와 데이터 유효성을 검사하여 센서 오류(OFFLINE/ERROR)를 감지하고,
DatasourceGenerator의 센서 상태를 갱신하도록 이벤트를 발행합니다.

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

장치 자동 제어, 센서 데이터 수신 자체는 REST API로 노출하지 않으며 MQTT/내부 로직으로만 동작합니다.

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

### Cultivation Service

- 재배별 목표 환경(environment_setting) 조회 (규칙 평가에 사용)

---

## 호출받는 서비스

### MQTT Broker

- DatasourceGenerator가 발행한 센서 데이터 구독

### API Gateway

- 현재 환경 / 통계 / 차트 조회

### Cultivation Service

- 현재 센서 상태 조회
- 환경 통계 조회

---

# MQTT

## Subscribe Topic

```
sensor/+
```

수신 데이터

```json
{
  "sensorId": 1,
  "temperature": 22.4,
  "humidity": 88.1,
  "co2": 1050,
  "light": 420,
  "measuredAt": "2026-08-15T12:30:00"
}
```

---

# RabbitMQ

Rule Engine Service는 자기 자신(구 Sensor Service 역할)을 위해서는 더 이상 RabbitMQ를 쓰지 않습니다.
Notification Service처럼 완전히 분리된 서비스에 알릴 때만 사용합니다.

## Publish Event

### EnvironmentControlEvent

자동 제어가 발생했을 때 발행합니다.

```json
{
  "cultivationId": 3,
  "device": "HUMIDIFIER",
  "action": "ON",
  "reason": "Humidity below target",
  "timestamp": "2026-08-15T12:31:00"
}
```

구독 서비스: Notification Service

---

### SensorErrorEvent

센서 오류/연결 해제가 감지되었을 때 발행합니다.

구독 서비스: DatasourceGenerator (센서 상태 갱신), Notification Service (알림)

---

# Scheduler

## Weekly Scheduler

매주 환경 데이터를 집계하여 AI Service로 전달합니다.

---

## Monthly Scheduler

매월 환경 데이터를 집계하여 AI Service로 전달합니다.

---

# 자동 제어 예시

| 조건 | 제어 |
|------|------|
| Temperature ↑ | 냉각팬 ON |
| Temperature ↓ | 히터 ON |
| Humidity ↓ | 가습기 ON |
| Humidity ↑ | 제습기 ON |
| CO₂ ↑ | 환풍기 ON |
| Light ↓ | LED ON |

---

# Sequence

## 센서 데이터 처리 및 자동 제어

DatasourceGenerator

↓

MQTT Publish

↓

MQTT Broker

↓

Rule Engine Service

├── 규칙 평가 → 장치 자동 제어
├── Redis 저장 (최신 데이터)
└── InfluxDB 저장 (이력)

↓

RabbitMQ (필요 시)

↓

Notification Service

---

# 예외 상황

- MQTT Broker 연결 실패
- 규칙 평가 실패
- 장치 제어 실패
- 잘못된 센서 데이터
- Redis 저장 실패
- InfluxDB 저장 실패
- RabbitMQ 전송 실패 (Notification 알림용)
- 데이터 집계 실패
- AI Service 호출 실패

---

# 추후 개발 예정

- 사용자 정의 Rule 지원
- Rule 우선순위 설정
- Rule 활성화/비활성화
- Rule 시뮬레이션 기능
- AI 기반 Rule 자동 생성
- 이상 데이터 탐지 고도화
- 환경 예측
- Grafana 연동
