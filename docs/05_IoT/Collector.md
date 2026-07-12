# Collector Service

## 목적

MQTT Broker에서 센서 데이터를 수신한다.

---

## 역할

- MQTT Subscribe
- JSON 파싱
- DTO 변환
- Rule Engine 전달

---

## 처리 과정

```text
MQTT

↓

Collector

↓

SensorDataDTO

↓

Rule Engine
```

---

## 특징

Collector는 데이터를 저장하지 않는다.

Collector는 비즈니스 로직을 수행하지 않는다.