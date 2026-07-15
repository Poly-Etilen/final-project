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

mushroom_reference 조회 (mushroom_type 기준)

↓

환경 추천 반환

↓

Client

AI Service를 호출하지 않고, Cultivation Service가 자체 보유한 참조 테이블을 조회합니다.

---

### Response

```json
{
    "cultivationId": 1,
    "recommendedEnvironment": {
        "temperature": {"min": 15.0, "max": 18.0},
        "humidity": {"min": 85, "max": 95},
        "co2": {"min": 700, "max": 900},
        "light": {"min": 300, "max": 400}
    },
    "description": "느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다."
}
```

추천값은 mushroom_reference에 저장된 범위를 그대로 반환합니다. 사용자가 이 범위를 참고해서
아래 "환경 설정 저장"에서 실제 자동 제어 기준값을 직접 입력합니다.

---

# 환경 설정 저장

## PATCH /{cultivationId}/environment

사용자가 추천값(mushroom_reference)을 참고하여 실제 자동 제어 기준값(위험 한계값)을 저장합니다.

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

저장 시 단일 목표값을 허용 오차만큼 확장한 범위(min~max)로 변환해 저장하고,
RabbitMQ로 EnvironmentRangeUpdatedEvent를 발행합니다. (Rule Engine Service의 Redis 캐시 갱신용)

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

`environment`의 각 값은 environment_setting에 저장된 범위의 중간값 `(min+max)/2`을 조회
시점에 계산한 것입니다. 허용 오차가 대칭으로 적용되므로 저장 시 사용자가 입력했던 단일값과
정확히 일치합니다. (자세한 내용은 [cultivation-db.md](../03_Database/cultivation-db.md)의
"범위 → 단일값 역변환" 참고)

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
| C005 | 지원하지 않는 버섯 종류 (mushroom_reference에 없음) |

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
- EnvironmentRangeUpdatedEvent