# 센서 데이터 처리 시퀀스

## 개요

센서에서 생성된 환경 데이터를 실시간으로 수집하고 저장하는 과정입니다.

Rule Engine Service가 MQTT 수신부터 검증, Redis(현재 상태)/InfluxDB(시계열 데이터) 저장까지
하나의 서비스에서 처리합니다. (기존에는 Rule Engine이 검증 후 RabbitMQ로 별도의
Sensor Service에 저장을 위임했으나, 서비스 통합으로 내부 처리로 단순화되었습니다.)

저장된 데이터는 대시보드와 AI 분석에 활용됩니다.

---

# Sequence

```text
Sensor

↓

DatasourceGenerator

↓

MQTT Broker

↓

Rule Engine Service

├── 데이터 검증
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

## 2. DatasourceGenerator

센서 데이터를 수신합니다.

↓

MQTT Broker로 Publish합니다.

Topic

```
sensor/{cultivationId}
```

---

## 3. Rule Engine Service 수신 및 검증

MQTT Topic을 Subscribe합니다.

↓

데이터 검증 (수신 주기, 값 범위)

↓

같은 서비스 내부에서 즉시 저장 처리

---

## 4. Redis 저장

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

## 5. InfluxDB 저장

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

## 6. Dashboard 조회

사용자가 대시보드를 조회합니다.

↓

Rule Engine Service

↓

Redis 조회

↓

현재 환경 반환

---

## 7. 차트 조회

사용자가 기간별 차트를 요청합니다.

↓

Rule Engine Service

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

Subscribe

```
sensor/{cultivationId}
```

DatasourceGenerator가 발행한 데이터를 Rule Engine Service가 직접 구독하여 검증과 저장까지 처리합니다.

---

# RabbitMQ

사용하지 않습니다.

기존에는 검증 후 EnvironmentMeasuredEvent를 RabbitMQ로 발행해 Sensor Service에 전달했으나,
서비스 통합으로 별도 이벤트 없이 내부에서 바로 저장합니다.

---

# OpenFeign

사용하지 않습니다.

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
- Redis 저장 실패
- InfluxDB 저장 실패
- Dashboard 조회 실패

---

# 고려 사항

- Redis에는 항상 최신 데이터만 유지합니다.
- InfluxDB에는 모든 센서 데이터를 저장합니다.
- Dashboard는 현재 상태와 이력을 각각 다른 저장소에서 조회하지만, 조회 창구는 Rule Engine Service 하나입니다.
- AI 분석은 Redis가 아닌 InfluxDB 데이터를 기반으로 수행합니다.
