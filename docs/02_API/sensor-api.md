# Sensor API

## 개요

Sensor Service에서 제공하는 REST API 명세입니다.

Rule Engine Service로부터 RabbitMQ(EnvironmentMeasuredEvent)로 전달받아 저장한 센서 측정값을
현재값/통계/차트/주간 리포트 형태로 조회할 수 있습니다. 또한 센서 "장치" 메타데이터(sensor)와
목표 환경(위험 한계값, environment_setting)의 CRUD도 이 API가 제공합니다.

> ℹ️ **변경 이력**: 월간 데이터 조회(`GET /report/monthly`)를 제거했습니다. 재배 기간이 한 달을
> 넘지 않아 월간 리포트 자체가 의미가 없다고 판단했습니다.

> ℹ️ **변경 이력**: 팀 회의 결과 `sensor`/`environment_setting` 테이블과 관련 API가 Cultivation
> Service에서 이 서비스로 완전히 이관되었습니다. 이전에는 "센서 장치 자체의 등록/조회/삭제는
> Cultivation Service API를 확인하세요"라고 안내했지만, 이제 그 API들이 모두 이 문서에
> 포함됩니다. MQTT 수신·규칙 평가·자동 제어는 여전히 Rule Engine Service(REST API 없음,
> rule-Engine.md 참고)의 책임입니다. (자세한 내용은 [sensor-db.md](../03_Database/sensor-db.md),
> [README.md](../README.md)의 결정 사항 #24 참고)

Base URL

```
/api/v1/sensors
```

인증 방식

```
Bearer JWT
```

> ℹ️ **변경 이력**: 이전에는 DatasourceGenerator도 `/api/v1/sensors`로 시작하는 경로(장치 등록)를
> 사용해 경로 충돌 이슈가 있었습니다. 이후 센서 장치 CRUD가 Cultivation Service로 옮겨가며
> 해소되었다가, 이번에 다시 이 서비스로 돌아왔습니다. 측정값 조회 API는 `/cultivations/{id}`
> 하위 경로를 쓰지 않고(`/current`, `/statistics` 등), 장치/환경 CRUD API는 모두
> `/cultivations/{cultivationId}` 하위 경로를 사용하도록 구분해 충돌을 피합니다.

---

# 현재 환경 조회

## GET /current

### Query Parameter

```
cultivationId (required)
```

---

### Process

Sensor Service

↓

Redis 조회

```
cultivation:{cultivationId}:current
```

---

### Response

```json
{
    "temperature": 22.4,
    "humidity": 91.2,
    "co2": 810,
    "light": 430,
    "updatedAt": "2026-08-15T12:30:00"
}
```

---

# 환경 통계 조회

## GET /statistics

### Query Parameter

```
cultivationId (required)
period (required) - 예: 1h, 24h, 7d, 30d
```

---

### Process

Sensor Service

↓

InfluxDB 조회 (기간별 집계)

---

### Response

```json
{
    "averageTemperature": 22.1,
    "averageHumidity": 90.3,
    "averageCo2": 810,
    "averageLight": 430,
    "maxTemperature": 24.5,
    "minTemperature": 20.2,
    "environmentMaintainRate": 95
}
```

---

# 차트 데이터 조회

## GET /chart

### Query Parameter

```
cultivationId (required)
period (required) - 예: 1h, 24h, 7d, 30d
metric (required) - temperature | humidity | co2 | light
```

---

### Response

```json
{
    "metric": "humidity",
    "points": [
        { "timestamp": "2026-08-15T12:00:00", "value": 91.2 },
        { "timestamp": "2026-08-15T12:05:00", "value": 90.8 }
    ]
}
```

---

# 주간 데이터 조회

## GET /report/weekly

내부적으로 AI Service의 리포트 생성에 사용되는 집계 데이터를 반환합니다. Weekly Scheduler가
이 데이터를 AI Service에 전달(push)하는 것과 별개로, 필요 시 이 엔드포인트로 직접 조회할 수도
있습니다.

### Query Parameter

```
cultivationId (required)
```

---

### Response

```json
{
    "averageTemperature": 22.1,
    "averageHumidity": 90.3,
    "averageCo2": 810,
    "averageLight": 430,
    "environmentMaintainRate": 95,
    "autoControlCount": 6
}
```

---

# 센서 등록 (개별)

## POST /cultivations/{cultivationId}

재배 생성 이후 센서를 추가로 등록할 때 사용합니다. (재배 생성과 동시에 등록하려면 아래 "센서
일괄 등록(내부용)"이 대신 사용됩니다.)

### Process

Sensor Service

↓

Cultivation Service에 OpenFeign 호출 (`GET /api/v1/cultivations/{cultivationId}/owner`)로
소유권 확인

↓

소유자 불일치 시 에러 응답(S011)

↓

device_eui 중복 확인

↓

sensor 생성 + RabbitMQ Publish (SensorRegisteredEvent)

---

### Request

```json
{
    "deviceEui": "24e124128c067999",
    "place": "1동 A구역",
    "location": "선반 2단",
    "deviceModel": "DHT22",
    "sensorType": "TEMPERATURE"
}
```

`deviceEui`는 장치 고유 식별자이며 그대로 PK로 사용됩니다(서버가 별도로 채번하지 않음).
LoRaWAN 표준 64비트 DevEUI를 16자리 hex 문자열로 표현한 값이라 문자열(`VARCHAR`)입니다.

---

### Response

```json
{
    "deviceEui": "24e124128c067999",
    "message": "센서가 등록되었습니다."
}
```

---

# 센서 일괄 등록 (내부용)

## POST /cultivations/{cultivationId}/batch

Cultivation Service가 재배 생성 시 `devices`와 함께 센서를 등록하기 위해서만 호출하는 내부용
엔드포인트입니다. 호출자가 이미 Cultivation Service 자신이므로 소유권 확인(`GET
/api/v1/cultivations/{cultivationId}/owner`)을 하지 않습니다.

> ℹ️ **변경 이력**: `sensor` 테이블이 Cultivation Service에서 이관되면서, 재배 생성 시
> `devices`를 함께 등록하던 흐름이 하나의 로컬 트랜잭션에서 서비스 간 동기 호출로 바뀌었습니다.
> 이 호출이 실패하면 Cultivation Service가 방금 생성한 cultivation을 보상 삭제합니다. (자세한
> 내용은 [sensor-db.md](../03_Database/sensor-db.md), [cultivation-api.md](./cultivation-api.md)
> 참고)

### Request

```json
{
    "devices": [
        {
            "deviceEui": "24e124128c067999",
            "place": "1동 A구역",
            "location": "선반 2단",
            "deviceModel": "DHT22",
            "sensorType": "TEMPERATURE"
        }
    ]
}
```

### Response

```json
{
    "registeredSensors": [
        { "deviceEui": "24e124128c067999", "message": "센서가 등록되었습니다." }
    ]
}
```

`devices`에 중복된 device_eui가 있거나 이미 등록된 device_eui가 섞여 있으면(S007) 전체 요청이
실패로 응답되며(전체 성공 또는 전체 실패), Cultivation Service가 이를 보상 삭제 트리거로
사용합니다.

---

# 센서 목록 조회

## GET /cultivations/{cultivationId}

### Response

```json
[
    { "deviceEui": "24e124128c067999", "place": "1동 A구역", "location": "선반 2단", "deviceModel": "DHT22", "sensorType": "TEMPERATURE", "status": "ONLINE" }
]
```

---

# 센서 상세 조회

## GET /cultivations/{cultivationId}/{deviceEui}

### Response

```json
{
    "deviceEui": "24e124128c067999",
    "place": "1동 A구역",
    "location": "선반 2단",
    "deviceModel": "DHT22",
    "sensorType": "TEMPERATURE",
    "status": "ONLINE"
}
```

---

# 센서 삭제

## DELETE /cultivations/{cultivationId}/{deviceEui}

Cultivation Service에 OpenFeign 호출로 소유권을 확인한 뒤 삭제합니다.

### Response

```json
{
    "message": "센서가 삭제되었습니다."
}
```

삭제와 동시에 RabbitMQ로 `SensorDeletedEvent`를 발행합니다.

---

# 환경 설정 저장

## PATCH /cultivations/{cultivationId}/environment

사용자가 추천값(mushroom_reference, Cultivation Service 소유)을 참고하여 실제 자동 제어
기준값(위험 한계값)을 저장합니다. Cultivation Service에 OpenFeign 호출로 소유권을 확인한 뒤
저장합니다.

### Request

```json
{
    "temperature":22
}
```

4개 필드(`temperature`/`humidity`/`co2`/`light`) 모두 선택 항목이며, 보낸 필드만 수정됩니다.
최소 1개 이상의 필드가 있어야 합니다(S009). 물론 기존처럼 여러 필드를 한 번에 보낼 수도 있습니다.

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

저장 시 보낸 필드마다 단일 목표값을 허용 오차만큼 확장한 범위(min~max)로 변환해 새 행을
INSERT합니다(수정하지 않은 항목은 기존 최신 행이 그대로 유지됩니다). 이후 재배의 4개 항목
전체(방금 수정한 것 + 기존 최신 값)를 모아 RabbitMQ로 `EnvironmentRangeUpdatedEvent`를
발행합니다.

---

# 환경 설정 조회

## GET /cultivations/{cultivationId}/environment

### Response

```json
{
    "temperature":22,
    "humidity":91,
    "co2":850,
    "light":420
}
```

각 값은 environment_setting에 저장된 범위의 중간값 `(min+max)/2`을 조회 시점에 계산한
것입니다. 허용 오차가 대칭으로 적용되므로 저장 시 사용자가 입력했던 단일값과 정확히 일치합니다.

---

# 재배 환경 평균 조회 (내부용)

## GET /cultivations/{cultivationId}/environment-average

특정 재배 하나의 환경 평균값(기간 가중 평균)을 조회합니다. 사용자가 아닌 **AI Service가
"인사이트" 조회 시점에 현재 재배 조건을 파악하기 위해서만 호출**하는 내부용 엔드포인트입니다.
재배가 아직 `RUNNING` 상태여도(harvest가 아직 없어도) 호출할 수 있습니다.

### Response

```json
{
    "cultivationId": 27,
    "avgTemperature": 22.1,
    "avgHumidity": 90.5,
    "avgCo2": 810.0,
    "avgLight": 370.0
}
```

`mushroomType`은 이 응답에 포함되지 않습니다. Sensor Service는 `cultivation`에 접근할 수
없어 이 값을 모르며, AI Service는 이미 자신이 호출한 Cultivation Service 응답(미임베딩 목록 등)
에서 받은 `mushroomType`을 그대로 사용합니다.

environment_setting 이력이 없으면 S010 에러를 반환합니다.

---

# 재배 환경 평균 일괄 조회 (내부용)

## POST /environment-averages

여러 cultivationId의 환경 평균을 한 번의 호출로 조회합니다. 사용자가 아닌 **AI Service의
인사이트 배치 스케줄러가 미임베딩 수확 건마다 개별 호출하지 않고 효율적으로 조회하기 위해서만
호출**하는 내부용 엔드포인트입니다.

> ℹ️ **변경 이력**: 인사이트 배치 스케줄러가 매 건마다 단건 조회 API를 호출하면 N+1 호출
> 문제가 생기므로, 배치 처리 효율을 위해 신설했습니다. (자세한 내용은
> [insight.md](../04_sequence/insight.md) 참고)

### Request

```json
{
    "cultivationIds": [12, 15, 18]
}
```

### Response

```json
[
    { "cultivationId": 12, "avgTemperature": 21.8, "avgHumidity": 89.2, "avgCo2": 780.5, "avgLight": 360.0 },
    { "cultivationId": 15, "avgTemperature": 20.1, "avgHumidity": 88.0, "avgCo2": 760.0, "avgLight": 340.0 }
]
```

environment_setting 이력이 없는 cultivationId는 결과 배열에서 제외됩니다(에러를 반환하지
않음 — 배치 특성상 일부 누락을 허용).

---

# 환경 변경 이력 조회 (내부용)

## GET /cultivations/{cultivationId}/environment-history

특정 재배의 environment_setting 변경 이력(항목별 변경 시각, min/max)을 조회합니다. 사용자가
아닌 **AI Service가 "일일 피드백" 기능에서 "언제 몇 도에서 몇 도로 바꿨는지"를 파악하기
위해서만 호출**하는 내부용 엔드포인트입니다.

### Response

```json
{
    "cultivationId": 3,
    "changes": [
        { "type": "TEMPERATURE", "min": 20.5, "max": 23.5, "createdAt": "2026-08-15T09:00:00" }
    ]
}
```

이력이 없으면 빈 배열을 반환합니다(에러가 아님 — 아직 환경을 한 번도 저장하지 않은 재배일 수
있음).

---

# 전체 센서 목록 조회 (내부용)

## GET /

특정 재배로 한정하지 않고, 전체 재배의 센서 목록을 조회합니다. 사용자가 아닌
**DatasourceGenerator가 서비스 재시작 시 캐시(In-Memory)를 재구성하기 위해서만 호출**하는
내부용 엔드포인트입니다. (Base URL이 이미 `/api/v1/sensors`이므로 이 엔드포인트의 전체 경로는
`GET /api/v1/sensors`이며, 다른 센서 API처럼 `/cultivations/{cultivationId}` 하위 경로가
아닌 것에 유의)

### Response

```json
[
    { "deviceEui": "24e124128c067999", "cultivationId": 3, "sensorType": "TEMPERATURE" }
]
```

place/location/deviceModel/status 등 상세 메타데이터는 포함하지 않습니다. DatasourceGenerator는
"어떤 device_eui에 대해 데이터를 생성/발행할지" 판단하는 데 필요한 최소 정보만 필요하기 때문입니다.

> ℹ️ **변경 이력**: 이 엔드포인트는 원래 Cultivation Service가 제공했으나, `sensor` 테이블과
> 함께 Sensor Service로 이관되었습니다. (자세한 내용은
> [cultivation-api.md](./cultivation-api.md), [datasource-generator-api.md](./datasource-generator-api.md) 참고)

---

# Error Code

| Code | Description |
|------|-------------|
| S001 | 존재하지 않는 재배 |
| S002 | 조회 데이터 없음 |
| S003 | 잘못된 period 값 |
| S004 | Redis 조회 실패 |
| S005 | InfluxDB 조회 실패 |
| S006 | RabbitMQ 구독 실패 |
| S007 | 이미 등록된 device_eui (센서 등록 시 중복) |
| S008 | 존재하지 않는 센서 |
| S009 | 환경 설정 저장 시 필드를 하나도 보내지 않음 (temperature/humidity/co2/light 중 최소 1개 필요) |
| S010 | environment_setting 이력이 없어 환경 평균을 계산할 수 없음 (환경을 한 번도 저장하지 않은 재배) |
| S011 | 요청자가 해당 cultivation의 소유자가 아님 (Cultivation Service 소유권 확인 실패) |

> ℹ️ **변경 이력**: S007~S011은 `sensor`/`environment_setting`이 Sensor Service로 이관되며
> Cultivation Service의 기존 에러 코드(C006 존재하지 않는 센서, C007 device_eui 중복, C008
> 환경 필드 미입력, C009 environment_setting 이력 없음)에 대응해 새로 추가했습니다.

---

# OpenFeign

호출받는 서비스

```
AI Service (센서 데이터 조회, 주간 데이터 조회, 환경 평균 단건/배치 조회, 환경 변경 이력 조회)
Cultivation Service (재배 생성 시 센서 일괄 등록)
Rule Engine Service (Redis 캐시 미스 시에만 호출하는 목표 환경 범위 fallback 조회)
```

호출하는 서비스

```
Cultivation Service (센서/환경 쓰기 요청 처리 전 소유권 확인, GET /api/v1/cultivations/{cultivationId}/owner)
```

> ℹ️ **변경 이력**: 이전에는 "호출하는 서비스 없음"이었지만, `sensor`/`environment_setting`
> 관리 책임을 넘겨받으며 개별 등록/수정/삭제 시 Cultivation Service의 소유권 확인 API를
> 호출하게 되었습니다.

Rule Engine Service와 측정값(EnvironmentMeasuredEvent)에 대해서는 REST/OpenFeign으로 직접
통신하지 않으며, RabbitMQ 구독으로만 연결됩니다. 단, 목표 환경 범위 fallback 조회는 예외적으로
Rule Engine Service가 이 서비스를 OpenFeign으로 호출합니다(반대 방향 — 위 "호출받는 서비스"에
해당).

---

# RabbitMQ

Subscribe

```
EnvironmentMeasuredEvent (Rule Engine Service 발행)
SensorErrorEvent (Rule Engine Service 발행, sensor.status 갱신용)
CultivationDeletedEvent (Cultivation Service 발행, sensor/environment_setting 보상 삭제용)
```

`EnvironmentMeasuredEvent` 수신 시 Redis(최신값)/InfluxDB(이력)에 저장합니다.

> ℹ️ **변경 이력**: `SensorErrorEvent`/`CultivationDeletedEvent` 구독이 추가되었습니다.
> `SensorErrorEvent`는 원래 Cultivation Service가 구독했으나 `sensor` 테이블 소유권 이전과
> 함께 옮겨왔고, `CultivationDeletedEvent`는 DB 분리로 사라진 `ON DELETE CASCADE`를 대체하기
> 위해 신설된 이벤트입니다.

---

# Event

발행 이벤트

```
SensorRegisteredEvent (센서 등록 시, 구독: DatasourceGenerator)
SensorDeletedEvent (센서 삭제 시, 구독: DatasourceGenerator)
EnvironmentRangeUpdatedEvent (환경 설정 저장 시, 구독: Rule Engine Service, Cultivation Service)
```

`EnvironmentRangeUpdatedEvent`는 Rule Engine Service(Redis 캐시 갱신)뿐 아니라 Cultivation
Service도 구독합니다. cultivation 상태가 `CREATED`이면 이를 계기로 `RUNNING`으로 전환합니다
(환경 저장과 재배 시작이 더 이상 하나의 로컬 트랜잭션이 아니기 때문).

> ℹ️ **변경 이력**: 원래 "발행하는 이벤트는 없습니다"였으나, `sensor`/`environment_setting`
> 관리 책임을 넘겨받으며 위 3개 이벤트의 발행 주체가 Cultivation Service에서 이 서비스로
> 바뀌었습니다.
