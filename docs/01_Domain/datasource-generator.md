# DatasourceGenerator

## 역할

DatasourceGenerator는 센서 데이터를 MQTT Broker로 발행(Publish)하는 서비스입니다. 실제
장치가 없는 개발/테스트 환경에서 센서 값을 시뮬레이션해 발행합니다. REST API를 제공하지
않으며, 별도의 영구 저장소도 갖지 않습니다.

---

# 책임

- 등록된 센서 목록 기준으로 측정값 시뮬레이션 및 MQTT 발행

---

# 주요 기능

## 센서 데이터 발행

메모리에 캐시된 센서 목록(`sensor_cache`: device_eui/cultivationId/sensorTypes)을 기준으로
1초 주기로 값을 생성해 MQTT로 발행합니다.

---

## 센서 캐시 관리

Sensor Service가 발행하는 `SensorRegisteredEvent`/`SensorDeletedEvent`를 구독해
`sensor_cache`를 갱신합니다. 서비스 재시작 시에는 Sensor Service의 내부용 전체 목록 조회
API를 OpenFeign으로 호출해 캐시를 일괄 재구성합니다.

---

# API

REST API를 제공하지 않습니다 (MQTT Publish + RabbitMQ Subscribe만 수행).

---

# Database

별도의 PostgreSQL DB를 사용하지 않습니다. `sensor_cache`는 메모리(In-Memory)에서만
관리합니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Sensor Service

- 서비스 시작 시 전체 센서 목록 조회 (`GET /api/v1/sensors`, 내부용)

---

# Event

## Subscribe

### SensorRegisteredEvent / SensorDeletedEvent

Sensor Service가 발행. `sensor_cache`를 갱신합니다.

---

# Sequence

관련 시퀀스는 [sensor-data.md](../04_sequence/sensor-data.md) 참고.

---

# 예외 상황

- MQTT Broker 연결 실패
- 서비스 재시작 시 Sensor Service 전체 조회 실패 (캐시가 비어 발행이 지연될 수 있음)

---

# 추후 개발 예정

- 이벤트 유실 대비 주기적 재동기화(reconciliation) 배치
