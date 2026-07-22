# Rule Engine Service

## 역할

Rule Engine Service는 MQTT로 수신한 센서 데이터를 검증하고, 목표 환경 범위를 기준으로
자동 제어를 판단하는 서비스입니다. 별도의 영구 저장소 없이 Redis 캐시만 사용하며, 측정값
자체의 저장/조회는 Cultivation Service가 담당합니다.

---

# 책임

- MQTT 수신 (Collector 역할)
- 측정값 검증
- 목표 환경 범위 기반 자동 제어 판단
- 센서 오류(연결 끊김 등) 감지

---

# 주요 기능

## 측정값 수신 및 검증

DatasourceGenerator가 MQTT로 발행한 값을 1초 주기로 수신해 유효 범위를 검증합니다.
검증 후 RabbitMQ로 `EnvironmentMeasuredEvent`를 발행해 Cultivation Service에 전달합니다.

---

## 자동 제어 판단

Redis에 캐싱된 목표 환경 범위(min~max)와 현재 측정값을 비교해 장치를 제어합니다. 제어를
**시작**하는 기준은 범위 경계(min/max)이고, 제어를 **멈추는** 기준은 범위의 중앙값
(mid = (min+max)/2)입니다 — 경계 부근에서 값이 미세하게 오르내려도 반복 On/Off가
발생하지 않도록 하기 위함입니다. 캐시가 없을 때만 Cultivation Service를 OpenFeign으로 호출해
값을 채웁니다.

---

## 센서 오류 감지

일정 시간 동안 값이 들어오지 않으면 `SensorErrorEvent`를 발행해 Cultivation Service(센서
상태 갱신)와 Notification Service(알림)에 전달합니다.

---

# API

MQTT/RabbitMQ 기반으로만 동작하며 REST API는 제공하지 않습니다.

---

# Database

영구 저장소가 없습니다.

---

# Redis

- 목표 환경 범위 캐시 (`cultivation:{cultivationId}:range`, TTL 24시간, write-through)

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Cultivation Service

- 목표 환경 범위 캐시 미스 시 fallback 조회

---

## 호출받는 서비스

### DatasourceGenerator

- MQTT로 측정값 발행

---

# Event

## Publish

### EnvironmentMeasuredEvent

Cultivation Service가 구독해 저장합니다.

### SensorErrorEvent

Cultivation Service(상태 갱신), Notification Service(알림)가 구독합니다.

---

## Subscribe

### EnvironmentRangeUpdatedEvent

Cultivation Service가 발행. 목표 환경 범위 캐시를 write-through로 갱신합니다.

---

# Sequence

관련 시퀀스는 [sensor-data.md](../04_sequence/sensor-data.md),
[environment-control.md](../04_sequence/environment-control.md),
[sensor-error.md](../04_sequence/sensor-error.md) 참고.

---

# 예외 상황

- MQTT 연결 끊김
- Redis 캐시 미스 + Cultivation Service 호출 실패
- 유효 범위를 벗어난 이상값 수신

---

# 추후 개발 예정

- 재배별/장치별 현재 ON/OFF 상태 추적 저장 방식 확정
