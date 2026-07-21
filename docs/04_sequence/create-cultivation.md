# 재배 생성 시퀀스

## 개요

사용자가 새로운 버섯 재배를 시작하는 과정입니다.

재배 생성 시 이름/버섯 종류와 함께 연결할 센서 장치도 선택할 수 있으며, Cultivation Service는
공공데이터 기반 참조 테이블(`mushroom_reference`)에서 버섯 종류에 맞는 최적 환경 범위를
추천합니다. 이후 사용자가 버섯 가이드(효능/주의사항, AI Service)를 확인하고, 추천값을 참고해
직접 환경 설정(위험 한계값)을 입력하고 저장하면 재배가 시작됩니다.

> ℹ️ **변경 이력**: 원래는 AI Service(Embedding/Vector Search/LLM)를 호출해 추천값을 생성했지만,
> 버섯 종류가 공공데이터 기준 5가지로 고정되어 있어 매번 동일한 값이 나오는 조회에는 AI가
> 불필요하다고 판단, Cultivation Service가 자체 보유한 참조 테이블 조회로 단순화했습니다.

> ℹ️ **변경 이력**: 원래 센서 장치 등록은 재배 생성과 분리된 별도 단계였지만, 재배 생성 화면에서
> 등록할 디바이스도 함께 선택하는 흐름이 자연스러워 재배 생성 요청에 디바이스 목록을 포함할 수
> 있도록 확장했습니다. (여전히 이후 단계에서 개별적으로 센서를 추가/삭제하는 것도 가능합니다.)

> ℹ️ **변경 이력**: 재배 생성 직후 버섯 효능/재배 시 주의사항을 보여주는 "버섯 가이드" 단계가
> 추가되었습니다. Client가 AI Service를 직접 호출하며(Cultivation Service를 거치지 않음),
> 버섯 종류가 5가지로 고정되어 있어 mushroomType 기준으로 캐싱됩니다.

> ℹ️ **변경 이력**: Rule Engine Service의 자동 제어 목표가 "범위 경계"가 아니라 "범위의
> 중앙값(mid)"이라는 점을 명확히 했습니다. 위험값(min/max)을 벗어나 장치가 켜지면, 값이 범위
> 안으로 돌아오는 즉시가 아니라 중앙값에 도달할 때까지 계속 제어합니다. (자세한 내용은
> [rule-Engine.md](../01_Domain/rule-Engine.md) 참고)

---

# Sequence

```text
Client

↓

API Gateway

↓

Cultivation Service

├── mushroom_reference 조회 (mushroomType 기준)
├── Cultivation 생성
└── devices 항목별 sensor 생성 (선택, 하나의 트랜잭션)

↓ (디바이스가 있다면)

RabbitMQ Publish (SensorRegisteredEvent, 디바이스별)

↓

환경 추천 + registeredSensors 반환

↓

Client

↓ (선택)

AI Service — 버섯 가이드 조회 (효능/주의사항, mushroomType 기준)

↓

"설정하러 가기" — 환경 설정 페이지 이동

↓

사용자가 추천 범위를 참고해 목표값 입력/수정

↓

환경 저장

↓

Cultivation Service — Environment Setting 저장 (범위 변환)

↓

재배 시작 (RUNNING)

↓ (이후 상시)

Rule Engine Service — 위험값(min/max) 이탈 시 장치 제어, 중앙값 도달까지 유지
```

---

# 상세 과정

## 1. 재배 생성 요청

사용자는

- 재배 이름
- 버섯 종류
- 등록할 센서 장치 (선택, 여러 개 가능)

를 입력합니다.

예시

```json
{
    "name":"느타리 1호기",
    "mushroomType":"OYSTER",
    "devices":[
        { "deviceEui": "24e124128c067999", "place": "1동 A구역", "location": "선반 2단", "deviceModel": "DHT22", "sensorType": "TEMPERATURE" }
    ]
}
```

`devices`는 생략하거나 빈 배열일 수 있습니다. 이 경우 센서 없이 재배만 생성되며, 이후 별도
API(`POST /cultivations/{cultivationId}/sensors`)로 추가할 수 있습니다.

---

## 2. Cultivation 생성 + 디바이스 등록

Cultivation Service는 아래를 하나의 트랜잭션으로 처리합니다.

- 재배 정보 생성 (초기 상태 `CREATED`, 아직 환경 정보는 저장하지 않음)
- `devices`의 각 항목으로 `sensor` 레코드 생성 (device_eui를 PK로 사용, 서버가 별도 채번하지 않음)

`devices`에 이미 등록된 device_eui가 섞여 있으면(C007) 전체가 롤백됩니다(재배 자체가
생성되지 않음).

디바이스가 1개 이상 등록되었다면, 커밋 후 디바이스별로 RabbitMQ `SensorRegisteredEvent`를
발행합니다. (DatasourceGenerator가 구독해 메모리 캐시를 갱신 — 자세한 내용은
[datasource-generator.md](../01_Domain/datasource-generator.md) 참고)

---

## 3. 참조 테이블 조회

Cultivation Service는

`mushroom_reference` 테이블을 `mushroom_type`으로 조회합니다. (내부 Repository 조회, 외부 서비스 호출 없음)

조회 결과 예시

```json
{
    "mushroomType": "OYSTER",
    "tempMin": 15.0,
    "tempMax": 18.0,
    "humidityMin": 85,
    "humidityMax": 95,
    "co2Min": 700,
    "co2Max": 900,
    "lightMin": 300,
    "lightMax": 400,
    "description": "느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다."
}
```

---

## 4. 추천 결과 반환

Cultivation Service

↓

Client

응답 예시

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

사용자는 이 추천 범위를 참고 자료로 확인합니다. `description`은 mushroom_reference의 짧은
한 줄 문구이며, 다음 단계의 "버섯 가이드"와는 별개입니다.

---

## 5. 버섯 가이드 조회 (선택)

Client는 AI Service를 직접 호출해 버섯 효능/재배 시 주의사항을 확인합니다.
Cultivation Service를 거치지 않으며, `cultivationId`도 필요하지 않습니다(mushroomType에만 의존).

요청 예시

```json
{
    "mushroomType": "OYSTER"
}
```

응답 예시

```json
{
    "mushroomType": "OYSTER",
    "benefits": "느타리버섯은 식이섬유와 베타글루칸이 풍부해 면역력 강화에 도움을 줍니다.",
    "precautions": "다습한 환경을 선호하지만 환기가 부족하면 곰팡이가 발생하기 쉬우니 CO₂ 농도 관리에 유의하세요."
}
```

AI Service는 이 응답을 mushroomType 기준으로 캐싱합니다(TTL 7일). 캐시 미스 시 AI Service는
Cultivation Service를 OpenFeign으로 호출해(`GET /api/v1/mushroom-references/{mushroomType}`)
mushroom_reference의 characteristics/healthBenefits/cultivationGuide/additionalInfo 원문을
가져오고, 이를 LLM 프롬프트의 RAG 컨텍스트로 활용해 benefits/precautions를 생성합니다. 버섯
종류가 5가지로 고정되어 있어 같은 종류라면 항상 같은 내용이 반환됩니다. (자세한 내용은
[ai-api.md](../02_API/ai-api.md)의 "버섯 가이드" 참고)

---

## 6. 환경 수정

사용자가 "설정하러 가기"를 누르면 환경 설정 페이지로 이동합니다. 페이지에는 3번에서 받은
추천 범위(mushroom_reference)가 표시되며, 사용자는 이를 참고하여

- Temperature
- Humidity
- CO₂
- Light

목표값을 직접 입력/수정합니다. 이 값이 실제 자동 제어의 기준(위험 한계값)이 됩니다.

---

## 7. 환경 저장

사용자가

저장 버튼을 누르면

Cultivation Service

↓

단일 목표값을 허용 오차만큼 확장하여 범위(min~max)로 변환

↓

Environment Setting이 생성됩니다.

```text
environment_setting
```

예시 (허용 오차 적용)

```
Temperature 22℃ → type=TEMPERATURE, min 20.5 / max 23.5
Humidity 91% → type=HUMIDITY, min 86 / max 96
```

항목별로 새 행이 INSERT됩니다(수정하지 않은 항목은 기존 최신 행 유지). API 요청/응답에는
단일 목표값만 노출되며, 범위 변환은 Cultivation Service 내부 저장 로직입니다. (자세한 내용은
[cultivation-db.md](../03_Database/cultivation-db.md) 참고)

↓

RabbitMQ Publish (EnvironmentRangeUpdatedEvent)

↓

Rule Engine Service가 구독하여 Redis 캐시(cultivation:{cultivationId}:range)를 갱신합니다.
규칙 평가 시 Cultivation Service를 매번 호출하지 않기 위한 캐시 예열(warm-up) 목적입니다.

---

## 8. 재배 시작

Cultivation 상태 변경

```
CREATED

↓

RUNNING
```

재배가 시작됩니다.

---

## 9. 위험값 초과 시 자동 제어 (이후 상시 동작)

재배가 RUNNING 상태가 된 이후, 등록된 센서에서 값이 들어올 때마다 Rule Engine Service가
environment_setting의 min/max와 비교합니다.

```
현재값이 min 미만 또는 max 초과

↓

해당 방향 장치 ON

↓

값이 중앙값 (min+max)/2 에 도달할 때까지 계속 제어

↓

중앙값 도달 시 장치 OFF
```

제어 목표가 "범위 경계로 복귀"가 아니라 "중앙값 도달"인 이유는 경계 부근에서 값이 미세하게
오르내릴 때 장치가 반복적으로 켜졌다 꺼지는 것(채터링)을 추가로 줄이기 위함입니다. (자세한
내용은 [rule-Engine.md](../01_Domain/rule-Engine.md), [environment-control.md](./environment-control.md) 참고)

---

# 사용 Database

## PostgreSQL

```
mushroom_reference (조회 전용)

cultivation

sensor (devices를 함께 등록한 경우)

environment_setting
```

---

# OpenFeign

재배 생성/환경 추천 단계에서는 다른 서비스를 호출하지 않습니다. (Cultivation Service 내부 조회로 완결)

버섯 가이드는 Client가 AI Service를 직접 호출하며, Cultivation Service를 거치지 않습니다.

---

# RabbitMQ

재배 생성 자체(참조 테이블 조회 등)는 사용자 응답이 필요한 기능이므로 동기 방식(내부 DB 조회)으로 처리합니다.

Publish

```
SensorRegisteredEvent (devices를 함께 등록한 경우, 디바이스별)
EnvironmentRangeUpdatedEvent (환경 저장 시점)
```

Subscribe

```
DatasourceGenerator (SensorRegisteredEvent)
Rule Engine Service (EnvironmentRangeUpdatedEvent, Redis 캐시 갱신)
```

---

# 상태 변화

초기

```
CREATED
```

↓

환경 저장

↓

```
RUNNING
```

↓

수확 완료

↓

```
FINISHED
```

---

# 예외 상황

- 존재하지 않는 버섯 종류 (mushroom_reference에 없음)
- devices에 이미 등록된 device_eui 포함 (재배 생성 자체가 롤백됨)
- 버섯 가이드 조회 실패 (AI Service 오류 — 재배 생성/진행에는 영향 없음, 선택적 단계이므로 건너뛸 수 있음)
- Environment 저장 실패
- DB 저장 실패
- EnvironmentRangeUpdatedEvent 발행 실패 (Rule Engine Service 캐시가 갱신되지 않으며, Rule Engine Service는 다음 규칙 평가 시 Cultivation Service를 직접 호출하는 fallback으로 동작)

---

# 고려 사항

- 추천 환경(mushroom_reference 조회 결과)은 Database에 별도로 저장하지 않습니다.
- 사용자가 최종 저장한 환경(environment_setting)만 저장합니다.
- mushroom_reference는 "최적 범위"(참고용), environment_setting은 "위험 한계값"(자동 제어 기준)으로 목적이 다릅니다.
- 버섯 가이드(효능/주의사항)는 mushroom_reference.description(짧은 참고 문구)과 다른, AI가 생성하는 별도 콘텐츠입니다. 재배 생성 자체와는 독립적인 선택 단계라 실패해도 재배 생성/진행에 영향을 주지 않습니다.
- 저장 시 단일 목표값은 허용 오차만큼 확장된 범위(min~max)로 변환되어 저장됩니다. API 스펙은 단일값을 그대로 유지합니다.
- 환경 저장 응답은 EnvironmentRangeUpdatedEvent 발행(비동기)을 기다리지 않고 즉시 반환합니다.
- 재배는 환경 저장 이후 RUNNING 상태가 됩니다.
- 참조 테이블 조회는 Cultivation Service 내부 DB 조회이므로 AI Service/Embedding Service/Elasticsearch에 대한 의존성이 없습니다.
- 재배 생성과 디바이스 등록은 하나의 트랜잭션이므로, 디바이스 등록이 실패하면 재배도 생성되지 않습니다.
- 자동 제어는 위험값(min/max) 경계가 아닌 중앙값을 목표로 동작합니다. (9번 참고)
