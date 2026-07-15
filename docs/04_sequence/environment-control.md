# 환경 자동 제어 시퀀스

## 개요

센서에서 수집한 환경 데이터를 기반으로 Rule Engine Service가 목표 환경과 비교하여
자동 제어 여부를 판단합니다.

Rule Engine Service는 MQTT 수신, 규칙 평가, Redis/InfluxDB 저장까지 하나의 서비스에서 처리합니다.
(기존에는 Rule Engine이 RabbitMQ를 거쳐 별도의 Sensor Service에 저장을 위임했으나,
서비스 통합으로 내부 처리로 단순화되었습니다. RabbitMQ는 이제 Notification Service 등
진짜 외부 서비스에 알릴 때만 사용합니다.)

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

├── 환경 비교
├── 제어 여부 판단
├── Redis 저장 (내부)
└── InfluxDB 저장 (내부)

↓

RabbitMQ

↓

Notification Service

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

환경 데이터를 수신합니다.

---

## 4. 목표 환경 조회

Rule Engine Service는

현재 재배의 목표 환경을 조회합니다.

예시

```
Temperature

22℃
```

```
Humidity

90%
```

---

## 5. 환경 비교

현재값

↓

목표값

↓

비교 수행

예시

```
현재 습도

82%
```

↓

```
목표

90%
```

↓

차이

```
8%
```

---

## 6. 자동 제어 판단

Rule Engine Service

↓

규칙 확인

예시

```
Humidity < Target

↓

Humidifier ON
```

또는

```
Temperature > Target

↓

Cooling Fan ON
```

---

## 7. Redis / InfluxDB 저장 (내부 처리)

같은 서비스 내부에서 즉시 저장합니다.

Redis

```
현재 환경
```

InfluxDB

```
환경 이력
```

---

## 8. RabbitMQ Publish

Rule Engine Service는

EnvironmentControlEvent를 발행합니다. (Notification Service 전달용)

```json
{
    "cultivationId":3,
    "temperature":20.5,
    "humidity":82,
    "co2":980,
    "light":310,
    "action":"HUMIDIFIER_ON"
}
```

---

## 9. Notification Service

이벤트를 구독합니다.

↓

사용자에게 알림 전송

예시

```
🍄 자동 제어

습도가 낮아

가습기를 실행했습니다.
```

---

# 사용 Database

## Redis

현재 환경 저장 (Rule Engine Service 내부)

---

## InfluxDB

환경 이력 저장 (Rule Engine Service 내부)

---

# MQTT

Subscribe

```
sensor/{cultivationId}
```

---

# RabbitMQ

Publish (다른 서비스로 전달할 때만 사용)

```
EnvironmentControlEvent
```

Subscribe

- Notification Service

---

# OpenFeign

사용하지 않습니다.

환경 제어는 MQTT 수신과 내부 처리, RabbitMQ 알림 전달로 구성됩니다.

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

# 예외 상황

- MQTT 연결 실패
- 규칙 평가 오류
- Redis 저장 실패
- InfluxDB 저장 실패
- RabbitMQ 발행 실패
- Notification 전송 실패

---

# 고려 사항

- Rule Engine Service는 현재 환경과 목표 환경 비교, 저장까지 하나의 서비스에서 처리하여 지연을 줄입니다.
- RabbitMQ는 Notification Service처럼 실제로 분리된 서비스에 알릴 때만 사용합니다.
- 최신 데이터는 Redis에, 모든 이력은 InfluxDB에 저장합니다.
- Notification Service는 이벤트만 수신하며 제어에는 관여하지 않습니다.
