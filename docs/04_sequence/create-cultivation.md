# 재배 생성 시퀀스

## 개요

사용자가 새로운 버섯 재배를 시작하는 과정입니다.

재배 생성 시 이름/버섯 종류와 함께 연결할 센서 장치도 함께 등록할 수 있으며, Cultivation
Service는 같은 DB에 있는 `mushroom_reference`/`mushroom_reference_threshold`를 직접
조회해 버섯 종류에 맞는 최적 환경 범위를 추천합니다(구 Sensor Service 통합으로 더 이상
OpenFeign 호출이 아닌 같은 트랜잭션 내 조회). 이후 사용자가 버섯 가이드(효능/주의사항,
AI Service)를 확인하고, 추천값을 참고해 직접 환경 설정(위험 한계값)을 입력하고 저장하면
재배가 시작됩니다.

---

# Sequence

```text
Client
↓
API Gateway
↓
Cultivation Service
└── cultivation 생성 (status = CREATED)
↓
mushroom_reference + mushroom_reference_threshold 조회 (같은 DB, 내부 조회)
↓ (디바이스가 있다면)
sensor 배치 등록 (같은 트랜잭션, 실패 시 롤백)
↓
환경 추천 + registeredSensors 반환
↓
Client
↓ (선택)
AI Service — 버섯 가이드 조회 (효능/주의사항, mushroomType 기준)
↓
사용자가 추천 범위를 참고해 목표값 입력
↓
Cultivation Service — Environment Setting 저장 (단일값→범위 변환)
↓
같은 트랜잭션에서 재배 시작 (CREATED → RUNNING)
↓
RabbitMQ (EnvironmentRangeUpdatedEvent) — Rule Engine Service 캐시 갱신용
↓ (이후 상시)
Rule Engine Service — 위험값(min/max) 이탈 시 장치 제어, 중앙값 도달까지 유지
```

---

# 상세 과정

## 1. 재배 생성 요청

```http
POST /api/v1/cultivations
```

```json
{
    "name": "느타리 1호기",
    "mushroomType": "OYSTER",
    "devices": [
        { "deviceEui": "24e124128c067999", "location": "A동", "locationDetail": "선반1", "deviceModel": "Milesight AM107", "sensorTypes": ["TEMPERATURE", "HUMIDITY", "CO2", "LIGHT"] }
    ]
}
```

`devices`는 생략하거나 빈 배열일 수 있습니다. 이 경우 센서 없이 재배만 생성되며, 이후
개별 등록 API(`POST /cultivations/{id}/sensors`)로 추가할 수 있습니다.

---

## 2. Cultivation 생성 + 디바이스 등록

Cultivation Service는 `cultivation` 행을 먼저 생성합니다(초기 상태 `CREATED`, 아직 환경
정보는 저장하지 않음). `devices`가 1개 이상이면 같은 트랜잭션 안에서 `device_eui`별로
`sensor` 행을 생성합니다(PK는 대리키 `id`, `device_eui`는 UNIQUE 제약을 가진 일반
컬럼).

중복 `device_eui`가 있으면 트랜잭션 전체가 롤백되어 `cultivation` 행도 함께 만들어지지
않습니다 — 병합 이전에는 Cultivation Service가 Sensor Service를 OpenFeign으로 호출한
뒤 실패 시 보상 삭제를 하는 방식이었지만, 병합 이후에는 하나의 로컬 트랜잭션이라 더
견고합니다.

디바이스별로 RabbitMQ `SensorRegisteredEvent`를 발행합니다(DatasourceGenerator가
구독해 메모리 캐시를 갱신).

---

## 3. 참조 테이블 조회

Cultivation Service는 같은 DB의 `mushroom_reference` + `mushroom_reference_threshold`를
직접 조회합니다(내부 조회, `GET /api/v1/mushroom-references/{mushroomType}`도 다른
서비스를 위해 내부용으로 계속 제공).

```json
{
    "mushroomType": "OYSTER",
    "mushroomNameKo": "느타리버섯",
    "mushroomNameEn": "Oyster Mushroom",
    "thresholds": [
        { "measurementType": "TEMPERATURE", "min": 15.0, "max": 18.0, "unit": "℃" },
        { "measurementType": "HUMIDITY", "min": 85, "max": 95, "unit": "%" },
        { "measurementType": "CO2", "min": 700, "max": 900, "unit": "ppm" },
        { "measurementType": "LIGHT", "min": 300, "max": 400, "unit": "lux" }
    ]
}
```

---

## 4. 추천 결과 반환

```json
{
    "cultivationId": 3,
    "recommendedEnvironment": {
        "temperature": { "min": 15.0, "max": 18.0 },
        "humidity": { "min": 85, "max": 95 },
        "co2": { "min": 700, "max": 900 },
        "light": { "min": 300, "max": 400 }
    },
    "registeredSensors": [
        { "deviceEui": "24e124128c067999", "sensorId": 12 }
    ]
}
```

사용자는 이 추천 범위를 참고 자료로 확인합니다.

---

## 5. 버섯 가이드 조회 (선택)

Client는 AI Service를 직접 호출해 버섯 효능/재배 시 주의사항을 확인합니다. Cultivation
Service를 거치지 않으며, `cultivationId`도 필요하지 않습니다(`mushroomType`에만 의존).

```http
POST /api/v1/ai/mushroom-guide
```

```json
{ "mushroomType": "OYSTER" }
```

AI Service는 Cultivation Service의 `mushroom_reference` 텍스트 컬럼(characteristics/
healthBenefits/cultivationGuide/additionalInfo)을 RAG 컨텍스트로 사용해 자연스러운
문장으로 재구성하고, `mushroomType` 기준으로 캐싱합니다(TTL 7일). 버섯 종류가 5종
고정이라 같은 종류라면 항상 같은 내용이 반환됩니다.

---

## 6. 환경 입력

사용자가 "설정하러 가기"를 누르면 환경 설정 페이지로 이동합니다. 페이지에는 3번에서
받은 추천 범위가 표시되며, 사용자는 이를 참고해 온도/습도/CO₂/조도 목표값을 직접
입력합니다. 이 값이 실제 자동 제어의 기준(위험 한계값)이 됩니다.

---

## 7. 환경 저장

```http
PATCH /api/v1/cultivations/{cultivationId}/environment
```

Cultivation Service는 요청자가 이 재배의 소유자인지 같은 서비스 안에서 바로 확인한 뒤
(더 이상 별도 OpenFeign 호출이 아님), 단일 목표값을 허용 오차만큼 확장해 범위(min~max)로
변환합니다.

```
Temperature 22℃ → measurementType=TEMPERATURE, min 20.5 / max 23.5
Humidity 91% → measurementType=HUMIDITY, min 86 / max 96
```

항목별로 `environment_setting`에 새 행이 INSERT됩니다(수정하지 않은 항목은 기존 최신
행 유지). API 요청/응답에는 단일 목표값만 노출되며, 범위 변환은 내부 저장 로직입니다.

저장과 같은 트랜잭션 안에서 RabbitMQ로 `EnvironmentRangeUpdatedEvent`를 발행합니다.
Rule Engine Service가 이를 구독해 Redis 캐시(`cultivation:{cultivationId}:range`)를
갱신합니다.

---

## 8. 재배 시작

`cultivation` 상태가 아직 `CREATED`라면, 환경 저장과 같은 트랜잭션 안에서 바로
`RUNNING`으로 전환합니다(멱등 처리 — 이미 `RUNNING`/`FINISHED`인 재배가 환경을
재수정하는 경우에는 상태를 다시 변경하지 않습니다). 병합 이전에는 이 전환이
`EnvironmentRangeUpdatedEvent`를 스스로 구독해 처리하는 방식이었지만, 같은 서비스
안이 되면서 이벤트 왕복 없이 직접 처리합니다.

```
CREATED
↓ (EnvironmentRangeUpdatedEvent 최초 수신)
RUNNING
```

---

## 9. 위험값 초과 시 자동 제어 (이후 상시 동작)

재배가 `RUNNING` 상태가 된 이후, 등록된 센서에서 값이 들어올 때마다 Rule Engine
Service가 `environment_setting`의 min/max와 비교합니다.

```
현재값이 min 미만 또는 max 초과
↓
해당 방향 장치 ON
↓
값이 중앙값 (min+max)/2 에 도달할 때까지 계속 제어
↓
중앙값 도달 시 장치 OFF
```

자세한 내용은 [environment-control.md](./environment-control.md) 참고.

---

# 사용 Database

## Cultivation Service — PostgreSQL

```
cultivation
sensor (devices를 함께 등록한 경우)
mushroom_reference / mushroom_reference_threshold (조회 전용)
environment_setting
```

병합 이전에는 Cultivation DB와 Sensor DB로 나뉘어 있었지만, 이제 하나의 DB이므로 위
테이블들은 모두 같은 트랜잭션 안에서 다룰 수 있습니다.

---

# OpenFeign

```
(참조 테이블 조회, 디바이스 배치 등록, 환경 저장 시 소유권 확인 모두 같은 서비스 내부
호출로 통합되어 더 이상 OpenFeign을 쓰지 않습니다)
```

버섯 가이드는 Client가 AI Service를 직접 호출하며, Cultivation Service를 거치지
않습니다.

---

# RabbitMQ

Publish

```
SensorRegisteredEvent (devices를 함께 등록한 경우, 디바이스별, Cultivation Service 발행)
EnvironmentRangeUpdatedEvent (환경 저장 시점, Cultivation Service 발행)
```

Subscribe

```
DatasourceGenerator (SensorRegisteredEvent)
Rule Engine Service (EnvironmentRangeUpdatedEvent, Redis 캐시 갱신)
```

`CREATED → RUNNING` 상태 전환은 더 이상 이벤트 구독이 아니라 환경 저장과 같은
트랜잭션 안에서 직접 처리됩니다.

---

# 상태 변화

```
CREATED
↓ 환경 저장
RUNNING
↓ 재배 종료 (수확 기록과는 별개 시점)
FINISHED
```

---

# 예외 상황

- 존재하지 않는 버섯 종류 (`mushroom_reference`에 없음)
- `devices`에 이미 등록된 `device_eui` 포함 (같은 트랜잭션 롤백으로 재배 생성 자체가
  취소됨)
- 버섯 가이드 조회 실패 (재배 생성/진행에는 영향 없음, 선택 단계이므로 건너뛸 수 있음)
- 환경 저장 실패
- `EnvironmentRangeUpdatedEvent` 발행 실패 (Rule Engine Service 캐시 갱신만 지연됨 —
  CREATED → RUNNING 전환은 같은 트랜잭션에서 이미 처리되어 영향받지 않음)

---

# 고려 사항

- 추천 환경(`mushroom_reference` 조회 결과)은 Database에 별도로 저장하지 않습니다.
  사용자가 최종 저장한 환경(`environment_setting`)만 저장합니다.
- `mushroom_reference`는 "최적 범위"(참고용), `environment_setting`은 "위험 한계값"
  (자동 제어 기준)으로 목적이 다릅니다.
- 저장 시 단일 목표값은 허용 오차만큼 확장된 범위(min~max)로 변환되어 저장됩니다. API
  스펙은 단일값을 그대로 유지합니다.
- 재배 생성과 디바이스 등록은 하나의 로컬 트랜잭션입니다(병합 이전에는 Cultivation
  Service가 Sensor Service를 동기 호출하고 실패 시 보상 삭제하는 방식으로 "전체 성공
  또는 전체 실패"를 흉내냈지만, 이제는 진짜 트랜잭션으로 보장됩니다).
- 자동 제어는 위험값(min/max) 경계가 아닌 중앙값을 목표로 동작합니다.
