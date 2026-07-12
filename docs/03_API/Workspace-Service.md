# Workspace Service API

## Workspace 생성

POST /api/workspaces

### Request

```json
{
    "name":"우리집 몬스테라",
    "description":"거실",
    "plant":"몬스테라"
}
```

### Response

```json
{
    "workspaceId":1
}
```

---

## Workspace 목록 조회

GET /api/workspaces

---

## Workspace 상세조회

GET /api/workspaces/{workspaceId}

---

## Workspace 수정

PATCH /api/workspaces/{workspaceId}

---

## Workspace 삭제

DELETE /api/workspaces/{workspaceId}

---

# AI 추천

POST /api/workspaces/recommend

### Request

```json
{
    "prompt":"몬스테라 키우기 좋은 환경으로 구성해줘."
}
```

### Response

```json
{
    "temperature":24,
    "humidity":65,
    "co2":600,
    "ph":6.5,
    "reason":"..."
}
```

> 이 API는 AI 추천만 수행한다.
> EnvironmentSetting은 저장되지 않는다.

---

# 환경 저장

POST /api/workspaces/{workspaceId}/environment

### Request

```json
{
    "temperature":25,
    "humidity":70,
    "co2":700,
    "ph":6.4,

    "temperatureMin":23,
    "temperatureMax":27,

    "humidityMin":60,
    "humidityMax":80
}
```

> 사용자가 최종 저장 버튼을 눌렀을 때 호출된다.

---

# Device 등록

POST /api/workspaces/{workspaceId}/devices

```json
{
    "mqttClientId":"device001",

    "topic":"ecosphere/device001",

    "sensorType":"TEMPERATURE"
}
```

---

# Workspace 초대

POST /api/workspaces/{workspaceId}/invite

### Response

```json
{
    "inviteCode":"ABCD1234"
}
```

---

# 초대 수락

POST /api/workspaces/join

```json
{
    "inviteCode":"ABCD1234"
}
```

---

# AI Report 조회

GET /api/workspaces/{workspaceId}/reports