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

새로운 재배를 생성합니다. 이 재배에 연결할 센서 "장치"도 `devices`에 함께 담아 한 번에 등록할 수
있습니다. (물론 나중에 아래 "센서 등록" API로 추가/삭제하는 것도 계속 가능합니다.)

> ℹ️ **변경 이력**: 원래는 재배 생성(이름+버섯종류)과 센서 등록이 완전히 분리된 API였지만,
> 실제 사용자 흐름상 재배 생성 화면에서 등록할 디바이스도 함께 선택하는 것이 자연스러워
> `devices`를 재배 생성 요청에 포함할 수 있도록 확장했습니다. 기존 `POST
> /{cultivationId}/sensors`(센서 개별 등록)는 재배 생성 이후 추가로 센서를 붙일 때 그대로 사용합니다.

### Request

```json
{
    "name": "느타리버섯 1호기",
    "mushroomType": "OYSTER",
    "devices": [
        {
            "deviceEui": 4,
            "place": "1동 A구역",
            "location": "선반 2단",
            "deviceModel": "DHT22",
            "sensorType": "TEMPERATURE"
        }
    ]
}
```

`devices`는 선택 항목입니다. 생략하거나 빈 배열이면 센서 없이 재배만 생성되며, 이후 "센서 등록"
API로 추가할 수 있습니다.

---

### Process

Client

↓

Cultivation Service

↓

mushroom_reference 조회 (mushroom_type 기준)

↓

Cultivation 생성 + devices의 각 항목으로 sensor 레코드 생성 (하나의 트랜잭션)

↓

디바이스가 1개 이상 등록되었다면 RabbitMQ로 `SensorRegisteredEvent`를 디바이스별로 발행

↓

환경 추천 반환

↓

Client

AI Service를 호출하지 않고, Cultivation Service가 자체 보유한 참조 테이블을 조회합니다.
`devices`에 중복된 device_eui가 있거나 이미 등록된 device_eui가 섞여 있으면(C007) 재배 생성
자체가 롤백됩니다(전체 성공 또는 전체 실패).

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
    "description": "느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다.",
    "registeredSensors": [
        { "deviceEui": 4, "message": "센서가 등록되었습니다." }
    ]
}
```

추천값은 mushroom_reference에 저장된 범위를 그대로 반환합니다. 사용자가 이 범위를 참고해서
아래 "환경 설정 저장"에서 실제 자동 제어 기준값을 직접 입력합니다. `description`은
mushroom_reference의 짧은 참고 문구이며, 버섯 효능/재배 시 주의사항을 다루는 AI 생성 콘텐츠는
[ai-api.md](./ai-api.md)의 "버섯 가이드"를 참고하세요. `devices`를 생략했다면 `registeredSensors`는
빈 배열입니다.

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

# 센서 등록

## POST /{cultivationId}/sensors

재배 생성 이후 센서를 추가로 등록할 때 사용합니다. (재배 생성과 동시에 등록하려면 위 "재배 생성"의
`devices`를 사용하세요.)

### Request

```json
{
    "deviceEui": 4,
    "place": "1동 A구역",
    "location": "선반 2단",
    "deviceModel": "DHT22",
    "sensorType": "TEMPERATURE"
}
```

`deviceEui`는 장치 고유 식별자이며 그대로 PK로 사용됩니다(서버가 별도로 채번하지 않음).

---

### Response

```json
{
    "deviceEui": 4,
    "message": "센서가 등록되었습니다."
}
```

등록과 동시에 RabbitMQ로 `SensorRegisteredEvent`를 발행합니다. (DatasourceGenerator의
sensor_cache 갱신용)

---

# 센서 목록 조회

## GET /{cultivationId}/sensors

### Response

```json
[
    { "deviceEui": 4, "place": "1동 A구역", "location": "선반 2단", "deviceModel": "DHT22", "sensorType": "TEMPERATURE", "status": "ONLINE" }
]
```

---

# 센서 상세 조회

## GET /{cultivationId}/sensors/{deviceEui}

### Response

```json
{
    "deviceEui": 4,
    "place": "1동 A구역",
    "location": "선반 2단",
    "deviceModel": "DHT22",
    "sensorType": "TEMPERATURE",
    "status": "ONLINE"
}
```

---

# 센서 삭제

## DELETE /{cultivationId}/sensors/{deviceEui}

### Response

```json
{
    "message": "센서가 삭제되었습니다."
}
```

삭제와 동시에 RabbitMQ로 `SensorDeletedEvent`를 발행합니다.

---

# 센서 전체 목록 조회 (내부용)

## GET /api/v1/sensors

특정 재배로 한정하지 않고, 전체 재배의 센서 목록을 조회합니다. 사용자가 아닌 **DatasourceGenerator가
서비스 재시작 시 캐시(In-Memory)를 재구성하기 위해서만 호출**하는 내부용 엔드포인트입니다.
(다른 센서 API처럼 `/cultivations/{cultivationId}` 하위 경로가 아닌 것에 유의)

### Response

```json
[
    { "deviceEui": 4, "cultivationId": 3, "sensorType": "TEMPERATURE" }
]
```

place/location/deviceModel/status 등 상세 메타데이터는 포함하지 않습니다. DatasourceGenerator는
"어떤 device_eui에 대해 데이터를 생성/발행할지" 판단하는 데 필요한 최소 정보만 필요하기 때문입니다.

> ℹ️ **변경 이력**: DatasourceGenerator가 `sensor_cache`를 PostgreSQL이 아닌 메모리(In-Memory)에만
> 보관하도록 바뀌면서, 서비스 재시작 시 이벤트만으로는 캐시를 복구할 수 없는 문제가 생겼습니다.
> 이를 해결하기 위해 재시작 시점에 이 엔드포인트를 OpenFeign으로 호출해 전체 센서 목록을 한 번에
> 받아와 캐시를 재구성합니다.

---

# Error Code

| Code | Description |
|------|-------------|
| C001 | 존재하지 않는 재배 |
| C002 | 이미 종료된 재배 |
| C003 | 권한 없음 |
| C004 | 환경 저장 실패 |
| C005 | 지원하지 않는 버섯 종류 (mushroom_reference에 없음) |
| C006 | 존재하지 않는 센서 |
| C007 | 이미 등록된 device_eui (센서 등록 시 중복) |

---

# OpenFeign

사용 서비스

- AI Service
- Sensor Service
- DatasourceGenerator (서비스 시작 시에만, `GET /api/v1/sensors` 전체 목록 조회)

센서 등록/삭제 자체는 DatasourceGenerator를 동기 호출하지 않고 이벤트(RabbitMQ)로만 전달합니다.
DatasourceGenerator가 Cultivation Service를 OpenFeign으로 호출하는 것은 재시작 시 메모리 캐시를
재구성하는 경우뿐입니다.

---

# Event

발행 이벤트

- CultivationCreatedEvent
- CultivationFinishedEvent
- EnvironmentRangeUpdatedEvent
- SensorRegisteredEvent (센서 등록 시, 구독: DatasourceGenerator — 재배 생성 시 `devices`로 함께 등록한 센서도 각각 발행됨)
- SensorDeletedEvent (센서 삭제 시, 구독: DatasourceGenerator)

구독 이벤트

- SensorErrorEvent (Rule Engine Service 발행, sensor.status 갱신용)