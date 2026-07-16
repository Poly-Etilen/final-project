# DatasourceGenerator

## 역할

DatasourceGenerator는 데이터 소스를 관리하고, 센서 데이터를 MQTT Broker로 발행(Publish)하는 서비스입니다.

(기존 명칭: Datasource Service)

실제 운영 환경에서는 IoT 센서와 연동되며, 개발 환경에서는 CSV 등을 반복적으로 읽어 센서 데이터를 시뮬레이션 생성합니다.
서비스 이름의 "Generator"는 이 시뮬레이션 데이터 생성 역할을 강조한 것입니다.

> ℹ️ **변경 이력**: 센서 "장치"의 등록/조회/삭제(CRUD)는 원래 이 서비스가 담당했지만, 센서가
> 항상 특정 재배(cultivation)에 종속되는 정보라는 점 때문에 Cultivation Service로 이전했습니다.
> DatasourceGenerator는 이제 "데이터를 생성/발행하는 것"에만 집중하며, 어떤 센서에 대해
> 데이터를 만들지는 Cultivation Service가 발행하는 이벤트를 구독해서 판단합니다.

---

# 책임

- 데이터 소스 관리
- 센서 데이터 발행 (실제 또는 시뮬레이션)
- MQTT Publish
- 센서 시뮬레이션 대상 목록 캐시 관리 (sensor_cache, 이벤트 기반)

---

# 주요 기능

## 데이터 소스 등록

센서가 연결될 데이터 소스를 등록합니다.

등록 정보

- 데이터 소스 이름
- 데이터 소스 타입
- 위치
- 설명

---

## 센서 데이터 발행

`sensor_cache`에 있는 센서(Cultivation Service가 등록한 센서)에 대해서만 데이터를 생성하여
MQTT Broker로 발행합니다.

발행 데이터

- Temperature
- Humidity
- CO₂
- Light

---

## 센서 시뮬레이션 대상 목록 캐시 관리

센서 장치 자체는 더 이상 관리하지 않지만, 어떤 sensorId에 대해 데이터를 생성해야 하는지는
알아야 합니다. Cultivation Service가 센서를 등록/삭제할 때 발행하는 `SensorRegisteredEvent`/
`SensorDeletedEvent`를 구독해 `sensor_cache` 테이블을 갱신합니다.

- 등록 시: sensor_cache에 Upsert
- 삭제 시: sensor_cache에서 삭제

센서 상태(ONLINE/OFFLINE/ERROR)는 이제 Cultivation Service의 책임입니다. Rule Engine
Service가 발행하는 SensorErrorEvent도 더 이상 이 서비스가 구독하지 않습니다.

---

# API

## 데이터 소스 등록

POST /datasources

---

## 데이터 소스 조회

GET /datasources

---

# Database

DatasourceGenerator는 PostgreSQL을 사용합니다.

### Table

- datasource
- sensor_cache (Cultivation Service의 sensor를 이벤트로 반영한 읽기 전용 캐시)

---

# Redis

사용하지 않습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

없음

---

## 호출받는 서비스

- API Gateway (관리 기능)

---

# Event

## 구독 이벤트

### SensorRegisteredEvent

Cultivation Service가 센서를 등록할 때 발행합니다. `sensor_cache`에 Upsert합니다.

---

### SensorDeletedEvent

Cultivation Service가 센서를 삭제할 때 발행합니다. `sensor_cache`에서 해당 행을 삭제합니다.

---

발행하는 이벤트는 없습니다. 센서 데이터는 MQTT를 통해서만 전송합니다.

> ℹ️ **변경 이력**: SensorErrorEvent(Rule Engine Service 발행)는 더 이상 이 서비스가 구독하지
> 않습니다. sensor.status의 원본이 Cultivation Service로 옮겨지면서 구독 주체도 함께
> 이전했습니다.

---

# MQTT

## Publish Topic

```
sensor/{sensorId}
```

### Payload

```json
{
  "sensorId": 1,
  "temperature": 21.8,
  "humidity": 92.4,
  "co2": 760,
  "light": 430,
  "measuredAt": "2026-08-15T12:30:00"
}
```

---

# Sequence

## 센서 데이터 생성/발행

센서 (실제 또는 시뮬레이션, sensor_cache에 있는 센서만)

↓

DatasourceGenerator

↓

MQTT Publish

↓

MQTT Broker

↓

Rule Engine Service

---

## sensor_cache 동기화

Cultivation Service

↓

센서 등록/삭제

↓

RabbitMQ Publish (SensorRegisteredEvent / SensorDeletedEvent)

↓

DatasourceGenerator

↓

sensor_cache Upsert/삭제

---

# 예외 상황

- sensor_cache에 없는 센서 (등록 이벤트가 아직 반영되지 않았거나 유실된 경우)
- 존재하지 않는 데이터 소스
- MQTT Broker 연결 실패
- 센서 연결 실패
- 센서 데이터 형식 오류
- SensorRegisteredEvent/SensorDeletedEvent 구독 실패 (sensor_cache가 최신 상태를 반영하지 못함)

---

# 추후 개발 예정

- 실제 IoT 센서 연동
- 센서 Health Check
- 센서 Firmware 관리
- sensor_cache 재동기화(reconciliation) 배치
