# AI API

## 개요

AI Service에서 제공하는 REST API 명세입니다.

Base URL

```
/api/v1/ai
```

인증 방식

```
Bearer JWT
```

내부 서비스(Cultivation Service 등) 호출은 OpenFeign을 통해 이루어집니다.

> ℹ️ **변경 이력**: 환경 추천 API(`POST /environment`)는 제거되었습니다. 버섯 종류가 공공데이터
> 기준 5가지로 고정되어 있어 Vector Search/LLM 없이도 항상 동일한 값이 나오므로, Cultivation
> Service가 자체 참조 테이블(`mushroom_reference`)을 직접 조회하는 방식으로 이전했습니다.
> 자세한 내용은 [cultivation-api.md](./cultivation-api.md)의 "재배 생성" 참고.

> ℹ️ **변경 이력**: `POST /mushroom-guide`가 새로 추가되었습니다. 재배 생성 직후 버섯의
> 효능/재배 주의사항을 자연어로 보여주는 기능으로, Client가 재배 생성 응답을 받은 뒤 이
> 엔드포인트를 직접 호출합니다(Cultivation Service를 거치지 않음). 환경 추천과 달리 이 기능은
> "설명 문서 생성"이 목적이라 LLM을 그대로 사용하되, `mushroomType`(5종 고정) 기준으로
> 캐싱해 동일 종류에 대한 반복 호출을 피합니다.

> ℹ️ **변경 이력**: `mushroom_reference`(Cultivation DB)에 특성/효능/재배 가이드/추가 정보
> 원문이 추가되면서, 캐시 미스 시 AI Service가 Cultivation Service를 OpenFeign으로 호출해
> (`GET /api/v1/mushroom-references/{mushroomType}`) 이 원문을 RAG 컨텍스트로 가져온 뒤
> LLM에 전달합니다. LLM은 원문을 그대로 반환하지 않고 자연스러운 문장으로 재구성합니다.

> ℹ️ **변경 이력**: `GET /chat/history`가 새로 추가되었습니다. `POST /chat`이 매 질의응답을
> `chat_message` 테이블(PostgreSQL)에 저장하며, 이 엔드포인트로 이전 대화를 조회할 수
> 있습니다. AI Service가 처음으로 PostgreSQL DB를 갖게 되었습니다. 기존 `ai:{hash}`
> Redis 캐시(동일 질문 재요청 시 LLM 재호출 방지용)는 역할이 겹치지 않아 그대로 유지됩니다.
> (자세한 내용은 [ai-db.md](../03_Database/ai-db.md) 참고)

> ℹ️ **변경 이력**: 월간 리포트를 폐기하고(재배 기간이 한 달을 넘지 않아 의미 없음), 리포트
> 생성 방식을 사용자 요청 기반(`POST /report`)에서 **Sensor Service의 Weekly Scheduler가
> 매주 먼저 생성해 두는 방식**으로 바꿨습니다. `POST /report`는 제거되었고, 이미 생성된
> 리포트를 읽기만 하는 `GET /report`로 대체되었습니다. 또한 "일일 피드백" 기능이 추가되어
> `GET /feedback/daily`로 조회할 수 있습니다. (자세한 내용은
> [ai-report.md](../04_sequence/ai-report.md), [daily-feedback.md](../04_sequence/daily-feedback.md) 참고)

> ℹ️ **변경 이력**: "인사이트" 기능이 추가되어 `GET /insight`로 조회할 수 있습니다. 일일
> 피드백/AI 리포트와 달리 스케줄러가 미리 만들어두지 않으며, 사용자가 호출한 시점에 Embedding
> Service 검색 + LLM 요약이 즉시 수행됩니다(on-demand). 배치 임베딩 적재는 별도의 Insight
> Batch Scheduler(00시)가 담당하며 사용자가 호출하는 API가 아닙니다. (자세한 내용은
> [insight.md](../04_sequence/insight.md) 참고)

---

# 생육 분석 (Vision)

## POST /analysis

사용자가 업로드한 사진을 Vision 모델로 분석합니다.

카메라 센서가 아닌, 사용자가 직접 촬영하여 업로드한 사진을 대상으로 합니다.

### Request

```json
{
    "cultivationId": 3,
    "mushroomType": "OYSTER",
    "imageUrl": "https://minio/mushroom-photos/3/20260815-090000.jpg"
}
```

---

### Process

AI Service

↓

MinIO에서 이미지 조회 (imageUrl)

↓

Vision 모델 분석

↓

균사 성장률 / 갓 크기 / 색상 / 병충해 판별

↓

생육 점수 계산 (균사 40% + 갓크기 30% + 색상 15% + 병충해 15%)

↓

LLM (결과 해석 및 개선 방안 생성)

---

### Response

```json
{
    "growthScore": 85,
    "myceliumGrowthRate": 82,
    "capSize": "중(3.2cm)",
    "colorStatus": "정상",
    "diseaseStatus": "정상",
    "growthStage": "자실체 형성기",
    "expectedHarvestDate": "2026-08-20",
    "improvement": "균사 성장은 양호하나 조도가 다소 낮게 유지되고 있어 갓 형성이 더딜 수 있습니다."
}
```

Vision 모델이 산출한 지표(growthScore, myceliumGrowthRate, capSize, colorStatus, diseaseStatus, growthStage, expectedHarvestDate)는
결정론적으로 계산되며, LLM은 improvement(해석/개선 방안)만 생성합니다.

---

# 버섯 가이드

## POST /mushroom-guide

재배 생성 직후, 선택한 버섯 종류의 효능/재배 시 주의사항을 자연어로 생성합니다. Client가
재배 생성 응답(`cultivationId`, `recommendedEnvironment`)을 받은 뒤 바로 호출하는 것을
가정하지만, `cultivationId`는 필요하지 않습니다(버섯 종류에만 의존).

### Request

```json
{
    "mushroomType": "OYSTER"
}
```

---

### Process

AI Service

↓

Redis 캐시 조회 (ai:mushroom:{mushroomType}:guide)

↓

Cache Miss 시 Cultivation Service OpenFeign 호출 (`GET /api/v1/mushroom-references/{mushroomType}`, RAG 컨텍스트 조회)

↓

LLM 호출 (characteristics/healthBenefits/cultivationGuide/additionalInfo를 참고해 효능/주의사항 생성) → Redis에 캐시 저장 (TTL 7일)

---

### Response

```json
{
    "mushroomType": "OYSTER",
    "benefits": "느타리버섯은 식이섬유와 베타글루칸이 풍부해 면역력 강화에 도움을 줍니다.",
    "precautions": "다습한 환경을 선호하지만 환기가 부족하면 곰팡이가 발생하기 쉬우니 CO₂ 농도 관리에 유의하세요."
}
```

`benefits`/`precautions`는 `mushroom_reference`의 원문(characteristics/healthBenefits/
cultivationGuide/additionalInfo)을 LLM이 참고해 재구성한 결과이며, 원문을 그대로 반환하지
않습니다.

버섯 종류가 5가지로 고정되어 있어 같은 `mushroomType`이면 항상 같은 응답이 캐시에서
반환됩니다. `mushroom_reference.description`(짧은 한 줄 참고 문구, 재배 생성 응답에 포함)과는
별개의 콘텐츠입니다.

---

# AI 챗봇

## POST /chat

### Request

```json
{
    "cultivationId": 3,
    "message": "왜 성장이 느린가요?"
}
```

---

### Process

AI Service

↓

Redis 캐시 조회 (ai:{hash})

↓

Cache Miss 시 Sensor Service 조회 + Embedding Service 유사 사례 검색 (선택)

↓

LLM

↓

chat_message 저장 (user_id, cultivationId, message, answer)

---

### Response

```json
{
    "answer": "현재 습도가 목표보다 5% 낮은 상태가 지속되고 있어 생육 속도가 느려졌을 수 있습니다. 가습기 가동 주기를 조금 더 짧게 설정하는 것을 권장합니다."
}
```

Redis 캐시 히트 여부와 무관하게, 매 요청은 `chat_message`에 새 행으로 저장됩니다(캐시된
응답이라도 "언제 다시 물어봤는지"는 별도 이력이므로).

---

# AI 챗봇 대화 이력 조회

## GET /chat/history

특정 재배에 대해 로그인한 사용자가 챗봇과 나눈 이전 질문/답변을 최신순으로 조회합니다.

### Query Parameter

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| cultivationId | long | O | 조회할 재배 ID |
| page | int | X | 페이지 번호 (기본값 0) |
| size | int | X | 페이지 크기 (기본값 20) |

---

### Process

AI Service

↓

`chat_message` 조회 (cultivationId, 요청자 소유 검증, 최신순)

---

### Response

```json
{
    "messages": [
        {
            "message": "왜 성장이 느린가요?",
            "answer": "현재 습도가 목표보다 5% 낮은 상태가 지속되고 있어 생육 속도가 느려졌을 수 있습니다.",
            "createdAt": "2026-08-15T13:30:00"
        }
    ]
}
```

---

# AI 리포트 조회

## GET /report

이미 생성된 가장 최근 주간 리포트를 조회합니다. 리포트 생성 자체는 이 API가 트리거하지
않으며, Sensor Service의 Weekly Scheduler가 매주 미리 만들어 둡니다(자세한 내용은
[ai-report.md](../04_sequence/ai-report.md) 참고). 재배 기간이 한 달을 넘지 않아 월간
리포트는 제공하지 않습니다.

### Query Parameter

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| cultivationId | long | O | 조회할 재배 ID |

---

### Process

AI Service

↓

Redis 조회 (report:{cultivationId}:weekly)

↓

없으면(아직 첫 주간 리포트가 생성되지 않음) AI011 반환

---

### Response

```json
{
    "period": "weekly",
    "averageTemperature": 22.1,
    "averageHumidity": 90.3,
    "environmentMaintainRate": 95,
    "autoControlCount": 6,
    "report": "지난 7일 동안 평균 온도는 22.1℃로 적정 범위를 유지했습니다. 전체적으로 매우 안정적인 재배 환경을 유지하고 있습니다."
}
```

---

# 일일 피드백 조회

## GET /feedback/daily

특정 재배에 대해 지금까지 생성된 일일 피드백을 최신순으로 조회합니다. 생성 자체는 이 API가
트리거하지 않으며, Daily Scheduler가 매일 자동으로 생성합니다(자세한 내용은
[daily-feedback.md](../04_sequence/daily-feedback.md) 참고).

### Query Parameter

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| cultivationId | long | O | 조회할 재배 ID |
| page | int | X | 페이지 번호 (기본값 0) |
| size | int | X | 페이지 크기 (기본값 20) |

---

### Process

AI Service

↓

`daily_feedback` 조회 (cultivationId, 요청자 소유 검증, 최신순)

---

### Response

```json
{
    "feedbacks": [
        {
            "feedbackDate": "2026-08-15",
            "hasGrowthData": true,
            "content": "온도를 22℃에서 24℃로 높인 이후 균사 성장률이 평균 6%p 개선되는 추세입니다.",
            "createdAt": "2026-08-15T23:00:00"
        },
        {
            "feedbackDate": "2026-08-14",
            "hasGrowthData": false,
            "content": "전날 사진이 없어 피드백을 남길 수 없습니다.",
            "createdAt": "2026-08-14T23:00:00"
        }
    ]
}
```

---

# 인사이트 조회

## GET /insight

같은 버섯 종류이면서 유사한(오차 범위 내) 온도로 재배했던 타인의 완료된 재배 사례를 바탕으로
현재 재배 상태에 대한 피드백을 생성합니다. 일일 피드백(자기 자신의 이력 비교)과 달리, 이
API는 사용자가 호출한 시점에 즉시 검색/요약이 수행됩니다(사용자 요청 시점, on-demand).

### Query Parameter

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| cultivationId | long | O | 조회할 재배 ID |

---

### Process

AI Service

↓

Redis 캐시 조회 (ai:{cultivationId}:insight)

↓

Cache Miss 시 Cultivation Service OpenFeign 호출 (`GET /cultivations/{cultivationId}`, 버섯
종류 조회) + Sensor Service OpenFeign 호출 (`GET
/api/v1/sensors/cultivations/{cultivationId}/environment-average`, 현재 재배의 환경 평균 조회)

↓

Embedding Service OpenFeign 호출 (버섯 종류 + 온도 오차 범위로 `cultivation_insight` 인덱스 검색)

↓

매칭 사례 있음 → LLM 요약 → Redis 캐시 저장 (TTL 24시간)
매칭 사례 없음 → LLM 미호출, 고정 안내 문구 (캐시하지 않음)

---

### Response

```json
{
    "cultivationId": 27,
    "matchedCaseCount": 8,
    "insight": "비슷한 온도(21.5~22.5℃)로 느타리버섯을 재배한 다른 사례 8건과 비교했을 때, 현재 재배의 생육 점수는 평균보다 다소 높은 편입니다. 이 온도대에서 습도를 90% 전후로 유지한 사례들의 수확량이 특히 높았습니다."
}
```

유사 사례가 없을 때의 응답:

```json
{
    "cultivationId": 27,
    "matchedCaseCount": 0,
    "insight": "아직 비교할 수 있는 유사 사례가 충분하지 않습니다."
}
```

---

# Error Code

| Code | Description |
|------|-------------|
| AI001 | Embedding 검색 실패 (챗봇 유사 사례 검색) |
| AI002 | LLM 응답 실패 |
| AI003 | Redis Cache 조회 실패 |
| AI004 | Sensor 데이터 부족 |
| AI005 | 등록된 사진 없음 |
| AI006 | Vision 모델 분석 실패 |
| AI007 | API 호출 시간 초과 |
| AI008 | 버섯 가이드 생성 실패 (LLM 응답 실패, Cultivation Service RAG 컨텍스트 조회 실패 포함) |
| AI009 | chat_message 저장 실패 (PostgreSQL) |
| AI010 | 다른 사용자의 재배에 대한 대화 이력 조회 시도 |
| AI011 | 아직 생성된 주간 리포트 없음 |
| AI012 | growth_record 저장 실패 (PostgreSQL) |
| AI013 | daily_feedback 저장 실패 (PostgreSQL) |
| AI014 | 일일 피드백 생성 시 Sensor Service 호출 실패 (environment_setting 이력 조회 실패) |
| AI015 | 인사이트 조회 시 Cultivation Service 호출 실패 (버섯 종류 조회 실패) 또는 Sensor Service 호출 실패 (환경 평균 조회 실패) |
| AI016 | 인사이트 조회 시 Embedding Service 검색 실패 |
| AI017 | 인사이트 배치 임베딩 적재 실패 (Cultivation Service 미임베딩 조회/완료 처리, Sensor Service 환경 평균 일괄 조회, Embedding Service 저장 요청 실패 포함) |

---

# OpenFeign

호출하는 서비스

```
Embedding Service (챗봇 유사 재배 사례 검색, 선택적 호출)
Sensor Service (센서 데이터 조회, 챗봇에서 참고용으로 사용. 주간 통계는 더 이상 요청 시점에 조회하지 않음)
Cultivation Service (버섯 가이드 RAG 컨텍스트 조회, GET /api/v1/mushroom-references/{mushroomType}, 캐시 미스 시에만)
Sensor Service (일일 피드백 생성 시 environment_setting 최근 변경 이력 조회, GET /api/v1/sensors/cultivations/{cultivationId}/environment-history, Daily Scheduler 실행 시)
Cultivation Service (인사이트 배치 임베딩 적재 시, GET /api/v1/harvests/unembedded-count, GET /api/v1/harvests/unembedded, PATCH /api/v1/harvests/embedded — Insight Batch Scheduler 실행 시)
Sensor Service (인사이트 배치 임베딩 적재 시 환경 평균 일괄 조회, POST /api/v1/sensors/environment-averages — Insight Batch Scheduler 실행 시)
Cultivation Service (인사이트 조회 시 버섯 종류 조회, GET /cultivations/{cultivationId} — 사용자 요청 시점, 캐시 미스 시에만)
Sensor Service (인사이트 조회 시 환경 평균 조회, GET /api/v1/sensors/cultivations/{cultivationId}/environment-average — 사용자 요청 시점, 캐시 미스 시에만)
Embedding Service (인사이트 배치 임베딩 요청 — Insight Batch Scheduler 실행 시)
Embedding Service (인사이트 유사 사례 검색 — 사용자 요청 시점, 캐시 미스 시에만)
```

> ℹ️ **변경 이력**: `environment_setting` 관련 호출(일일 피드백 이력 조회, 인사이트 환경
> 평균 조회)이 Cultivation Service에서 Sensor Service로 바뀌었습니다. 인사이트 배치
> 임베딩 적재 시 환경 평균 일괄 조회가 Sensor Service 호출로 새로 추가되었습니다.

호출받는 서비스

```
Cultivation Service (생육 사진 Vision 분석 요청)
Sensor Service (Weekly Scheduler가 집계한 주간 통계 전달, push)
API Gateway (AI 챗봇 요청, 버섯 가이드 요청, AI 리포트 조회, 일일 피드백 조회, 인사이트 조회)
```

---

# Redis

캐시 대상

- AI 생육 분석 결과 (ai:{cultivationId}:analysis, TTL 6시간) — `growth_record`에도 영구 저장됨
- AI 챗봇 응답 (ai:{hash}, TTL 24시간)
- AI 리포트 (report:{cultivationId}:weekly, TTL 24시간) — Weekly Scheduler가 push로 미리 채워둠
- 버섯 가이드 (ai:mushroom:{mushroomType}:guide, TTL 7일) — cultivationId가 아닌 mushroomType 기준으로 캐싱됩니다.
- 인사이트 (ai:{cultivationId}:insight, TTL 24시간) — 사용자 요청 시점에 생성되어 캐시됩니다(유일하게 push가 아닌 pull로 채워지는 캐시). 유사 사례가 없어 고정 문구로 응답한 경우는 캐시하지 않습니다.

일일 피드백은 Redis에 캐시하지 않고 `daily_feedback` 테이블에 바로 영구 저장합니다(하루에
한 번만 생성되므로 캐시가 불필요).

---

# MinIO

사용자가 업로드한 사진을 읽기 전용으로 조회합니다. (Cultivation Service가 저장)
