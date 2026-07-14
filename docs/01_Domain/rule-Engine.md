# Rule Engine Service

## 역할

Rule Engine Service는 MQTT를 통해 수신한 센서 데이터를 분석하여
미리 정의된 규칙(Rule)에 따라 자동으로 재배 환경을 제어하는 서비스입니다.

환경이 목표 범위를 벗어나면 적절한 장치를 제어하고,
결과를 RabbitMQ를 통해 다른 서비스에 전달합니다.

---

# 책임

- 센서 데이터 수신
- 환경 상태 분석
- 규칙(Rule) 평가
- 장치 자동 제어
- 이벤트 발행
- 자동 제어 이력 관리

---

# 주요 기능

## 센서 데이터 분석

MQTT Broker로부터 센서 데이터를 수신합니다.

분석 항목

- Temperature
- Humidity
- CO₂
- Light

---

## 규칙 평가

현재 센서 값과 목표 환경을 비교하여 규칙을 평가합니다.

예시

- 목표 습도 : 90%
- 현재 습도 : 82%

↓

가습기 ON

---

## 장치 자동 제어

환경에 맞게 장치를 자동으로 제어합니다.

제어 대상

- 가습기
- 환풍기
- 냉각기
- 히터
- LED 조명

---

## 이벤트 발행

자동 제어 결과를 RabbitMQ로 전송합니다.

전송 대상

- Sensor Service
- Notification Service

---

# API

Rule Engine Service는 외부에서 직접 호출하지 않습니다.

MQTT와 RabbitMQ를 통해 동작합니다.

---

# Database

별도의 Database를 사용하지 않습니다.

---

# Redis

사용하지 않습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

없음

---

## 호출받는 서비스

MQTT Broker

---

## 이벤트 발행 대상

RabbitMQ

↓

Sensor Service

↓

Notification Service

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

## Publish Event

### EnvironmentControlEvent

환경 자동 제어 결과

예시

```json
{
  "cultivationId": 3,
  "device": "HUMIDIFIER",
  "action": "ON",
  "reason": "Humidity below target",
  "timestamp": "2026-08-15T12:31:00"
}
```

---

# 자동 제어 예시

## 습도

```
현재 습도 < 목표 습도

↓

가습기 ON
```

---

## 온도

```
현재 온도 > 목표 온도

↓

냉각기 ON
```

---

## CO₂

```
현재 CO₂ > 목표 CO₂

↓

환풍기 ON
```

---

## 조도

```
현재 조도 < 목표 조도

↓

LED ON
```

---

# Sequence

Datasource Service

↓

MQTT Publish

↓

MQTT Broker

↓

Rule Engine

↓

Rule 평가

↓

장치 제어

↓

RabbitMQ

↓

Sensor Service

Notification Service

---

# 예외 상황

- MQTT Broker 연결 실패
- 규칙 평가 실패
- 장치 제어 실패
- RabbitMQ 전송 실패
- 잘못된 센서 데이터

---

# 추후 개발 예정

- 사용자 정의 Rule 지원
- Rule 우선순위 설정
- Rule 활성화/비활성화
- Rule 시뮬레이션 기능
- AI 기반 Rule 자동 생성