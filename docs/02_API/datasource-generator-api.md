# DatasourceGenerator API

## 개요

DatasourceGenerator에서 제공하는 REST API 명세입니다. (기존 명칭: Datasource API)

IoT 데이터 소스 및 센서 "장치"를 등록/관리합니다. 센서가 측정한 환경 값 자체의 조회는
Sensor API(`/api/v1/sensors/current` 등)를 참고하세요. 이 문서는 장치 메타데이터 관리용입니다.

Base URL

```
/api/v1/datasources
/api/v1/sensors  (⚠️ 경로 충돌 주의, 아래 참고)
```

인증 방식

```
Bearer JWT (관리자 권한)
```

---

# ⚠️ 알려진 이슈: /sensors 경로 충돌

DatasourceGenerator(장치 등록: `POST /sensors`, `GET /sensors`, `GET /sensors/{sensorId}`)와
Sensor Service(측정값 조회: `GET /sensors/current`, `GET /sensors/statistics` 등)가
동일한 `/api/v1/sensors` 경로 프리픽스를 사용하고 있습니다.

두 서비스는 처음부터 별도 서비스였으며(Rule Engine Service는 측정값을 저장/제공하지 않고 MQTT/RabbitMQ로만 동작),
이 경로 충돌은 계속 남아 있습니다.

API Gateway가 서비스 이름이 아닌 경로만으로 라우팅할 경우 충돌 가능성이 있으므로,
아래 중 하나로 확정이 필요합니다.

- DatasourceGenerator의 장치 관리 경로를 `/api/v1/devices`로 변경
- Gateway 라우팅 규칙에 서비스 접두사 추가 (예: `/api/v1/datasource-generator/sensors`)

이 문서에서는 현행 도메인 문서(datasource-generator.md) 기준인 `/sensors`를 그대로 표기합니다.

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

# 센서 등록

## POST /sensors

### Request

```json
{
    "datasourceId": 1,
    "cultivationId": 3,
    "name": "온도 센서 1",
    "sensorType": "TEMPERATURE"
}
```

---

### Response

```json
{
    "sensorId": 4,
    "sensorUuid": "b3f1c2a0-...",
    "message": "센서가 등록되었습니다."
}
```

---

# 센서 조회

## GET /sensors

### Query Parameter

```
datasourceId (optional)
cultivationId (optional)
```

---

### Response

```json
[
    { "sensorId": 4, "name": "온도 센서 1", "sensorType": "TEMPERATURE", "status": "ONLINE" }
]
```

---

# 센서 상태 조회

## GET /sensors/{sensorId}

### Response

```json
{
    "sensorId": 4,
    "name": "온도 센서 1",
    "sensorType": "TEMPERATURE",
    "status": "ONLINE",
    "installedAt": "2026-07-01T09:00:00"
}
```

---

# Error Code

| Code | Description |
|------|-------------|
| D001 | 존재하지 않는 센서 |
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
SensorErrorEvent (Rule Engine Service 발행) - sensor.status 갱신
```

발행하는 이벤트는 없습니다. 센서 데이터는 MQTT를 통해서만 전송합니다.
