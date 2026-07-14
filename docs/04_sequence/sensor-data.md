# 센서 데이터 처리 시퀀스

## 개요

센서에서 생성된 환경 데이터를 실시간으로 수집하고 저장하는 과정입니다.

수신된 데이터는

- Redis(현재 상태)
- InfluxDB(시계열 데이터)

에 각각 저장되며, 대시보드와 AI 분석에 활용됩니다.

---

# Sequence

```text
Sensor

↓

Datasource Service

↓

MQTT Broker

↓

Rule Engine

↓

RabbitMQ

↓

Sensor Service

├── Redis 저장
└── InfluxDB 저장

↓

Dashboard

↓

현재 상태 조회 (Redis)

↓

차트 조회 (InfluxDB)
```

---

# 상세 과정

## 1. 센서 데이터 생성

센서가 환경 정보를 측정합니다.

```json
{
    "temperature":22.4,
    "humidity":91.5,
    "co2":810,
    "light":420
}
```

---

## 2. Datasource Service

센서 데이터를 수신합니다.

↓

MQTT Broker로 Publish합니다.

Topic

```
sensor/{cultivationId}
```

---

## 3. Rule Engine

MQTT Topic을 Subscribe합니다.

↓

데이터 검증

↓

EnvironmentMeasuredEvent 생성

↓

RabbitMQ Publish

---

## 4. Sensor Service

RabbitMQ를 Subscribe합니다.

↓

센서 데이터를 저장합니다.

---

## 5. Redis 저장

최신 환경 데이터를 저장합니다.

Key

```
cultivation:{cultivationId}:current
```

Value

```json
{
    "temperature":22.4,
    "humidity":91.5,
    "co2":810,
    "light":420,
    "updatedAt":"2026-08-15T12:30:00"
}
```

기존 데이터는 덮어씁니다.

---

## 6. InfluxDB 저장

Measurement

```
environment
```

Tag

- cultivationId
- sensorId

Field

- temperature
- humidity
- co2
- light

Timestamp

```
2026-08-15T12:30:00Z
```

---

## 7. Dashboard 조회

사용자가 대시보드를 조회합니다.

↓

Sensor Service

↓

Redis 조회

↓

현재 환경 반환

---

## 8. 차트 조회

사용자가 기간별 차트를 요청합니다.

↓

Sensor Service

↓

InfluxDB 조회

↓

차트 데이터 반환

---

# 사용 Database

## Redis

최신 환경 데이터 저장

---

## InfluxDB

센서 이력 저장

---

# MQTT

Publish

```
sensor/{cultivationId}
```

---

# RabbitMQ

Publish

```
EnvironmentMeasuredEvent
```

Subscribe

```
Sensor Service
```

---

# OpenFeign

사용하지 않습니다.

센서 데이터 처리는 비동기(Event-Driven) 방식으로 처리합니다.

---

# Dashboard

## 현재 환경

조회 대상

```
Redis
```

응답 예시

```json
{
    "temperature":22.4,
    "humidity":91.5,
    "co2":810,
    "light":420
}
```

---

## 차트

조회 대상

```
InfluxDB
```

기간

- 최근 1시간
- 최근 24시간
- 최근 7일
- 최근 30일

---

# 예외 상황

- MQTT Broker 연결 실패
- RabbitMQ 장애
- Redis 저장 실패
- InfluxDB 저장 실패
- Dashboard 조회 실패

---

# 고려 사항

- Redis에는 항상 최신 데이터만 유지합니다.
- InfluxDB에는 모든 센서 데이터를 저장합니다.
- Dashboard는 현재 상태와 이력을 각각 다른 저장소에서 조회합니다.
- AI 분석은 Redis가 아닌 InfluxDB 데이터를 기반으로 수행합니다.