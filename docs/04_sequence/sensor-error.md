# 센서 오류 알림 시퀀스

## 개요

센서가 정상적으로 데이터를 전송하지 않거나 비정상적인 값을 전송할 경우
이를 감지하여 사용자에게 알리는 과정입니다.

Rule Engine Service가 수신 주기와 데이터 유효성을 검사하며,
이상이 감지되면 Cultivation Service의 센서 상태를 갱신하고 Notification Service를 통해 알립니다.

> ℹ️ **변경 이력**: 센서 장치 CRUD가 DatasourceGenerator에서 Cultivation Service로 이전되면서,
> SensorErrorEvent 구독(센서 상태 갱신) 주체도 함께 옮겨졌습니다.

---

# Sequence

```text
Sensor (미수신 / 이상값)

↓

MQTT Broker

↓

Rule Engine Service

↓

수신 주기 검사

↓

데이터 유효성 검사

↓

이상 판단

↓

RabbitMQ

↓

SensorErrorEvent 발행

├── Cultivation Service
│     └── sensor.status 갱신
│
└── Notification Service

↓

WebSocket
Telegram
Discord
```

---

# 상세 과정

## 1. 정상 수신 주기

센서는 일정 주기로 MQTT Broker에 데이터를 Publish합니다.

예시

```
1초 주기
```

---

## 2. 수신 주기 검사

Rule Engine Service는 Scheduler를 통해

센서별 마지막 수신 시각을 주기적으로 확인합니다.

예시

```
마지막 수신: 2026-08-15 12:30:00

현재 시각: 2026-08-15 12:31:10

경과: 70초
```

기준 시간(60초)을 초과하면 OFFLINE으로 판단합니다.

---

## 3. 데이터 유효성 검사

수신된 값이 정상 범위를 벗어나는 경우 ERROR로 판단합니다.

예시

```json
{
    "sensorId":1,
    "temperature":-999,
    "humidity":91
}
```

허용 범위를 벗어난 값은 이상 데이터로 처리합니다.

---

## 4. 이상 판단

Rule Engine Service

↓

센서 상태 결정

```
OFFLINE
```

또는

```
ERROR
```

---

## 5. RabbitMQ Publish

Rule Engine Service는

SensorErrorEvent를 발행합니다.

```json
{
    "cultivationId":3,
    "sensorId":1,
    "status":"OFFLINE",
    "reason":"60초 이상 데이터 미수신",
    "detectedAt":"2026-08-15T12:31:10"
}
```

---

## 6. Cultivation Service 처리

RabbitMQ Subscribe

↓

sensor 테이블의 status 컬럼을 갱신합니다.

```
ONLINE → OFFLINE
```

---

## 7. Notification Service 처리

동일 이벤트를 구독합니다.

↓

사용자에게 알림 전송

예시

```
🍄 센서 오류

느타리 1호기의 습도 센서가
60초 이상 응답하지 않습니다.
```

---

# 사용 Database

## PostgreSQL

```
sensor (Cultivation DB)
```

---

# MQTT

Subscribe

```
sensor/+
```

수신 주기 및 값 검증에 사용합니다.

---

# RabbitMQ

Publish

```
SensorErrorEvent
```

Subscribe

- Cultivation Service
- Notification Service

---

# OpenFeign

사용하지 않습니다.

센서 오류 감지 및 알림은 이벤트 기반으로 처리합니다.

---

# 예외 상황

- MQTT Broker 연결 자체 장애 (전체 센서 감지 불가)
- RabbitMQ 발행 실패
- Cultivation Service 상태 갱신 실패
- Notification 전송 실패
- 일시적 지연으로 인한 오탐(False Positive)

---

# 고려 사항

- Rule Engine Service는 센서별 마지막 수신 시각을 메모리 또는 캐시에 관리합니다.
- 수신 주기 기준(60초)은 센서 타입에 따라 다르게 설정할 수 있습니다.
- 센서 오류 상태는 Cultivation DB의 sensor.status에서만 관리하며 InfluxDB에는 기록하지 않습니다. (DatasourceGenerator는 이 상태를 알 필요가 없으며, sensor_cache에도 상태 컬럼을 두지 않습니다.)
- 센서가 다시 정상 데이터를 전송하면 상태를 ONLINE으로 복구합니다. (SensorRecoveredEvent는 추후 개발 예정)
- 오탐을 줄이기 위해 일정 횟수 이상 이상값이 반복될 때만 ERROR로 판단하는 방식은 추후 개선 대상입니다.
