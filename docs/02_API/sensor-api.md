# Sensor API

## 개요

Sensor Service에서 제공하는 REST API 명세입니다.

Rule Engine Service로부터 RabbitMQ(EnvironmentMeasuredEvent)로 전달받아 저장한 센서 측정값을
현재값/통계/차트/주간·월간 리포트 형태로 조회할 수 있습니다.

센서 "장치" 자체의 등록/관리는 DatasourceGenerator API를, MQTT 수신·규칙 평가·자동 제어는
Rule Engine Service(REST API 없음, rule-Engine.md 참고)를 확인하세요.

Base URL

```
/api/v1/sensors
```

인증 방식

```
Bearer JWT
```

⚠️ DatasourceGenerator도 `/api/v1/sensors`로 시작하는 경로(장치 등록)를 사용합니다.
Gateway 라우팅 시 HTTP Method와 세부 경로만으로 두 서비스를 구분하기 어려우므로,
서비스 식별을 위한 경로 컨벤션 확정이 필요합니다. (datasource-generator-api.md 참고)

---

# 현재 환경 조회

## GET /current

### Query Parameter

```
cultivationId (required)
```

---

### Process

Sensor Service

↓

Redis 조회

```
cultivation:{cultivationId}:current
```

---

### Response

```json
{
    "temperature": 22.4,
    "humidity": 91.2,
    "co2": 810,
    "light": 430,
    "updatedAt": "2026-08-15T12:30:00"
}
```

---

# 환경 통계 조회

## GET /statistics

### Query Parameter

```
cultivationId (required)
period (required) - 예: 1h, 24h, 7d, 30d
```

---

### Process

Sensor Service

↓

InfluxDB 조회 (기간별 집계)

---

### Response

```json
{
    "averageTemperature": 22.1,
    "averageHumidity": 90.3,
    "averageCo2": 810,
    "averageLight": 430,
    "maxTemperature": 24.5,
    "minTemperature": 20.2,
    "environmentMaintainRate": 95
}
```

---

# 차트 데이터 조회

## GET /chart

### Query Parameter

```
cultivationId (required)
period (required) - 예: 1h, 24h, 7d, 30d
metric (required) - temperature | humidity | co2 | light
```

---

### Response

```json
{
    "metric": "humidity",
    "points": [
        { "timestamp": "2026-08-15T12:00:00", "value": 91.2 },
        { "timestamp": "2026-08-15T12:05:00", "value": 90.8 }
    ]
}
```

---

# 주간 데이터 조회

## GET /report/weekly

내부적으로 AI Service의 리포트 생성에 사용되는 집계 데이터를 반환합니다.

### Query Parameter

```
cultivationId (required)
```

---

### Response

```json
{
    "averageTemperature": 22.1,
    "averageHumidity": 90.3,
    "averageCo2": 810,
    "averageLight": 430,
    "environmentMaintainRate": 95,
    "autoControlCount": 6
}
```

---

# 월간 데이터 조회

## GET /report/monthly

### Query Parameter

```
cultivationId (required)
```

응답 구조는 주간 데이터 조회와 동일하며 집계 기간만 다릅니다.

---

# Error Code

| Code | Description |
|------|-------------|
| S001 | 존재하지 않는 재배 |
| S002 | 조회 데이터 없음 |
| S003 | 잘못된 period 값 |
| S004 | Redis 조회 실패 |
| S005 | InfluxDB 조회 실패 |
| S006 | RabbitMQ 구독 실패 |

---

# OpenFeign

호출받는 서비스

```
AI Service (센서 데이터 조회, 주간/월간 데이터 조회)
Cultivation Service (현재 센서 상태 조회, 환경 통계 조회)
```

호출하는 서비스

```
없음
```

Rule Engine Service와는 REST/OpenFeign으로 직접 통신하지 않으며, RabbitMQ(EnvironmentMeasuredEvent) 구독으로만 연결됩니다.

---

# RabbitMQ

Subscribe

```
EnvironmentMeasuredEvent (Rule Engine Service 발행)
```

수신 시 Redis(최신값)/InfluxDB(이력)에 저장합니다.

---

# Event

발행하는 이벤트는 없습니다.
