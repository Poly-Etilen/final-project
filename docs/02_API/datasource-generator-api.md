# DatasourceGenerator API

## 개요

DatasourceGenerator에서 제공하는 REST API 명세입니다. (기존 명칭: Datasource API)

데이터 소스(장치가 설치되는 물리적 위치/그룹) 등록·조회만 제공합니다. 센서 "장치"의
등록/조회/삭제는 Cultivation Service가 담당합니다([cultivation-api.md](./cultivation-api.md)의
"센서 등록"/"센서 목록 조회"/"센서 삭제" 참고). 센서가 측정한 환경 값 자체의 조회는
Sensor API(`/api/v1/sensors/current` 등)를 참고하세요.

Base URL

```
/api/v1/datasources
```

인증 방식

```
Bearer JWT (관리자 권한)
```

> ℹ️ **변경 이력**: 이전에는 이 서비스가 `/api/v1/sensors`로 센서 장치도 등록/조회했는데,
> Sensor Service가 측정값 조회에 같은 경로 프리픽스(`/api/v1/sensors/current` 등)를 쓰고 있어
> 경로 충돌 이슈가 있었습니다. 센서 CRUD가 Cultivation Service(`/api/v1/cultivations/{id}/sensors`)로
> 옮겨지면서 이 충돌은 해소되었습니다.

---

# 데이터 소스 등록

## POST /datasources

### Request

```json
{
    "name": "재배실 1번",
    "type": "GREENHOUSE",
    "location": "1동 A구역",
    "description": "느타리버섯 재배실"
}
```

---

### Response

```json
{
    "datasourceId": 1,
    "message": "데이터 소스가 등록되었습니다."
}
```

---

# 데이터 소스 조회

## GET /datasources

### Response

```json
[
    { "datasourceId": 1, "name": "재배실 1번", "type": "GREENHOUSE", "status": "ACTIVE" }
]
```

---

# Error Code

| Code | Description |
|------|-------------|
| D001 | sensor_cache에 없는 센서 (Cultivation Service 이벤트 미반영/유실) |
| D002 | 존재하지 않는 데이터 소스 |
| D003 | MQTT Broker 연결 실패 |
| D004 | 센서 연결 실패 |
| D005 | 센서 데이터 형식 오류 |

---

# OpenFeign

호출받는 서비스

```
API Gateway (관리 기능)
```

호출하는 서비스

```
없음
```

---

# MQTT

Publish Topic

```
sensor/{sensorId}
```

Payload

```json
{
    "sensorId": 4,
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
SensorRegisteredEvent (Cultivation Service 발행) - sensor_cache Upsert
SensorDeletedEvent (Cultivation Service 발행) - sensor_cache에서 삭제
```

발행하는 이벤트는 없습니다. 센서 데이터는 MQTT를 통해서만 전송합니다.
