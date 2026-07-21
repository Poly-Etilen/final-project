# 인사이트 시퀀스

## 개요

"일일 피드백"이 **내 재배**의 어제와 오늘을 비교하는 기능이라면, 인사이트는 같은 버섯
종류이면서 유사한(오차 범위 내) 온도로 재배했던 **타인의 완료된 재배 사례**를 바탕으로 현재
내 재배 상태를 피드백해주는 기능입니다.

이 기능은 서로 독립적인 두 흐름으로 구성됩니다.

1. **배치 임베딩 적재** (스케줄 기반, push가 아닌 내부 적재 작업): AI Service의 Insight
   Batch Scheduler가 매일 00시에 실행되어, 아직 임베딩되지 않은 수확(harvest)이 전체 합산
   20건 이상 쌓였는지 확인합니다. 쌓였다면 그 배치를 Embedding Service로 보내 임베딩한 뒤
   Elasticsearch(`cultivation_insight` 인덱스)에 저장합니다.
2. **인사이트 조회** (사용자 요청 시점, on-demand): 배치 적재와 완전히 별개로, 사용자가
   특정 재배에 대해 인사이트를 요청한 시점에만 검색/요약이 이루어집니다. 일일 피드백/AI
   리포트처럼 미리 생성해두지 않습니다.

두 흐름이 섞이지 않도록 유의해야 합니다 — 배치 임베딩이 "데이터를 쌓는" 흐름이라면, 조회는
이미 쌓인 데이터에서 "찾아서 보여주는" 흐름입니다.

> ℹ️ **참고**: "임계치에 도달한 만큼만 임베딩하고 카운터가 다시 쌓이길 기다리는" 배치 방식이라,
> 하루에 여러 번 임계치를 넘을 수도, 며칠간 한 번도 안 넘을 수도 있습니다. "매일 전체를 다시
> 훑어 재임베딩"하는 방식이 아닙니다.

> ℹ️ **변경 이력**: 팀 회의 결과 `environment_setting` 테이블이 Cultivation Service에서
> Sensor Service로 완전히 이관되었습니다. 환경 평균 계산이 더 이상 Cultivation Service의
> 책임이 아니게 되면서, 배치 임베딩 적재의 환경 평균 조회(③번, 신규 배치 조회 API)와 인사이트
> 조회의 환경 평균 조회(⑧번)가 모두 Sensor Service 호출로 바뀌었습니다. (자세한 내용은
> [sensor.md](../01_Domain/sensor.md), [sensor-db.md](../03_Database/sensor-db.md),
> [README.md](../README.md)의 결정 사항 #24 참고)

---

# Sequence

## ① 배치 임베딩 적재

```text
Insight Batch Scheduler (AI Service, 매일 00시)

↓

Cultivation Service OpenFeign 호출 (GET /api/v1/harvests/unembedded-count)

├── 20건 미만 → 종료 (다음 날 다시 확인)
└── 20건 이상 ↓

Cultivation Service OpenFeign 호출 (GET /api/v1/harvests/unembedded, 미임베딩 목록)

↓

Sensor Service OpenFeign 호출 (POST /api/v1/sensors/environment-averages, cultivationId 목록으로 환경 평균 일괄 조회)

↓

AI Service 자체 growth_record 조회 (건별 생육 점수 병합)

↓

Embedding Service OpenFeign 호출 (POST /api/v1/embeddings/insights, 배치 임베딩 요청)

↓

Embedding Service: 자연어 요약 생성 → Embedding Model → Elasticsearch 저장 (cultivation_insight)

↓

Cultivation Service OpenFeign 호출 (PATCH /api/v1/harvests/embedded, 완료 처리)
```

---

## ② 인사이트 조회

```text
Client (사용자가 특정 재배의 인사이트 요청)

↓

API Gateway

↓

AI Service

↓

Redis 캐시 조회 (ai:{cultivationId}:insight)

├── Cache Hit → 즉시 반환
└── Cache Miss ↓

Cultivation Service OpenFeign 호출 (GET /cultivations/{cultivationId}, 버섯 종류 조회)
+ Sensor Service OpenFeign 호출 (GET /api/v1/sensors/cultivations/{cultivationId}/environment-average, 환경 평균 조회)

↓

Embedding Service OpenFeign 호출 (POST /api/v1/embeddings/insights/search, mushroomType + 온도 오차 범위)

├── 매칭 사례 있음 → LLM 요약 → Redis 캐시 저장 (TTL 24시간) → 반환
└── 매칭 사례 없음 → LLM 미호출, 고정 안내 문구 반환 (캐시하지 않음)

↓

Client
```

---

# 상세 과정

## 1. Insight Batch Scheduler 실행

AI Service의 Insight Batch Scheduler가 매일 00시에 실행됩니다. Daily Scheduler(일일 피드백,
23시)와는 별개의 스케줄러이며, 실행 시각도 다릅니다.

---

## 2. 미임베딩 건수 확인

```
GET /api/v1/harvests/unembedded-count
```

```json
{
    "count": 23
}
```

전체 사용자를 합산한 전역 카운트입니다. 20건 미만이면 이번 실행은 여기서 종료합니다.

---

## 3. 미임베딩 목록 조회 + 환경 평균 일괄 조회

20건 이상일 때만 실행합니다.

```
GET /api/v1/harvests/unembedded
```

```json
[
    {
        "cultivationId": 12,
        "mushroomType": "OYSTER",
        "harvestWeight": 3200
    }
]
```

이 목록의 `cultivationId`들을 모아 Sensor Service에 환경 평균을 일괄 조회합니다.

```
POST /api/v1/sensors/environment-averages
```

```json
{
    "cultivationIds": [12, 15, 18]
}
```

```json
[
    { "cultivationId": 12, "avgTemperature": 21.8, "avgHumidity": 89.2, "avgCo2": 780.5, "avgLight": 360.0 }
]
```

`avgTemperature`/`avgHumidity`/`avgCo2`/`avgLight`는 Sensor Service가 `environment_setting`
이력 전체를 기간 가중 평균(각 설정값이 적용되었던 기간의 길이로 가중)하여 계산한 값입니다.
AI Service는 두 응답을 `cultivationId` 기준으로 합칩니다.

> ℹ️ **변경 이력**: 이 환경 평균은 원래 `GET /api/v1/harvests/unembedded` 응답에 Cultivation
> Service가 미리 계산해서 포함해주던 필드였습니다. `environment_setting`이 Sensor Service로
> 이관되면서 더 이상 Cultivation Service가 계산할 수 없어, 별도 배치 조회 API 호출로
> 분리했습니다.

---

## 4. growth_record 병합

목록의 각 `cultivationId`에 대해 AI Service 자체 DB의 `growth_record`에서 마지막(가장 최근)
생육 분석 결과의 `growthScore`를 조회해 병합합니다.

```sql
SELECT growth_score FROM growth_record
WHERE cultivation_id = ?
ORDER BY analyzed_at DESC
LIMIT 1;
```

---

## 5. Embedding Service로 배치 임베딩 요청

```json
{
    "cases": [
        {
            "cultivationId": 12,
            "mushroomType": "OYSTER",
            "avgTemperature": 21.8,
            "avgHumidity": 89.2,
            "avgCo2": 780.5,
            "avgLight": 360.0,
            "growthScore": 88,
            "harvestWeight": 3200
        }
    ]
}
```

Embedding Service는 각 case를 자연어 요약 문장으로 변환한 뒤 임베딩하고, `cultivation_insight`
인덱스에 저장합니다(자세한 내용은 [embedding.md](../01_Domain/embedding.md),
[elasticSearch.md](../03_Database/elasticSearch.md) 참고).

---

## 6. 임베딩 완료 처리

Elasticsearch 저장에 성공한 건에 한해 Cultivation Service에 완료 처리를 요청합니다.

```json
{
    "cultivationIds": [12, 15, 18]
}
```

저장에 실패한 건은 완료 처리를 요청하지 않으며, `is_embedded`가 계속 `FALSE`로 남아 다음
배치(카운터가 다시 20건에 도달했을 때) 때 재시도됩니다.

---

## 7. 인사이트 조회 요청

배치 적재와 완전히 별개로, 사용자가 원할 때 호출합니다.

```http
GET /api/v1/ai/insight?cultivationId=27
```

---

## 8. 현재 재배 버섯 종류 + 환경 평균 조회 (Redis Cache Miss 시)

버섯 종류는 Cultivation Service에서 조회합니다.

```
GET /cultivations/{cultivationId}
```

```json
{
    "cultivationId": 27,
    "name": "느타리 1호기",
    "mushroomType": "OYSTER",
    "status": "RUNNING",
    "createdAt": "2026-08-01"
}
```

환경 평균은 Sensor Service에서 조회합니다.

```
GET /api/v1/sensors/cultivations/{cultivationId}/environment-average
```

```json
{
    "cultivationId": 27,
    "avgTemperature": 22.1,
    "avgHumidity": 90.5,
    "avgCo2": 810.0,
    "avgLight": 370.0
}
```

수확 목록 조회(위 3번)와 달리, 재배가 아직 `RUNNING` 상태여도(harvest가 없어도) 지금까지의
평균으로 조회할 수 있습니다.

> ℹ️ **변경 이력**: 이 두 값은 원래 Cultivation Service의 단일 엔드포인트(`GET
> /api/v1/cultivations/{cultivationId}/environment-average`)가 함께 반환했습니다.
> `environment_setting`이 Sensor Service로 이관되면서 두 서비스를 각각 호출하는 방식으로
> 나뉘었습니다.

---

## 9. 유사 사례 검색

```json
{
    "mushroomType": "OYSTER",
    "avgTemperature": 22.1,
    "tolerance": 1.5,
    "topK": 10
}
```

Embedding Service는 `mushroomType`(정확히 일치) + `avgTemperature`(오차 범위 내)로
Elasticsearch를 필터링해 매칭된 사례를 반환합니다. 코사인 유사도 기반 Vector Search가
아니라 term/range 필터 기반입니다(자세한 내용은 [elasticSearch.md](../03_Database/elasticSearch.md)
참고).

```json
{
    "matchedCaseCount": 8,
    "cases": [
        { "avgTemperature": 21.8, "avgHumidity": 89.2, "avgCo2": 780.5, "avgLight": 360.0, "growthScore": 88, "harvestWeight": 3200 }
    ]
}
```

---

## 10-A. 매칭 사례가 있는 경우

AI Service

↓

LLM

입력

- 현재 재배의 환경 평균 + 버섯 종류
- 매칭된 유사 사례들의 환경 평균 / 생육 점수 / 수확량

출력 예시

```json
{
    "matchedCaseCount": 8,
    "insight": "비슷한 온도(21.5~22.5℃)로 느타리버섯을 재배한 다른 사례 8건과 비교했을 때, 현재 재배의 생육 점수는 평균보다 다소 높은 편입니다. 이 온도대에서 습도를 90% 전후로 유지한 사례들의 수확량이 특히 높았습니다."
}
```

결과를 Redis에 캐시합니다(`ai:{cultivationId}:insight`, TTL 24시간).

---

## 10-B. 매칭 사례가 없는 경우

LLM을 호출하지 않고 고정 문구를 사용합니다. 이 응답은 캐시하지 않습니다 — 다음 요청 시점에는
그 사이 배치 임베딩이 진행되어 매칭될 수도 있기 때문입니다.

```json
{
    "matchedCaseCount": 0,
    "insight": "아직 비교할 수 있는 유사 사례가 충분하지 않습니다."
}
```

---

# 사용 Database

## PostgreSQL

```
harvest.is_embedded (Cultivation DB) — 임베딩 여부 플래그, AI Service는 조회/갱신만
cultivation.mushroom_type (Cultivation DB, 조회 전용) — 버섯 종류
environment_setting (Sensor DB, 조회 전용) — 환경 평균 계산의 원본
growth_record (AI DB, 조회 전용) — 생육 점수 병합용
```

## Elasticsearch

```
cultivation_insight — 완료된 재배 사례의 임베딩 저장/검색 대상
```

## Redis

```
ai:{cultivationId}:insight (TTL 24시간) — 인사이트 조회 결과 캐시. 매칭 사례가 없으면 캐시하지 않음.
```

---

# OpenFeign

```
AI Service → Cultivation Service
  GET /api/v1/harvests/unembedded-count
  GET /api/v1/harvests/unembedded
  PATCH /api/v1/harvests/embedded
  GET /cultivations/{cultivationId}

AI Service → Sensor Service
  POST /api/v1/sensors/environment-averages
  GET /api/v1/sensors/cultivations/{cultivationId}/environment-average

AI Service → Embedding Service
  POST /api/v1/embeddings/insights
  POST /api/v1/embeddings/insights/search
```

Embedding Service는 Elasticsearch를 직접 다루는 유일한 서비스입니다. AI Service, Cultivation
Service, Sensor Service 모두 Elasticsearch를 직접 호출하지 않습니다(기존 버섯 가이드 RAG,
챗봇 유사 사례 검색과 동일한 원칙).

> ℹ️ **변경 이력**: `environment_setting`이 Sensor Service로 이관되면서 환경 평균 조회가
> Cultivation Service 대신 Sensor Service 호출로 바뀌었습니다. 배치 임베딩 적재는 이제
> Cultivation Service(미임베딩 목록)와 Sensor Service(환경 평균 일괄 조회)를 각각
> 호출합니다.

---

# RabbitMQ

이 기능은 RabbitMQ 이벤트를 발행/구독하지 않습니다. 배치 임베딩 적재는 스케줄러가 직접
OpenFeign으로 오케스트레이션하며, 조회는 사용자 요청에 대한 동기 응답이라 별도 이벤트가
필요하지 않습니다. (일일 피드백의 `DailyFeedbackCompletedEvent`, AI 리포트의
`WeeklyReportCompletedEvent`와 다른 점입니다 — 두 기능은 알림 발송을 위해 이벤트를
발행하지만, 인사이트는 사용자가 직접 요청해서 즉시 받는 기능이라 별도 알림이 없습니다.)

---

# 예외 상황

- Insight Batch Scheduler 실행 실패 (해당 날짜는 배치가 진행되지 않으며, 미임베딩 카운트는 계속 누적되어 있으므로 다음 날 다시 시도됨)
- 배치 임베딩 적재 시 Cultivation Service 호출 실패 (미임베딩 건수/목록 조회 실패) — 이번 배치를 건너뜀
- 배치 임베딩 적재 시 Sensor Service 호출 실패 (환경 평균 일괄 조회 실패) — 이번 배치를 건너뜀
- 배치 임베딩 적재 시 Embedding Service 저장 실패 — 해당 건들은 완료 처리되지 않고 `is_embedded = FALSE`로 남아 다음 배치 때 재시도됨
- 인사이트 조회 시 Cultivation Service 호출 실패 (버섯 종류 조회 실패) 또는 Sensor Service 호출 실패 (현재 재배 환경 평균 조회 실패)
- 인사이트 조회 시 Embedding Service 호출 실패
- 인사이트 조회 시 매칭되는 유사 사례 없음 — LLM 미호출, 고정 안내 문구 반환 (오류가 아닌 정상 응답)
- LLM 응답 실패

---

# 고려 사항

- 배치 임베딩 적재는 사용자 요청이 아니라 Insight Batch Scheduler가 트리거합니다. 인사이트 조회만 사용자가 직접 요청합니다.
- "임베딩 여부"의 소유권은 Cultivation Service(`harvest.is_embedded`)에 있습니다. AI Service는 별도의 워터마크(마지막 처리 시각/ID)를 관리하지 않고, 매번 단순 조회로 판단합니다.
- 임계치(20건)는 전체 사용자를 합산한 전역 카운트이며, 사용자별/버섯 종류별로 나누어 세지 않습니다.
- 배치는 "임계치에 도달한 만큼만" 처리하는 방식입니다. 하루에 여러 번 임계치를 넘을 수도 있고, 며칠간 한 번도 안 넘을 수도 있습니다.
- 환경 평균은 "기간 가중 평균"입니다. 예를 들어 재배 기간 중 온도를 20℃로 20일, 22℃로 10일 유지했다면 단순 산술 평균(21℃)이 아니라 기간 길이로 가중한 평균을 사용합니다. Sensor Service가 `environment_setting` 이력을 조회해 계산하며, 별도 컬럼에 저장하지 않고 조회 시점마다 계산합니다.
- 버섯 종류(mushroomType)와 환경 평균은 서로 다른 서비스(Cultivation Service, Sensor Service)가 소유합니다. AI Service가 두 값을 각각 조회해 조합합니다.
- 인사이트 조회는 진행 중인 재배(RUNNING)에 대해서도 가능합니다 — "지금까지의" 평균으로 비교합니다. 반면 배치 임베딩 대상은 완료된 재배(harvest가 있는 건)로 한정됩니다.
- 인사이트는 일일 피드백과 달리 Redis에만 캐시하고 PostgreSQL에 영구 저장하지 않습니다. 매번 최신 Elasticsearch 데이터를 기반으로 다시 계산하는 것이 의미 있고(더 많은 유사 사례가 쌓일수록 결과가 달라질 수 있음), 하루에 한 번만 생성되는 일일 피드백과 달리 재조회 빈도가 예측되지 않기 때문입니다.
- 매칭 사례가 없는 응답은 캐시하지 않습니다. 배치 임베딩이 계속 진행되므로 다음 조회 시점에는 매칭될 가능성이 있기 때문입니다.
- Embedding Service는 검색 결과를 반환할 뿐 자연어 요약(LLM 호출)은 하지 않습니다. LLM 요약은 AI Service의 책임입니다.
