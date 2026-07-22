# 센서 데이터 처리 시퀀스

## 개요

센서에서 생성된 환경 데이터를 실시간으로 수집하고 저장하는 과정입니다.

Rule Engine Service가 MQTT 수신과 검증을 담당하고, RabbitMQ(`EnvironmentMeasuredEvent`)로
전달하면 Sensor Service가 이를 구독해 Redis(현재 상태)/InfluxDB(시계열 데이터)에
저장합니다.

센서는 1초 주기로 값을 보내며, `EnvironmentMeasuredEvent`도 매초 발행됩니다. 다만
Sensor Service는 Redis와 InfluxDB를 다른 주기로 처리합니다 — **Redis는 매초 그대로
갱신**하고, **InfluxDB는 재배별로 10초 이상 지났을 때만 기록**합니다. 재배실 환경은
물리적으로 초 단위로 급변하지 않으므로 10초 해상도로도 이력 데이터 품질에 문제가 없으며,
InfluxDB 저장 용량을 약 1/10로 줄일 수 있습니다. 대시보드가 보여주는 "현재값"은 Redis
기준이라 체감 실시간성에는 영향이 없습니다.

저장된 데이터는 대시보드와 AI 분석에 활용됩니다.

---

# Sequence

```text
Sensor (1초 주기)
↓
DatasourceGenerator
↓
MQTT Broker
↓
Rule Engine Service
↓
데이터 검증
↓
RabbitMQ (EnvironmentMeasuredEvent, 매초 발행)
↓
Sensor Service
├── Redis 저장 (매초, 항상)
└── InfluxDB 저장 (재배별 10초 이상 경과했을 때만)
↓
Dashboard
├── 현재 상태 조회 (Redis, 매초 최신)
└── 차트 조회 (InfluxDB, 10초 간격 이력)
```

---

# 상세 과정

## 1. 센서 데이터 생성

```json
{
    "temperature": 22.4,
    "humidity": 91.5,
    "co2": 810,
    "light": 420
}
```

---

## 2. DatasourceGenerator

센서 데이터를 MQTT Broker로 Publish합니다.

Topic: `sensor/{deviceEui}`

---

## 3. Rule Engine Service 수신 및 검증

MQTT Topic을 Subscribe해 수신 주기와 값 범위를 검증합니다.

---

## 4. RabbitMQ Publish

```json
{
    "cultivationId": 3,
    "deviceEui": "24e124128c067999",
    "temperature": 22.4,
    "humidity": 91.5,
    "co2": 810,
    "light": 420,
    "measuredAt": "2026-08-15T12:30:00"
}
```

발행 이벤트: `EnvironmentMeasuredEvent`

---

## 5. Sensor Service 저장

RabbitMQ Subscribe → 이벤트를 받을 때마다 Redis는 항상 저장하고, InfluxDB는 스로틀링
여부를 판단합니다.

---

## 6. Redis 저장 (매초, 항상)

```
Key: cultivation:{cultivationId}:current
```

```json
{
    "temperature": 22.4,
    "humidity": 91.5,
    "co2": 810,
    "light": 420,
    "updatedAt": "2026-08-15T12:30:00"
}
```

기존 데이터는 덮어씁니다. 이벤트를 받을 때마다(매초) 항상 갱신합니다.

---

## 7. InfluxDB 저장 (10초 스로틀링)

재배별 마지막 InfluxDB 기록 시각을 확인합니다.

```
마지막 기록: 2026-08-15T12:30:00
현재 이벤트: 2026-08-15T12:30:07
경과: 7초 → 10초 미만이므로 이번 이벤트는 건너뜀
```

10초 이상 경과한 경우에만 기록합니다.

```
Measurement: environment
Tag: cultivationId, deviceEui
Field: temperature, humidity, co2, light
Timestamp: 2026-08-15T12:30:00Z
```

기록 후 해당 재배의 "마지막 InfluxDB 기록 시각"을 갱신합니다.

---

## 8. Dashboard 조회 (현재값)

Sensor Service → Redis 조회 → 현재 환경 반환

---

## 9. 차트 조회 (이력)

Sensor Service → InfluxDB 조회 → 차트 데이터 반환

---

# 사용 Database

## Redis

최신 환경 데이터 저장 (Sensor Service, 매초 갱신)

## InfluxDB

센서 이력 저장 (Sensor Service, 재배별 10초 간격 스로틀링)

---

# MQTT

Subscribe

```
sensor/{deviceEui}
```

DatasourceGenerator가 발행한 데이터를 Rule Engine Service가 구독해 검증합니다.

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

Rule Engine Service(검증)와 Sensor Service(저장)는 RabbitMQ로만 연결되며 서로 직접
호출하지 않습니다.

---

# OpenFeign

사용하지 않습니다.

---

# Dashboard

## 현재 환경

조회 대상: Redis

```json
{ "temperature": 22.4, "humidity": 91.5, "co2": 810, "light": 420 }
```

## 차트

조회 대상: InfluxDB

기간: 최근 1시간 / 최근 24시간 / 최근 7일

---

# 예외 상황

- MQTT Broker 연결 실패
- RabbitMQ 발행/구독 실패
- Redis 저장 실패 / InfluxDB 저장 실패
- 서비스 재시작 직후 스로틀링 기준 시각 초기화로 인한 일시적 중복 기록 (최초 1회 정도,
  영향 미미)
- Dashboard 조회 실패

---

# 고려 사항

- Redis에는 항상 최신 데이터만 유지하며, 이벤트를 받을 때마다(매초) 갱신합니다.
- InfluxDB에는 모든 센서 데이터를 저장하지 않고, 재배별로 10초 간격으로 스로틀링해
  저장합니다.
- 재배실 환경은 물리적으로 초 단위로 급변하지 않으므로 10초 해상도로도 이력/통계/AI
  분석 품질에 문제가 없습니다.
- 자동 제어 반응 속도는 Rule Engine Service가 원본(매초) 데이터를 그대로 평가하므로
  저장 스로틀링과 무관하게 유지됩니다.
- Dashboard는 현재 상태와 이력을 각각 다른 저장소에서 조회하며, 조회 창구는 Sensor
  Service입니다.
- AI 분석은 Redis가 아닌 InfluxDB 데이터를 기반으로 수행합니다.
- Rule Engine Service(검증·규칙평가)와 Sensor Service(저장·조회)는 서로 다른 서비스이며
  RabbitMQ로만 연결됩니다.
