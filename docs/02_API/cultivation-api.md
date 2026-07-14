# Cultivation API

## 개요

Cultivation Service에서 제공하는 REST API 명세입니다.

Base URL

```
/api/v1/cultivations
```

인증 방식

```
Bearer JWT
```

---

# 재배 생성

## POST /

새로운 재배를 생성합니다.

### Request

```json
{
    "name": "느타리버섯 1호기",
    "mushroomType": "OYSTER"
}
```

---

### Process

Client

↓

Cultivation Service

↓

AI Service

↓

Embedding Service

↓

LLM

↓

환경 추천 반환

↓

Client

---

### Response

```json
{
    "cultivationId": 1,
    "recommendedEnvironment": {
        "temperature": 21,
        "humidity": 90,
        "co2": 800,
        "light": 350
    }
}
```

---

# 환경 설정 저장

## PATCH /{cultivationId}/environment

사용자가 AI 추천값을 수정하여 저장합니다.

### Request

```json
{
    "temperature":22,
    "humidity":91,
    "co2":850,
    "light":420
}
```

---

### Response

```json
{
    "message":"Environment Saved"
}
```

---

# 재배 목록 조회

## GET /

### Response

```json
[
    {
        "cultivationId":1,
        "name":"느타리 1호기",
        "status":"RUNNING"
    }
]
```

---

# 재배 상세 조회

## GET /{cultivationId}

### Response

```json
{
    "cultivationId":1,
    "name":"느타리 1호기",

    "environment":{

        "temperature":22,
        "humidity":91,
        "co2":850,
        "light":420

    },

    "status":"RUNNING",

    "createdAt":"2026-08-01"
}
```

---

# 재배 수정

## PATCH /{cultivationId}

### Request

```json
{
    "name":"새로운 이름"
}
```

---

# 재배 삭제

## DELETE /{cultivationId}

---

# 재배 종료

## PATCH /{cultivationId}/finish

### Request

```json
{
    "harvestWeight":3200,
    "memo":"생육 상태 양호"
}
```

---

### Response

```json
{
    "message":"Cultivation Finished"
}
```

---

# 재배 이력 조회

## GET /history

### Response

```json
[
    {
        "cultivationId":3,
        "harvestWeight":3200,
        "duration":27
    }
]
```

---

# Error Code

| Code | Description |
|------|-------------|
| C001 | 존재하지 않는 재배 |
| C002 | 이미 종료된 재배 |
| C003 | 권한 없음 |
| C004 | 환경 저장 실패 |
| C005 | AI 추천 실패 |

---

# OpenFeign

사용 서비스

- AI Service
- Sensor Service

---

# Event

발행 이벤트

- CultivationCreatedEvent
- CultivationFinishedEvent