# DatasourceGenerator

## 역할

DatasourceGenerator는 센서 데이터를 MQTT Broker로 발행(Publish)하는 서비스입니다.

(기존 명칭: Datasource Service)

실제 운영 환경에서는 IoT 센서와 연동되며, 개발 환경에서는 CSV 등을 반복적으로 읽어 센서 데이터를 시뮬레이션 생성합니다.
서비스 이름의 "Generator"는 이 시뮬레이션 데이터 생성 역할을 강조한 것입니다.

> ℹ️ **변경 이력**: 센서 "장치"의 등록/조회/삭제(CRUD)는 원래 이 서비스가 담당했지만, 센서가
> 항상 특정 재배(cultivation)에 종속되는 정보라는 점 때문에 Cultivation Service로 이전했습니다.
> DatasourceGenerator는 이제 "데이터를 생성/발행하는 것"에만 집중하며, 어떤 센서에 대해
> 데이터를 만들지는 Cultivation Service가 발행하는 이벤트를 구독해서 판단합니다.

> ℹ️ **변경 이력**: 센서 등록 시 사용자가 입력하는 위치 정보(place/location)가 별도 엔티티
> 없이 센서 레코드에 직접 저장되는 방식으로 바뀌면서, 이 서비스가 관리하던 "데이터 소스"
> 개념 자체가 폐지되었습니다. 데이터 소스 등록/조회 API도 함께 제거되어, 이제 이 서비스는
> REST API를 전혀 제공하지 않습니다.

> ℹ️ **변경 이력**: `sensor_cache`는 device_eui/cultivationId/sensorType만 갖는 단순 조회용
> 캐시로, 조인/트랜잭션이 필요 없어 관계형 DB(PostgreSQL)를 둘 이유가 없다고 판단했습니다.
> 이제 별도 DB 없이 메모리(In-Memory, 예: ConcurrentHashMap)에서만 관리합니다. 다만 재시작하면
> 메모리가 비므로, 시작 시점에 Cultivation Service를 OpenFeign으로 호출해(`GET
> /api/v1/sensors`) 전체 센서 목록을 한 번에 받아와 캐시를 재구성합니다.

---

# 책임

- 센서 데이터 발행 (실제 또는 시뮬레이션)
- MQTT Publish
- 센서 시뮬레이션 대상 목록 캐시 관리 (sensor_cache, 메모리 + 이벤트 기반 갱신)

---

# 주요 기능

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

센서 장치 자체는 더 이상 관리하지 않지만, 어떤 device_eui에 대해 데이터를 생성해야 하는지는
알아야 합니다. `sensor_cache`는 별도 DB 없이 메모리(In-Memory)에서 관리하며, 두 가지 방식으로
채워집니다.

- **평상시(이벤트 기반)**: Cultivation Service가 센서를 등록/삭제할 때 발행하는
  `SensorRegisteredEvent`/`SensorDeletedEvent`를 구독해 메모리 캐시를 갱신합니다.
  - 등록 시: sensor_cache에 Upsert
  - 삭제 시: sensor_cache에서 삭제
- **서비스 시작 시(재구성)**: 메모리 캐시는 재시작하면 비어 있으므로, Cultivation Service의
  `GET /api/v1/sensors`를 OpenFeign으로 호출해 전체 센서 목록을 한 번에 받아와 캐시를 채웁니다.
  이후에는 다시 이벤트 기반으로 갱신됩니다.

센서 상태(ONLINE/OFFLINE/ERROR)는 이제 Cultivation Service의 책임입니다. Rule Engine
Service가 발행하는 SensorErrorEvent도 더 이상 이 서비스가 구독하지 않습니다.

---

# API

DatasourceGenerator는 REST API를 제공하지 않습니다.

MQTT Publish와 RabbitMQ Subscribe만으로 동작하는 이벤트/메시지 기반 서비스입니다.
센서 "장치" 자체의 등록/조회/삭제는 [cultivation-api.md](../02_API/cultivation-api.md)를 참고하세요.

---

# Database

DatasourceGenerator는 별도의 Database를 사용하지 않습니다.

`sensor_cache`는 device_eui/cultivationId/sensorType만 갖는 단순 조회용 캐시라 PostgreSQL
같은 관계형 DB 없이 메모리(In-Memory)에서만 관리합니다. (자세한 내용은
[datasource-generator-db.md](../03_Database/datasource-generator-db.md) 참고)

---

# Redis

사용하지 않습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Cultivation Service

서비스 시작 시에만 `GET /api/v1/sensors`를 OpenFeign으로 호출해 전체 센서 목록을 조회하고,
메모리 캐시(sensor_cache)를 재구성합니다. 평상시에는 호출하지 않습니다(이벤트로만 갱신).

---

## 호출받는 서비스

없음 (REST API가 없어 다른 서비스가 직접 호출하지 않습니다)

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
sensor/{deviceEui}
```

### Payload

```json
{
  "deviceEui": "24e124128c067999",
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

메모리 캐시(sensor_cache) Upsert/삭제

---

## 서비스 시작 시 캐시 재구성

DatasourceGenerator 시작

↓

Cultivation Service에 OpenFeign 호출 (`GET /api/v1/sensors`)

↓

전체 센서 목록 수신

↓

메모리 캐시(sensor_cache) 일괄 채움

↓

이후 이벤트(SensorRegisteredEvent/SensorDeletedEvent) 기반으로 갱신

---

# 예외 상황

- sensor_cache에 없는 센서 (등록 이벤트가 아직 반영되지 않았거나 유실된 경우)
- MQTT Broker 연결 실패
- 센서 연결 실패
- 센서 데이터 형식 오류
- SensorRegisteredEvent/SensorDeletedEvent 구독 실패 (sensor_cache가 최신 상태를 반영하지 못함)
- 서비스 시작 시 Cultivation Service 호출 실패 (캐시를 재구성하지 못해 재시도할 때까지 데이터 발행 불가)

---

# 추후 개발 예정

- 실제 IoT 센서 연동
- 센서 Health Check
- 센서 Firmware 관리
- 이벤트 유실에 대비한 주기적 재동기화(reconciliation) — 현재는 서비스 시작 시에만 전체 재조회
