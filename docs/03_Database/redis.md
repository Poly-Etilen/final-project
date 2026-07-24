# Redis

## 개요

Redis는 실시간 데이터와 임시 데이터를 저장하기 위해 사용합니다.

본 프로젝트에서는 영구적으로 저장할 필요가 없는 데이터와
빠른 조회가 필요한 데이터를 관리합니다.

---

# 사용 서비스

| Service | 용도 |
|----------|------|
| Auth Service | Refresh Token |
| Auth Service | 이메일 인증 |
| AI Service | AI 응답 캐시 |
| Rule Engine Service | 목표 환경 범위 캐시 |
| Cultivation Service | 최신 센서 데이터 |

---

# Key 구조

## Auth

### Refresh Token

Key

```
refresh:{userId}
```

Example

```
refresh:15
```

Value

```
JWT Refresh Token
```

TTL

```
14일
```

---

### Email Verification

Key

```
email:{email}
```

Example

```
email:test@example.com
```

Value

```
845132
```

TTL

```
5분
```

---

# AI

## AI 챗봇 응답 Cache

사용자의 동일한 질문에 대해
LLM 호출을 줄이기 위해 사용합니다.

Key

```
ai:{hash}
```

hash는 cultivationId와 질문 내용을 조합하여 생성합니다.

Example

```
ai:82ad7f19...
```

Value

```json
{
  "answer": "...",
  "createdAt": "2026-08-15T10:22:30"
}
```

TTL

```
24시간
```

---

## AI 생육 분석 결과 Cache

Vision 분석은 비용이 크므로, 동일 재배의 최근 분석 결과를 재사용하기 위해 캐싱합니다.
(`GET /cultivations/{cultivationId}/analysis`가 새로 분석하지 않고 이 캐시를 반환합니다.)
같은 분석 결과는 이 캐시와 별개로 `growth_record` 테이블(PostgreSQL, AI DB)에도 영구
저장됩니다. 이 캐시는 TTL이 지나면 사라지는 "빠른 재조회"용이고, `growth_record`는 만료되지
않는 "이력 비교"용입니다(일일 피드백에 사용). 아래 Value 구조는 `growth_record.analysis_data`
(JSONB)에 저장되는 필드 구성과 동일합니다 — 다만 `growth_record`는 여기에 더해
`cultivation_photo_id`(사진 소프트 참조)와 `analyzed_at`을 함께 컬럼으로 갖습니다.
(자세한 내용은 [ai-db.md](./ai-db.md), [daily-feedback.md](../04_sequence/daily-feedback.md) 참고)

Key

```
ai:{cultivationId}:analysis
```

Example

```
ai:3:analysis
```

Value

```json
{
  "growthScore": 85,
  "myceliumGrowthRate": 82,
  "capSize": "중(3.2cm)",
  "colorStatus": "정상",
  "diseaseStatus": "정상",
  "growthStage": "자실체 형성기",
  "expectedHarvestDate": "2026-08-20"
}
```

TTL

```
6시간
```

---

## 버섯 가이드 Cache

버섯 종류(공공데이터 기준 5가지 고정)별 효능/재배 주의사항은 항상 같은 내용이므로,
재배(cultivationId)가 아닌 mushroomId 기준으로 캐싱해 반복 LLM 호출을 피합니다.

Key

```
ai:mushroom:{mushroomId}:guide
```

Example

```
ai:mushroom:3:guide
```

Value

```json
{
  "benefits": "...",
  "precautions": "..."
}
```

TTL

```
7일
```

다른 AI 캐시보다 TTL이 긴 이유는, 이 값이 특정 재배가 아니라 버섯 종류 자체에 대한 고정적인
설명이라 자주 바뀔 필요가 없기 때문입니다.

---

## 인사이트 후보 Cache

같은 버섯 종류 + 유사한 환경으로 재배했던 타인의 사례 후보 리스트입니다. 웹 챗봇의
`/인사이트` 명령어로 요청 시(Cache Miss)에만 AI DB `insight` 테이블 검색(SQL 필터)을
거쳐 채워집니다. LLM은 호출하지 않습니다 — 요약은 적재 시점에 이미 만들어 둔
`insight.summary`를 후보 상세 조회 때 재사용합니다. (자세한 내용은
[insight.md](../04_sequence/insight.md) 참고)

Key

```
ai:{cultivationId}:insight:candidates
```

Example

```
ai:27:insight:candidates
```

Value

```json
{
  "candidates": [
    { "insightId": 41, "growthScore": 88, "harvestWeight": 3200, "createdAt": "2026-08-10T09:00:00" },
    { "insightId": 39, "growthScore": 82, "harvestWeight": 2900, "createdAt": "2026-08-05T09:00:00" }
  ]
}
```

TTL

```
24시간
```

유사 사례가 하나도 매칭되지 않아 빈 리스트로 응답한 경우는 캐시에 저장하지 않습니다.
새로운 수확이 계속 기록되며 `insight` 테이블에 사례가 쌓이므로, 다음 요청 시점에는
매칭될 수 있기 때문입니다. 후보를 선택한 뒤의 상세 조회(날짜별 환경/피드백)는 캐시하지
않습니다 — `cultivation_id` + 날짜 범위의 가벼운 인덱스 조회라 캐싱 이득이 크지
않습니다.

---

# Rule Engine

## 목표 환경 범위 캐시

규칙 평가 시마다 Cultivation Service를 호출하지 않도록,
목표 환경 범위(min~max)를 캐싱해 둡니다.

Key

```
cultivation:{cultivationId}:range
```

Example

```
cultivation:3:range
```

Value

```json
{
  "tempMin": 20.5,
  "tempMax": 23.5,
  "humidityMin": 85,
  "humidityMax": 95,
  "co2Min": 750,
  "co2Max": 850,
  "lightMin": 320,
  "lightMax": 380,
  "updatedAt": "2026-08-15T09:00:00"
}
```

TTL

```
24시간
```

Cultivation Service가 environment_setting을 생성/수정할 때 발행하는 EnvironmentRangeUpdatedEvent를
구독해 값을 갱신하며, 이때 TTL도 24시간으로 다시 연장합니다(write-through). 캐시가 없을 때만
Cultivation Service를 OpenFeign으로 호출해 값을 채워 넣습니다.

---

# Sensor

## Current Environment

실시간 대시보드 조회를 위한
최신 센서 데이터입니다.

Key

```
cultivation:{cultivationId}:current
```

Example

```
cultivation:3:current
```

Value

```json
{
  "temperature":22.5,
  "humidity":91.2,
  "co2":820,
  "light":430,
  "updatedAt":"2026-08-15T10:22:30"
}
```

TTL

```
없음
```

최신 데이터가 들어올 때마다(매초) 덮어씁니다. Cultivation Service는 EnvironmentMeasuredEvent를 받을 때마다
Redis는 항상 갱신하지만, InfluxDB는 재배별 10초 간격으로 스로틀링하여 저장합니다. Redis는 값 1건만
유지하는 덮어쓰기 구조라 매초 갱신해도 저장 용량에는 영향이 없습니다. (자세한 내용은
[influxdb.md](./influxdb.md), [cultivation.md](../01_Domain/cultivation.md) 참고)

---

# Redis 사용 목적

## Auth Service

저장 데이터

- Refresh Token
- 이메일 인증번호

---

## AI Service

저장 데이터

- AI 챗봇 응답 (ai:{hash}, TTL 24시간)
- AI 생육 분석 결과 (ai:{cultivationId}:analysis, TTL 6시간)
- 버섯 가이드 (ai:mushroom:{mushroomId}:guide, TTL 7일)
- 인사이트 후보 (ai:{cultivationId}:insight:candidates, TTL 24시간, 요청 시점에 채워짐)

일일 피드백(환경 통계 포함)은 Redis에 캐시하지 않습니다. 하루에 한 번만 생성되고
`daily_feedback` 테이블(PostgreSQL)에 바로 영구 저장되므로 별도 캐시가 필요하지
않습니다.

---

## Rule Engine Service

저장 데이터

- 목표 환경 범위 (temp/humidity/co2/light의 min~max)

---

## Cultivation Service

저장 데이터

- 최신 온도
- 최신 습도
- 최신 CO₂
- 최신 조도

---

# 데이터 흐름

## Refresh Token

```
Login

↓

JWT 발급

↓

Redis 저장
```

---

## Email Verification

```
인증번호 생성

↓

Redis 저장

↓

5분 후 자동 삭제
```

---

## AI Cache

```
질문

↓

Redis 조회

↓

있음

↓

바로 응답

----------------

없음

↓

LLM 호출

↓

Redis 저장
```

---

## Sensor Cache

```
MQTT 수신 (1초 주기)

↓

Rule Engine Service (검증 + 규칙평가)

↓

RabbitMQ (EnvironmentMeasuredEvent, 매초 발행)

↓

Cultivation Service

↓

Redis 저장 (매초, 항상) ── InfluxDB 저장 (10초 이상 경과했을 때만)

↓

Dashboard 조회
```

---

## 인사이트 후보 Cache

```
사용자 요청 ('/인사이트' 명령어)

↓

Redis 조회

↓

있음

↓

바로 응답

----------------

없음

↓

Cultivation Service OpenFeign 호출 (버섯 종류 + 환경 평균 + 내 cultivation_id 목록 조회)

↓

AI DB insight 테이블 검색 (mushroom_id 정확히 일치 + 온습도/CO2/조도 오차 범위 + 내 cultivation 제외, 최신순 SQL 필터)

↓

매칭 사례 있음 → Redis 저장 (LLM 미호출)
매칭 사례 없음 → 고정 문구 반환 (Redis 저장 안 함)

----------------

후보 선택 시 (캐시하지 않음)

↓

AI DB daily_feedback 테이블 조회 (선택한 cultivation_id, 날짜순, 수확일은 insight.summary로 대체)
```

---

## Rule Engine Range Cache

```
Cultivation Service

↓

environment_setting 생성/수정

↓

RabbitMQ (EnvironmentRangeUpdatedEvent)

↓

Rule Engine Service

↓

Redis 저장 (write-through)

----------------

캐시 미스 시

↓

Rule Engine Service

↓

Cultivation Service OpenFeign 호출 (fallback)

↓

Redis 저장
```

---

# 메모리 관리

TTL을 사용하는 데이터

- Refresh Token (14일)
- 이메일 인증 (5분)
- AI 챗봇 응답 캐시 (24시간)
- AI 생육 분석 결과 캐시 (6시간)
- 버섯 가이드 캐시 (7일)
- 인사이트 후보 캐시 (24시간, 요청 시점에 채워짐)
- 목표 환경 범위 캐시 (Rule Engine Service, 24시간)

TTL을 사용하지 않는 데이터

- 최신 센서 데이터

---

# 장애 대응

Redis 장애 발생 시

## Auth

- 로그인은 가능
- Refresh Token 재발급 불가

---

## AI

- Cache Miss 처리
- LLM 직접 호출
- 인사이트 후보는 원래도 요청마다 캐시 미스 시 재검색되는 흐름이라, Redis 장애 시에도 매번 insight 테이블 검색을 거쳐 응답은 가능합니다(속도만 저하). 후보 상세 조회는 원래 캐시하지 않으므로 영향이 없습니다.

---

## Rule Engine

- 캐시 미스 상태와 동일하게 동작 (매번 Cultivation Service OpenFeign 호출)
- Cultivation Service에 순간적으로 트래픽이 몰릴 수 있음
- 자동 제어 판단 자체는 계속 동작하지만 응답 지연이 늘어날 수 있음

---

## Sensor

- InfluxDB에서 최신 데이터 조회
- 성능은 다소 저하될 수 있음

---

# 추후 개발 예정

- Redis Cluster
- Pub/Sub
- Stream
- 분산 Lock