# 환경 자동 제어 시퀀스

## 개요

센서에서 수집한 환경 데이터를 기반으로 Rule Engine Service가 목표 환경 범위와 비교하여
자동 제어 여부를 판단합니다.

목표 환경은 단일값이 아닌 범위(min~max)로 저장되어 있습니다. 현재값이 범위 안에 있으면 제어하지
않고, 범위를 벗어난 경우에만 장치를 제어합니다. 불필요하게 장치를 자주 켜고 끄는 것(채터링)을
막기 위한 히스테리시스 목적입니다.

Rule Engine Service는 MQTT 수신, 검증, 규칙 평가, 자동 제어까지 담당하고, 저장은 직접 하지 않습니다.
검증을 마친 데이터를 RabbitMQ(EnvironmentMeasuredEvent)로 발행하면 Sensor Service가 구독하여
Redis/InfluxDB에 저장합니다.

> ℹ️ **변경 이력**: 한때 Rule Engine과 저장(Sensor Service)을 하나의 서비스로 통합해 내부 로직으로
> 즉시 저장하는 방식을 검토했었지만, 책임 분리를 위해 다시 별도 서비스로 나누고 RabbitMQ로 연결했습니다.

제어가 필요한 경우 제어 명령을 생성하고, 사용자에게 알림을 전송합니다.

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

├── 목표 환경 범위 조회 (Cultivation Service, OpenFeign)
├── 범위 비교 (min/max)
└── 제어 여부 판단

↓

RabbitMQ

├── EnvironmentMeasuredEvent → Sensor Service (Redis/InfluxDB 저장)
└── EnvironmentControlEvent (제어 발생 시만) → Notification Service

↓

WebSocket
Telegram
Discord
```

---

# 상세 과정

## 1. 센서 데이터 측정

센서가 다음 데이터를 측정합니다.

- Temperature
- Humidity
- CO₂
- Light

예시

```json
{
    "temperature":20.5,
    "humidity":82,
    "co2":980,
    "light":310
}
```

---

## 2. MQTT Publish

DatasourceGenerator가

MQTT Broker로 데이터를 Publish합니다.

Topic

```
sensor/{cultivationId}
```

---

## 3. Rule Engine Service 수신

Rule Engine Service가

MQTT를 Subscribe합니다.

↓

환경 데이터를 수신하고 검증합니다.

---

## 4. 목표 환경 범위 조회

Rule Engine Service는

OpenFeign으로 Cultivation Service를 호출하여 현재 재배의 목표 환경 범위(environment_setting의 min~max)를 조회합니다.

예시

```
Temperature

temp_min 20.5℃ ~ temp_max 23.5℃
```

```
Humidity

humidity_min 85% ~ humidity_max 95%
```

이 범위는 Cultivation Service가 사용자의 단일 목표값(예: 온도 22℃)에 허용 오차를 적용해
저장해 둔 값입니다.

---

## 5. 범위 비교

현재값

↓

min ~ max 범위

↓

비교 수행

예시

```
현재 습도

82%
```

↓

```
범위

85% ~ 95%
```

↓

판정

```
82% < min(85%) → 범위 미달
```

---

## 6. 자동 제어 판단

Rule Engine Service

↓

규칙 확인

예시

```
Humidity < humidity_min

↓

Humidifier ON
```

또는

```
Temperature > temp_max

↓

Cooling Fan ON
```

현재값이 min ~ max 범위 안에 있으면 아무 것도 제어하지 않습니다. (이미 켜져 있던 장치는 OFF)

---

## 7. RabbitMQ Publish

Rule Engine Service는 두 종류의 이벤트를 발행합니다.

### EnvironmentMeasuredEvent (저장용, 매번 발행)

```json
{
    "cultivationId":3,
    "sensorId":1,
    "temperature":20.5,
    "humidity":82,
    "co2":980,
    "light":310,
    "measuredAt":"2026-08-15T12:30:00"
}
```

구독: Sensor Service

### EnvironmentControlEvent (제어 발생 시만 발행)

```json
{
    "cultivationId":3,
    "temperature":20.5,
    "humidity":82,
    "co2":980,
    "light":310,
    "action":"HUMIDIFIER_ON",
    "reason":"Humidity below humidity_min"
}
```

구독: Notification Service

범위 안에서 정상 상태로 유지되는 경우 EnvironmentControlEvent는 발행하지 않습니다.

---

## 8. Sensor Service 저장

RabbitMQ Subscribe (EnvironmentMeasuredEvent)

↓

Redis 저장 (최신값)

↓

InfluxDB 저장 (이력)

---

## 9. Notification Service

RabbitMQ Subscribe (EnvironmentControlEvent)

↓

사용자에게 알림 전송

예시

```
🍄 자동 제어

습도가 목표 범위(85~95%) 아래로 떨어져

가습기를 실행했습니다.
```

---

# 사용 Database

## Redis

현재 환경 저장 (Sensor Service)

---

## InfluxDB

환경 이력 저장 (Sensor Service)

---

# MQTT

Subscribe

```
sensor/{cultivationId}
```

---

# RabbitMQ

Publish

```
EnvironmentMeasuredEvent
EnvironmentControlEvent
```

Subscribe

- Sensor Service (EnvironmentMeasuredEvent)
- Notification Service (EnvironmentControlEvent)

---

# OpenFeign

```
Rule Engine Service

↓

Cultivation Service (목표 환경 범위 조회)
```

---

# 자동 제어 예시

| 조건 | 제어 |
|------|------|
| Temperature > temp_max | 냉각팬 ON |
| Temperature < temp_min | 히터 ON |
| Humidity < humidity_min | 가습기 ON |
| Humidity > humidity_max | 제습기 ON |
| CO₂ > co2_max | 환풍기 ON |
| Light < light_min | LED ON |
| 모든 항목이 범위 안 | 제어 없음 |

---

# 예외 상황

- MQTT 연결 실패
- 규칙 평가 오류
- 목표 환경 범위 조회 실패 (Cultivation Service 호출 실패)
- RabbitMQ 발행 실패
- Sensor Service 저장 실패
- Notification 전송 실패

---

# 고려 사항

- 목표 환경은 단일값이 아닌 범위(min~max)로 저장되어 있어, 범위 안에서는 장치를 켜고 끄지 않습니다.
- Rule Engine Service는 저장을 직접 하지 않고 RabbitMQ로 Sensor Service에 위임합니다.
- Rule Engine Service와 Sensor Service는 RabbitMQ로만 연결되며 서로 직접 호출하지 않습니다.
- Notification Service는 EnvironmentControlEvent만 구독하며, 저장용 EnvironmentMeasuredEvent는 구독하지 않습니다.
