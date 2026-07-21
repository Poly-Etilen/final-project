# DatasourceGenerator API

## 개요

DatasourceGenerator는 REST API를 제공하지 않습니다. (기존 명칭: Datasource API)

MQTT Publish와 RabbitMQ Subscribe만으로 동작하는 이벤트/메시지 기반 서비스입니다.

센서 "장치"의 등록/조회/삭제와 센서가 측정한 환경 값 자체의 조회는 모두
[sensor-api.md](./sensor-api.md)를 참고하세요.

> ℹ️ **변경 이력**: 이전에는 이 서비스가 `/api/v1/sensors`로 센서 장치를 등록/조회하고,
> `/api/v1/datasources`로 데이터 소스(위치 그룹)를 관리했습니다. 센서 CRUD가 Cultivation
> Service를 거쳐 이후 Sensor Service(`/api/v1/sensors/cultivations/{id}`)로 옮겨지면서
> 경로 충돌 이슈가 해소되었고, 이어서 위치 정보(place/location)가 센서 레코드에 직접
> 저장되는 방식으로 바뀌면서 "데이터 소스" 개념 자체와 `/api/v1/datasources` API도
> 제거되었습니다. 그 결과 이 서비스에는 더 이상 REST API가 남아있지 않습니다.

> ℹ️ **변경 이력**: `sensor_cache`를 PostgreSQL이 아닌 메모리(In-Memory)에서만 관리하도록
> 바뀌면서, 서비스 시작 시 Sensor Service의 `GET /api/v1/sensors`를 OpenFeign으로
> 호출해 전체 센서 목록을 받아와 캐시를 재구성합니다. (자세한 내용은
> [sensor-api.md](./sensor-api.md)의 "전체 센서 목록 조회 (내부용)" 참고)

---

# Error Code

REST API가 없어 HTTP Error Code 체계를 사용하지 않습니다. 내부 처리 오류는 로그/모니터링으로 관리합니다.

| 상황 | 설명 |
|------|------|
| sensor_cache에 없는 센서 | Sensor Service 이벤트가 아직 반영되지 않았거나 유실됨 |
| MQTT Broker 연결 실패 | 센서 데이터 발행 불가 |
| 센서 연결 실패 | 실제 IoT 센서 연동 시 |
| 센서 데이터 형식 오류 | 시뮬레이션/실제 데이터 생성 오류 |
| SensorRegisteredEvent/SensorDeletedEvent 구독 실패 | sensor_cache가 최신 상태를 반영하지 못함 |
| 서비스 시작 시 Sensor Service 호출 실패 | 메모리 캐시를 재구성하지 못해 재시도할 때까지 데이터 발행 불가 |

---

# OpenFeign

호출받는 서비스

```
없음 (REST API가 없어 다른 서비스가 직접 호출하지 않습니다)
```

호출하는 서비스

```
Sensor Service - GET /api/v1/sensors (서비스 시작 시에만, 메모리 캐시 재구성용)
```

평상시 센서 등록/삭제는 이벤트(RabbitMQ)로만 전달되며, 이 호출은 재시작 시에만 발생합니다.

---

# MQTT

Publish Topic

```
sensor/{deviceEui}
```

Payload

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

# Event

## 구독 이벤트

```
SensorRegisteredEvent (Sensor Service 발행) - sensor_cache Upsert
SensorDeletedEvent (Sensor Service 발행) - sensor_cache에서 삭제
```

발행하는 이벤트는 없습니다. 센서 데이터는 MQTT를 통해서만 전송합니다.
