# 재배 생성 시퀀스

## 개요

사용자가 새로운 버섯 재배를 시작하는 과정입니다.

재배 생성 시 Cultivation Service는 공공데이터 기반 참조 테이블(`mushroom_reference`)에서
버섯 종류에 맞는 최적 환경 범위를 조회하여 추천하며,
사용자가 추천값을 참고해 직접 환경 설정(위험 한계값)을 입력하고 저장하면 재배가 시작됩니다.

> ℹ️ **변경 이력**: 원래는 AI Service(Embedding/Vector Search/LLM)를 호출해 추천값을 생성했지만,
> 버섯 종류가 공공데이터 기준 5가지로 고정되어 있어 매번 동일한 값이 나오는 조회에는 AI가
> 불필요하다고 판단, Cultivation Service가 자체 보유한 참조 테이블 조회로 단순화했습니다.

---

# Sequence

```text
Client

↓

API Gateway

↓

Cultivation Service

↓

PostgreSQL

↓

Cultivation 생성

↓

mushroom_reference 조회 (mushroom_type 기준)

↓

환경 추천 생성

↓

Client

↓

사용자 확인

↓

환경 저장

↓

Cultivation Service

↓

Environment Setting 저장

↓

재배 시작
```

---

# 상세 과정

## 1. 재배 생성 요청

사용자는

- 재배 이름
- 버섯 종류

를 입력합니다.

예시

```json
{
    "name":"느타리 1호기",
    "mushroomType":"OYSTER"
}
```

---

## 2. Cultivation 생성

Cultivation Service는

재배 정보를 생성합니다.

초기 상태

```
CREATED
```

아직 환경 정보는 저장하지 않습니다.

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
    "description": "느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다."
}
```

사용자는 이 추천 범위를 참고 자료로 확인합니다.

---

## 5. 환경 수정

사용자는 추천 범위(mushroom_reference)를 참고하여

- Temperature
- Humidity
- CO₂
- Light

목표값을 직접 입력/수정합니다. 이 값이 실제 자동 제어의 기준(위험 한계값)이 됩니다.

---

## 6. 환경 저장

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
Temperature 22℃ → temp_min 20.5 / temp_max 23.5
Humidity 91% → humidity_min 86 / humidity_max 96
```

API 요청/응답에는 단일 목표값만 노출되며, 범위 변환은 Cultivation Service 내부 저장 로직입니다.

↓

RabbitMQ Publish (EnvironmentRangeUpdatedEvent)

↓

Rule Engine Service가 구독하여 Redis 캐시(cultivation:{cultivationId}:range)를 갱신합니다.
규칙 평가 시 Cultivation Service를 매번 호출하지 않기 위한 캐시 예열(warm-up) 목적입니다.

---

## 7. 재배 시작

Cultivation 상태 변경

```
CREATED

↓

RUNNING
```

재배가 시작됩니다.

---

# 사용 Database

## PostgreSQL

```
mushroom_reference (조회 전용)

cultivation

environment_setting
```

---

# OpenFeign

재배 생성/환경 추천 단계에서는 다른 서비스를 호출하지 않습니다. (Cultivation Service 내부 조회로 완결)

---

# RabbitMQ

재배 생성 자체(참조 테이블 조회 등)는 사용자 응답이 필요한 기능이므로 동기 방식(내부 DB 조회)으로 처리합니다.

환경 저장(6단계) 시점에는 비동기로 아래 이벤트를 발행합니다.

Publish

```
EnvironmentRangeUpdatedEvent
```

Subscribe

```
Rule Engine Service (Redis 캐시 갱신)
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
- Environment 저장 실패
- DB 저장 실패
- EnvironmentRangeUpdatedEvent 발행 실패 (Rule Engine Service 캐시가 갱신되지 않으며, Rule Engine Service는 다음 규칙 평가 시 Cultivation Service를 직접 호출하는 fallback으로 동작)

---

# 고려 사항

- 추천 환경(mushroom_reference 조회 결과)은 Database에 별도로 저장하지 않습니다.
- 사용자가 최종 저장한 환경(environment_setting)만 저장합니다.
- mushroom_reference는 "최적 범위"(참고용), environment_setting은 "위험 한계값"(자동 제어 기준)으로 목적이 다릅니다.
- 저장 시 단일 목표값은 허용 오차만큼 확장된 범위(min~max)로 변환되어 저장됩니다. API 스펙은 단일값을 그대로 유지합니다.
- 환경 저장 응답은 EnvironmentRangeUpdatedEvent 발행(비동기)을 기다리지 않고 즉시 반환합니다.
- 재배는 환경 저장 이후 RUNNING 상태가 됩니다.
- 참조 테이블 조회는 Cultivation Service 내부 DB 조회이므로 AI Service/Embedding Service/Elasticsearch에 대한 의존성이 없습니다.