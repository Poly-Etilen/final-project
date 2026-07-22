# 재배 생성 시퀀스

## 개요

사용자가 새로운 버섯 재배를 시작하는 과정입니다.

재배 생성 시 이름/버섯 종류와 함께 연결할 센서 장치도 함께 등록할 수 있으며, Cultivation
Service는 Sensor Service가 소유한 `mushroom_reference`/`mushroom_reference_threshold`를
OpenFeign으로 조회해 버섯 종류에 맞는 최적 환경 범위를 추천합니다. 이후 사용자가 버섯
가이드(효능/주의사항, AI Service)를 확인하고, 추천값을 참고해 직접 환경 설정(위험
한계값)을 입력하고 저장하면 재배가 시작됩니다.

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
Sensor Service OpenFeign 호출 (mushroom_reference + mushroom_reference_threshold 조회)
↓ (디바이스가 있다면)
Sensor Service OpenFeign 호출 (배치 등록) — 실패 시 cultivation 보상 삭제
↓
환경 추천 + registeredSensors 반환
↓
Client
↓ (선택)
AI Service — 버섯 가이드 조회 (효능/주의사항, mushroomType 기준)
↓
사용자가 추천 범위를 참고해 목표값 입력
↓
Sensor Service — Environment Setting 저장 (단일값→범위 변환, 소유권 확인 후)
↓
RabbitMQ (EnvironmentRangeUpdatedEvent)
↓
Cultivation Service — 재배 시작 (CREATED → RUNNING)
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
Sensor Service의 개별 등록 API(`POST /api/v1/sensors/cultivations/{cultivationId}`)로
추가할 수 있습니다.

---

## 2. Cultivation 생성 + 디바이스 등록

Cultivation Service는 `cultivation` 행을 먼저 생성합니다(초기 상태 `CREATED`, 아직 환경
정보는 저장하지 않음). `devices`가 1개 이상이면 Sensor Service의 배치 등록 엔드포인트를
OpenFeign으로 동기 호출합니다(`POST /api/v1/sensors/cultivations/{cultivationId}/batch`).

이 호출이 실패하면(중복 `device_eui` 포함) Cultivation Service는 방금 생성한
`cultivation`을 보상 삭제(compensating delete)하고 에러를 응답합니다. 진짜 분산
트랜잭션은 아니지만, 클라이언트 입장에서는 "전체 성공 또는 전체 실패"처럼 보입니다.

Sensor Service는 `device_eui`별로 `sensor` 행을 생성하고(PK는 대리키 `id`, `device_eui`는
UNIQUE 제약을 가진 일반 컬럼), 디바이스별로 RabbitMQ `SensorRegisteredEvent`를
발행합니다(DatasourceGenerator가 구독해 메모리 캐시를 갱신).

---

## 3. 참조 테이블 조회

Cultivation Service는 Sensor Service를 OpenFeign으로 호출해 `mushroom_reference` +
`mushroom_reference_threshold`를 조회합니다(`GET /api/v1/mushroom-references/{mushroomType}`).

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

AI Service는 Sensor Service의 `mushroom_reference` 텍스트 컬럼(characteristics/
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
PATCH /api/v1/sensors/cultivations/{cultivationId}/environment
```

Sensor Service는 Cultivation Service를 OpenFeign으로 호출해(`GET
/api/v1/cultivations/{cultivationId}/owner`) 소유권을 확인한 뒤, 단일 목표값을 허용
오차만큼 확장해 범위(min~max)로 변환합니다.

```
Temperature 22℃ → measurementType=TEMPERATURE, min 20.5 / max 23.5
Humidity 91% → measurementType=HUMIDITY, min 86 / max 96
```

항목별로 `environment_setting`에 새 행이 INSERT됩니다(수정하지 않은 항목은 기존 최신
행 유지). API 요청/응답에는 단일 목표값만 노출되며, 범위 변환은 Sensor Service 내부
저장 로직입니다.

저장 직후 RabbitMQ로 `EnvironmentRangeUpdatedEvent`를 발행합니다. Rule Engine
Service가 이를 구독해 Redis 캐시(`cultivation:{cultivationId}:range`)를 갱신합니다.

---

## 8. 재배 시작

Cultivation Service도 같은 `EnvironmentRangeUpdatedEvent`를 구독합니다. `cultivation`
상태가 아직 `CREATED`라면 이를 계기로 `RUNNING`으로 전환합니다(멱등 처리 — 이미
`RUNNING`/`FINISHED`인 재배가 환경을 재수정하는 경우에는 상태를 다시 변경하지 않습니다).

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
```

## Sensor Service — PostgreSQL

```
sensor (devices를 함께 등록한 경우)
mushroom_reference / mushroom_reference_threshold (조회 전용)
environment_setting
```

---

# OpenFeign

```
Cultivation Service → Sensor Service (참조 테이블 조회, 디바이스 배치 등록)
Sensor Service → Cultivation Service (환경 저장 시 소유권 확인)
```

버섯 가이드는 Client가 AI Service를 직접 호출하며, Cultivation Service를 거치지
않습니다.

---

# RabbitMQ

Publish

```
SensorRegisteredEvent (devices를 함께 등록한 경우, 디바이스별, Sensor Service 발행)
EnvironmentRangeUpdatedEvent (환경 저장 시점, Sensor Service 발행)
```

Subscribe

```
DatasourceGenerator (SensorRegisteredEvent)
Rule Engine Service (EnvironmentRangeUpdatedEvent, Redis 캐시 갱신)
Cultivation Service (EnvironmentRangeUpdatedEvent, CREATED → RUNNING 상태 전환)
```

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
- 참조 테이블 조회(Sensor Service OpenFeign 호출) 실패/타임아웃
- `devices`에 이미 등록된 `device_eui` 포함 (배치 등록 실패 → cultivation 보상 삭제)
- Sensor Service 배치 등록 호출 자체가 실패/타임아웃 — 동일하게 보상 삭제 처리
- 버섯 가이드 조회 실패 (재배 생성/진행에는 영향 없음, 선택 단계이므로 건너뛸 수 있음)
- 환경 저장 실패 (Sensor Service의 소유권 확인 실패 포함)
- `EnvironmentRangeUpdatedEvent` 발행 실패 (Rule Engine Service 캐시 갱신 지연, Cultivation
  Service의 CREATED → RUNNING 전환도 지연될 수 있음)

---

# 고려 사항

- 추천 환경(`mushroom_reference` 조회 결과)은 Database에 별도로 저장하지 않습니다.
  사용자가 최종 저장한 환경(`environment_setting`)만 저장합니다.
- `mushroom_reference`는 "최적 범위"(참고용), `environment_setting`은 "위험 한계값"
  (자동 제어 기준)으로 목적이 다릅니다.
- 저장 시 단일 목표값은 허용 오차만큼 확장된 범위(min~max)로 변환되어 저장됩니다. API
  스펙은 단일값을 그대로 유지합니다.
- 재배 생성과 디바이스 등록은 하나의 로컬 트랜잭션이 아닙니다. Cultivation Service가
  Sensor Service를 동기 호출하고, 실패 시 보상 삭제로 "전체 성공 또는 전체 실패"에
  근접한 동작을 흉내냅니다(진짜 분산 트랜잭션은 아님).
- 자동 제어는 위험값(min/max) 경계가 아닌 중앙값을 목표로 동작합니다.
