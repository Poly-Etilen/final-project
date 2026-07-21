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
있습니다. (물론 나중에 Sensor Service의 "센서 등록" API로 추가/삭제하는 것도 계속 가능합니다.
[sensor-api.md](./sensor-api.md) 참고)

> ℹ️ **변경 이력**: 원래는 재배 생성(이름+버섯종류)과 센서 등록이 완전히 분리된 API였지만,
> 실제 사용자 흐름상 재배 생성 화면에서 등록할 디바이스도 함께 선택하는 것이 자연스러워
> `devices`를 재배 생성 요청에 포함할 수 있도록 확장했습니다. 당시에는 `sensor` 테이블이 이
> Cultivation DB에 있어 하나의 로컬 트랜잭션으로 처리했습니다.

> ℹ️ **변경 이력**: 팀 회의 결과 `sensor`/`environment_setting` 테이블과 관련 API가 Sensor
> Service로 완전히 이관되었습니다. `devices`로 센서를 함께 등록하는 사용자 흐름 자체는
> 그대로 유지하기로 했지만(사용자 입장에서 재배 생성 화면 하나로 끝나야 함), 더 이상 하나의
> 로컬 트랜잭션이 아닙니다. Cultivation Service는 cultivation 행을 생성한 뒤 Sensor Service의
> 배치 등록 엔드포인트(`POST /api/v1/sensors/cultivations/{cultivationId}/batch`, [sensor-api.md](./sensor-api.md)
> 참고)를 OpenFeign으로 동기 호출하며, 이 호출이 실패하면 방금 생성한 cultivation을 보상
> 삭제(compensating delete)합니다. 진짜 분산 트랜잭션이 아니라 클라이언트 입장에서의 "전체
> 성공 또는 전체 실패"처럼 보이게 하는 근사치이며, 보상 삭제 자체가 실패하는 극단적인 경우는
> 별도 모니터링/재시도 대상으로 남겨둡니다. 개별 센서 등록(재배 생성 이후 추가)은 이제 Sensor
> Service의 `POST /api/v1/sensors/cultivations/{cultivationId}`를 직접 호출합니다. (자세한
> 내용은 [sensor-db.md](../03_Database/sensor-db.md), [README.md](../README.md)의 결정 사항
> #24 참고)

### Request

```json
{
    "name": "느타리버섯 1호기",
    "mushroomType": "OYSTER",
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

`devices`는 선택 항목입니다. 생략하거나 빈 배열이면 센서 없이 재배만 생성되며, 이후 "센서 등록"
API로 추가할 수 있습니다.

---

### Process

Client

↓

Cultivation Service

↓

Cultivation 생성

↓

devices가 1개 이상이면 Sensor Service에 OpenFeign 호출
(`POST /api/v1/sensors/cultivations/{cultivationId}/batch`)

↓

실패 시 방금 생성한 Cultivation을 보상 삭제하고 에러 응답 (C005)

↓

성공 시 mushroom_reference 조회 (mushroom_type 기준)

↓

환경 추천 반환

↓

Client

AI Service를 호출하지 않고, Cultivation Service가 자체 보유한 참조 테이블을 조회합니다.
`devices`의 device_eui 중복/기존 등록 여부 검증은 Sensor Service가 수행하며, 검증에 실패하면
배치 등록 호출 자체가 실패로 응답되어 위 보상 삭제 흐름을 탑니다.

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
        { "deviceEui": "24e124128c067999", "message": "센서가 등록되었습니다." }
    ]
}
```

추천값은 mushroom_reference에 저장된 범위를 그대로 반환합니다. 사용자가 이 범위를 참고해서
Sensor Service의 "환경 설정 저장" API로 실제 자동 제어 기준값을 직접 입력합니다
([sensor-api.md](./sensor-api.md) 참고). `description`은 mushroom_reference의 짧은 참고
문구이며, 버섯 효능/재배 시 주의사항을 다루는 AI 생성 콘텐츠는 [ai-api.md](./ai-api.md)의
"버섯 가이드"를 참고하세요. `devices`를 생략했다면 `registeredSensors`는 빈 배열입니다.
`registeredSensors`는 Sensor Service 배치 등록 호출의 응답을 그대로 전달(pass-through)한
것입니다.

> ℹ️ **변경 이력**: "환경 설정 저장"(`PATCH /{cultivationId}/environment`) API는 Sensor
> Service로 완전히 이관되어 이 문서에서 제거되었습니다. [sensor-api.md](./sensor-api.md)를
> 참고하세요.

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
    "mushroomType":"OYSTER",
    "status":"RUNNING",
    "createdAt":"2026-08-01"
}
```

> ℹ️ **변경 이력**: `environment` 필드(목표 환경값)가 이 응답에서 제거되었습니다.
> `environment_setting`이 Sensor Service로 이관되면서 Cultivation Service가 더 이상 직접
> 조회할 수 없기 때문입니다. Cultivation Service가 매 조회마다 Sensor Service를 동기
> 호출해 조합(aggregate)하는 대신, 클라이언트가 필요 시 Sensor Service의 환경 조회 API를
> 별도로 호출하도록 책임을 분리했습니다(다른 서비스도 이미 이 패턴을 따릅니다 — 예: AI
> Service의 인사이트 배치는 Cultivation Service와 Sensor Service를 각각 호출해 필요한
> 데이터를 조합합니다). (자세한 내용은 [sensor-api.md](./sensor-api.md) 참고)

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

> ℹ️ **변경 이력**: 센서 등록/목록조회/상세조회/삭제(`POST`/`GET`/`GET`/`DELETE
> /{cultivationId}/sensors...`) API는 `sensor` 테이블과 함께 Sensor Service로 완전히
> 이관되어 이 문서에서 제거되었습니다. [sensor-api.md](./sensor-api.md)를 참고하세요.

---

# 재배 소유권 확인 (내부용)

## GET /{cultivationId}/owner

특정 cultivation의 소유자(userId)와 상태를 조회합니다. 사용자가 아닌 **Sensor Service가
센서/환경 관련 쓰기 요청(등록·수정·삭제)을 처리하기 전 요청자가 해당 cultivation의 소유자가
맞는지 확인하기 위해서만 호출**하는 내부용 엔드포인트입니다.

> ℹ️ **변경 이력**: `sensor`/`environment_setting`이 Sensor Service로 이관되면서 Sensor
> Service는 더 이상 `cultivation.user_id`에 직접 접근할 수 없게 되었습니다. 이 엔드포인트로
> 소유권 확인 책임을 대신합니다. 단, Cultivation Service가 재배 생성 시 호출하는 배치 등록
> (`POST /api/v1/sensors/cultivations/{cultivationId}/batch`)은 호출자가 Cultivation
> Service 자신이므로 이 확인을 거치지 않습니다. (자세한 내용은
> [sensor-db.md](../03_Database/sensor-db.md) 참고)

### Response

```json
{
    "userId": 7,
    "status": "RUNNING"
}
```

---

# 버섯 참조 데이터 조회 (내부용)

## GET /api/v1/mushroom-references/{mushroomType}

`mushroom_reference`의 원문 텍스트 데이터를 조회합니다. 사용자가 아닌 **AI Service가 "버섯
가이드"(`POST /ai/mushroom-guide`) 생성 시 RAG 컨텍스트로 사용하기 위해서만 호출**하는
내부용 엔드포인트입니다.

> ℹ️ **변경 이력**: `mushroom_reference`에 이름/특성/효능/재배 가이드/추가 정보 컬럼이
> 추가되면서, AI Service가 "버섯 가이드"를 생성할 때 이 원문 데이터를 참고할 수 있도록 이
> 엔드포인트를 추가했습니다. (자세한 내용은 [cultivation-db.md](../03_Database/cultivation-db.md)
> 참고)

### Response

```json
{
    "mushroomType": "OYSTER",
    "mushroomNameKo": "느타리버섯",
    "mushroomNameEn": "Oyster Mushroom",
    "mushroomScientificName": "Pleurotus ostreatus",
    "characteristics": "군생하며 갓은 회갈색~담회색을 띠고, 균사 성장 속도가 빠른 편입니다.",
    "healthBenefits": "식이섬유와 베타글루칸이 풍부해 면역력 강화와 콜레스테롤 감소에 도움을 줍니다.",
    "cultivationGuide": "다습한 환경을 선호하지만 환기가 부족하면 곰팡이가 발생하기 쉬우니 CO₂ 농도 관리에 유의해야 합니다.",
    "additionalInfo": null
}
```

환경 범위(temp/humidity/co2/light)와 `description`(짧은 참고 문구)은 이미 "재배 생성" 응답에
포함되므로 이 엔드포인트 응답에는 포함하지 않습니다.

---

# 버섯 참조 데이터 전체 목록 조회 (내부용)

## GET /api/v1/mushroom-references

`mushroom_reference`의 전체 목록(모든 필드)을 조회합니다. **Embedding Service가 Elasticsearch
전체 재생성(예: 임베딩 모델 교체) 시에만 호출**하는 내부용 엔드포인트입니다. 평상시 동기화는
`MushroomReferenceUpdatedEvent`로 이루어지므로 이 호출은 드뭅니다.

### Response

```json
[
    {
        "mushroomType": "OYSTER",
        "mushroomNameKo": "느타리버섯",
        "mushroomNameEn": "Oyster Mushroom",
        "mushroomScientificName": "Pleurotus ostreatus",
        "temperature": {"min": 15.0, "max": 18.0},
        "humidity": {"min": 85, "max": 95},
        "co2": {"min": 700, "max": 900},
        "light": {"min": 300, "max": 400},
        "description": "느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다.",
        "characteristics": "군생하며 갓은 회갈색~담회색을 띠고, 균사 성장 속도가 빠른 편입니다.",
        "healthBenefits": "식이섬유와 베타글루칸이 풍부해 면역력 강화와 콜레스테롤 감소에 도움을 줍니다.",
        "cultivationGuide": "다습한 환경을 선호하지만 환기가 부족하면 곰팡이가 발생하기 쉬우니 CO₂ 농도 관리에 유의해야 합니다.",
        "additionalInfo": null
    }
]
```

---

> ℹ️ **변경 이력**: 센서 전체 목록 조회(`GET /api/v1/sensors`, DatasourceGenerator가 재시작 시
> 캐시 재구성을 위해 호출하던 내부용 엔드포인트)는 `sensor` 테이블과 함께 Sensor Service로
> 이관되었습니다. DatasourceGenerator는 이제 재시작 시 Cultivation Service가 아닌 Sensor
> Service를 호출합니다. (자세한 내용은 [sensor-api.md](./sensor-api.md),
> [datasource-generator-api.md](./datasource-generator-api.md) 참고)

---

# 미임베딩 수확 건수 조회 (내부용)

## GET /api/v1/harvests/unembedded-count

"인사이트" 기능을 위해 아직 Elasticsearch에 임베딩되지 않은 수확(harvest) 건수를 조회합니다.
사용자가 아닌 **AI Service가 00시 배치 스케줄러에서 임계치(20건) 도달 여부를 판단하기 위해서만
호출**하는 내부용 엔드포인트입니다.

> ℹ️ **변경 이력**: "인사이트" 기능(같은 버섯 종류 + 유사한 온도로 재배했던 타인의 사례를 바탕으로
> 피드백을 주는 기능) 추가를 위해 신설했습니다. AI Service는 별도의 워터마크(마지막으로 처리한
> 시각/ID 등)를 관리하지 않고, 이 엔드포인트로 단순 조회만 합니다. "임베딩 여부"의 소유권은
> Cultivation Service(`harvest.is_embedded`)에 있습니다. (자세한 내용은
> [insight.md](../04_sequence/insight.md) 참고)

### Response

```json
{
    "count": 23
}
```

전체 사용자의 수확을 합산한 전역 카운트이며, 특정 사용자로 한정하지 않습니다.

---

# 미임베딩 수확 목록 조회 (내부용)

## GET /api/v1/harvests/unembedded

임베딩되지 않은 수확 건의 상세 목록을 조회합니다. 사용자가 아닌 **AI Service가 위 카운트가
임계치(20건) 이상일 때, 실제 배치 임베딩 대상 데이터를 가져오기 위해서만 호출**하는 내부용
엔드포인트입니다.

### Response

```json
[
    {
        "cultivationId": 12,
        "mushroomType": "OYSTER",
        "harvestWeight": 3200
    }
]
```

> ℹ️ **변경 이력**: 이 응답에 있던 `avgTemperature`/`avgHumidity`/`avgCo2`/`avgLight`(기간
> 가중 평균)는 `environment_setting`이 Sensor Service로 이관되면서 더 이상 Cultivation
> Service가 계산할 수 없어 제거했습니다. AI Service의 인사이트 배치 스케줄러는 이제 이
> 엔드포인트로 미임베딩 수확 목록(cultivationId 등)을 받은 뒤, Sensor Service의
> `POST /api/v1/sensors/environment-averages`(cultivationId 목록을 보내는 일괄 조회)를
> 별도로 호출해 환경 평균을 채워 넣습니다. (자세한 내용은 [sensor-api.md](./sensor-api.md),
> [insight.md](../04_sequence/insight.md) 참고)

생육 점수(growthScore)는 이 응답에 포함되지 않습니다 — AI DB(`growth_record`)에 있는 데이터라
AI Service가 자체적으로 채워 넣습니다.

---

# 수확 임베딩 완료 처리 (내부용)

## PATCH /api/v1/harvests/embedded

지정한 수확 건들의 `is_embedded`를 `TRUE`로 갱신합니다. 사용자가 아닌 **AI Service가
Embedding Service로의 배치 임베딩 호출에 성공한 직후에만 호출**하는 내부용 엔드포인트입니다.

### Request

```json
{
    "cultivationIds": [12, 15, 18]
}
```

### Response

```json
{
    "message": "23 harvests marked as embedded"
}
```

이미 `is_embedded = TRUE`인 건이 섞여 있어도 오류 없이 무시합니다(멱등). 임베딩 자체가 실패한
경우 이 엔드포인트를 호출하지 않으며, 해당 건들은 다음 배치 때 다시 미임베딩 목록에 포함됩니다.

---

> ℹ️ **변경 이력**: 재배 환경 평균 조회(`GET /api/v1/cultivations/{cultivationId}/environment-average`,
> AI Service가 "인사이트" 조회 시점에 현재 재배 조건을 파악하기 위해 호출하던 내부용
> 엔드포인트)는 `environment_setting`과 함께 Sensor Service로 이관되었습니다. AI Service는
> 이제 Sensor Service의 `GET /api/v1/sensors/cultivations/{cultivationId}/environment-average`를
> 호출합니다. (자세한 내용은 [sensor-api.md](./sensor-api.md), [insight.md](../04_sequence/insight.md)
> 참고)

---

# Error Code

| Code | Description |
|------|-------------|
| C001 | 존재하지 않는 재배 |
| C002 | 이미 종료된 재배 |
| C003 | 권한 없음 |
| C004 | 지원하지 않는 버섯 종류 (mushroom_reference에 없음) |
| C005 | Sensor Service 배치 등록 실패로 재배 생성이 취소됨 (보상 삭제 처리) |

> ℹ️ **변경 이력**: 센서/환경 관련 에러 코드(기존 C004 환경 저장 실패, C006 존재하지 않는 센서,
> C007 device_eui 중복, C008 환경 필드 미입력, C009 environment_setting 이력 없음)는 해당
> API가 Sensor Service로 이관되면서 함께 제거되었습니다. Sensor Service의 에러 코드는
> [sensor-api.md](./sensor-api.md)를 참고하세요. 남은 코드는 C001~C005로 재정리했습니다.

---

# OpenFeign

사용 서비스

- AI Service ("버섯 가이드" 생성 시 `GET /api/v1/mushroom-references/{mushroomType}` 호출. 생육 사진 Vision 분석은 반대로 Cultivation Service가 AI Service를 호출하는 방향이라 여기 해당하지 않음. "인사이트" 기능을 위해 `GET /api/v1/harvests/unembedded-count`, `GET /api/v1/harvests/unembedded`, `PATCH /api/v1/harvests/embedded`도 호출)
- Sensor Service (센서/환경 쓰기 요청 처리 전 소유권 확인을 위해 `GET /{cultivationId}/owner` 호출)
- Embedding Service (Elasticsearch 전체 재생성 시에만, `GET /api/v1/mushroom-references` 전체 목록 조회. 평상시에는 MushroomReferenceUpdatedEvent로만 동기화)

> ℹ️ **변경 이력**: DatasourceGenerator는 더 이상 Cultivation Service를 호출하지 않습니다.
> 재시작 시 캐시 재구성용 전체 센서 목록 조회(`GET /api/v1/sensors`)가 Sensor Service로
> 이관되었기 때문입니다. (자세한 내용은 [datasource-generator-api.md](./datasource-generator-api.md) 참고)

Cultivation Service도 재배 생성 시 Sensor Service를 OpenFeign으로 호출합니다(배치 등록,
`POST /api/v1/sensors/cultivations/{cultivationId}/batch`). 이 호출은 위 "사용 서비스" 목록과
반대 방향(Cultivation Service → Sensor Service)이며, 실패 시 보상 삭제로 처리합니다.

---

# Event

발행 이벤트

- CultivationCreatedEvent
- CultivationFinishedEvent
- CultivationDeletedEvent (재배 삭제 시, 구독: Sensor Service — 해당 cultivation_id의 sensor/environment_setting을 정리하는 보상 삭제용)
- MushroomReferenceUpdatedEvent (관리자가 mushroom_reference를 등록/수정할 때, 구독: Embedding Service — Elasticsearch의 mushroom_environment 인덱스 갱신용)

> ℹ️ **변경 이력**: `EnvironmentRangeUpdatedEvent`/`SensorRegisteredEvent`/`SensorDeletedEvent`는
> 발행 주체가 Sensor Service로 바뀌면서 이 문서에서 제거되었습니다. 대신 DB 레벨
> `ON DELETE CASCADE`를 대체할 `CultivationDeletedEvent`를 새로 추가했습니다(cultivation_id가
> 다른 DB를 참조하는 순수 값이라 CASCADE가 걸리지 않습니다). (자세한 내용은
> [sensor-db.md](../03_Database/sensor-db.md) 참고)

구독 이벤트

- EnvironmentRangeUpdatedEvent (Sensor Service 발행) — cultivation 상태가 `CREATED`이면
  `RUNNING`으로 전환. `environment_setting`이 Sensor Service로 이관되며 "환경 저장 + 상태
  전환"이 하나의 로컬 트랜잭션일 수 없게 되어 이벤트 기반으로 대체했습니다.

(기존 `SensorErrorEvent` 구독은 `sensor.status` 갱신용이었는데, 해당 컬럼이 Sensor Service로
이관되면서 구독 주체도 Sensor Service로 바뀌었습니다.)