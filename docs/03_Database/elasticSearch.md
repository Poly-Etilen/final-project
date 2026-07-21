# Elasticsearch

## 개요

Elasticsearch는 버섯 재배 데이터를 Embedding(Vector) 형태로 저장하고
유사도(Vector Similarity Search)를 수행하기 위해 사용합니다.

Embedding Service는 버섯 재배 데이터를 임베딩하여 Elasticsearch에 저장하고,
AI Service는 AI 챗봇의 유사 재배 사례 검색(선택적 호출) 시 Vector Search를 수행합니다.

> ℹ️ **변경 이력**: 재배 생성 시의 "환경 추천"은 더 이상 이 Vector Search를 사용하지 않습니다.
> 버섯 종류가 공공데이터 기준 5가지로 고정되어 있어, Cultivation Service가 자체 참조 테이블
> (`mushroom_reference`, PostgreSQL)을 직접 조회하는 방식으로 대체했습니다. 이 Elasticsearch
> 인덱스는 AI 챗봇의 유사 사례 검색에만 사용됩니다.

> ℹ️ **변경 이력**: 원본 데이터 소스가 CSV/DatasourceGenerator에서 Cultivation DB의
> `mushroom_reference`로 바뀌었습니다. `mushroom_reference`에 이름/특성/효능/재배 가이드/추가
> 정보 컬럼이 추가되면서, 이 문서들도 같은 필드를 반영하도록 확장했습니다. Cultivation
> Service가 `MushroomReferenceUpdatedEvent`를 발행하면 Embedding Service가 구독해 이
> 인덱스를 갱신합니다(이벤트 기반, 버섯 종류가 5종 고정이라 매우 드물게 발생). 임베딩 벡터는
> Cultivation DB(PostgreSQL)에는 저장하지 않고 이 인덱스에만 저장해 데이터 중복을 피합니다.

> ℹ️ **변경 이력**: "인사이트" 기능(같은 버섯 종류 + 유사한 온도로 재배했던 타인의 사례 기반
> 피드백) 추가를 위해 `cultivation_insight` 인덱스가 새로 생겼습니다. `mushroom_environment`가
> "5종 고정 참조 데이터"를 다루는 것과 달리, `cultivation_insight`는 완료된 재배가 하나씩
> 끝날 때마다(정확히는 배치 임계치 20건 단위로) 계속 쌓이는 데이터입니다. 원본 데이터도
> Cultivation DB가 아닌 AI Service(자체 `growth_record` + Cultivation Service의
> `environment_setting`/`harvest` 조합)에서 옵니다. 검색 방식도 Cosine Similarity 기반 Vector
> Search가 아니라, 버섯 종류(정확히 일치) + 온도(오차 범위) term/range 필터입니다. (자세한
> 내용은 아래 "cultivation_insight" 관련 섹션들, [insight.md](../04_sequence/insight.md) 참고)

---

# 사용 서비스

| Service | 역할 |
|----------|------|
| Embedding Service | 임베딩 생성 및 저장 |
| AI Service | Vector Search 요청 |

---

# Index

```
mushroom_environment
cultivation_insight
```

---

# Document 구조

## mushroom_environment

문서 ID는 별도 필드 없이 `mushroomType` 값을 그대로 사용합니다(버섯 종류당 문서 1개).

| Field | Type | Description |
|--------|------|-------------|
| mushroomType | Keyword | 버섯 종류 (문서 ID로도 사용) |
| mushroomNameKo | Keyword | 버섯 한글 이름 |
| mushroomNameEn | Keyword | 버섯 영문 이름 |
| mushroomScientificName | Keyword | 학명 |
| tempMin / tempMax | Float | 최적 온도 범위 |
| humidityMin / humidityMax | Float | 최적 습도 범위 |
| co2Min / co2Max | Integer | 최적 CO₂ 범위 |
| lightMin / lightMax | Integer | 최적 조도 범위 |
| description | Text | 짧은 참고 문구 |
| characteristics | Text | 버섯 특성 설명 |
| healthBenefits | Text | 효능 |
| cultivationGuide | Text | 재배 시 주의사항/가이드 |
| additionalInfo | Text | 기타 추가 정보 |
| embedding | Dense Vector | characteristics/healthBenefits/cultivationGuide/additionalInfo를 결합해 생성한 임베딩 벡터 |

이 필드들은 Cultivation DB의 `mushroom_reference`와 1:1로 대응합니다. Cultivation DB에는
`embedding`을 저장하지 않으므로, 이 컬럼만 Elasticsearch에만 존재합니다.

---

## cultivation_insight

문서 ID는 별도 필드 없이 `cultivationId` 값을 그대로 사용합니다(완료된 재배 1건당 문서 1개).

| Field | Type | Description |
|--------|------|-------------|
| cultivationId | Keyword | 재배 ID (문서 ID로도 사용) |
| mushroomType | Keyword | 버섯 종류 (검색 필터용) |
| avgTemperature | Float | 기간 가중 평균 온도 (검색 필터용) |
| avgHumidity | Float | 기간 가중 평균 습도 |
| avgCo2 | Float | 기간 가중 평균 CO₂ |
| avgLight | Float | 기간 가중 평균 조도 |
| growthScore | Integer | 생육 점수 (AI Service `growth_record` 기준 마지막 분석 결과) |
| harvestWeight | Float | 수확량 |
| summary | Text | 자연어 요약 원문 (임베딩 대상) |
| embedding | Dense Vector | summary를 임베딩해 생성한 벡터 |
| createdAt | Date | 임베딩 저장 시각 |

`avgTemperature`/`mushroomType`은 검색 시 term/range 필터로 사용되며, `embedding`은 현재
필터링된 결과의 정렬/재랭킹 용도로만 예약되어 있고 인사이트 검색 자체는 필터만으로 동작합니다
(아래 "검색 방식" 참고).

---

# Mapping 예시

## mushroom_environment

```json
{
  "mappings": {
    "properties": {
      "mushroomType": { "type": "keyword" },
      "mushroomNameKo": { "type": "keyword" },
      "mushroomNameEn": { "type": "keyword" },
      "mushroomScientificName": { "type": "keyword" },
      "tempMin": { "type": "float" },
      "tempMax": { "type": "float" },
      "humidityMin": { "type": "float" },
      "humidityMax": { "type": "float" },
      "co2Min": { "type": "integer" },
      "co2Max": { "type": "integer" },
      "lightMin": { "type": "integer" },
      "lightMax": { "type": "integer" },
      "description": { "type": "text" },
      "characteristics": { "type": "text" },
      "healthBenefits": { "type": "text" },
      "cultivationGuide": { "type": "text" },
      "additionalInfo": { "type": "text" },
      "embedding": {
        "type": "dense_vector",
        "dims": 1024,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}
```

---

## cultivation_insight

```json
{
  "mappings": {
    "properties": {
      "cultivationId": { "type": "keyword" },
      "mushroomType": { "type": "keyword" },
      "avgTemperature": { "type": "float" },
      "avgHumidity": { "type": "float" },
      "avgCo2": { "type": "float" },
      "avgLight": { "type": "float" },
      "growthScore": { "type": "integer" },
      "harvestWeight": { "type": "float" },
      "summary": { "type": "text" },
      "embedding": {
        "type": "dense_vector",
        "dims": 1024,
        "index": true,
        "similarity": "cosine"
      },
      "createdAt": { "type": "date" }
    }
  }
}
```

---

# 저장 예시

## mushroom_environment

```json
{
  "mushroomType": "OYSTER",
  "mushroomNameKo": "느타리버섯",
  "mushroomNameEn": "Oyster Mushroom",
  "mushroomScientificName": "Pleurotus ostreatus",
  "tempMin": 15.0,
  "tempMax": 18.0,
  "humidityMin": 85,
  "humidityMax": 95,
  "co2Min": 700,
  "co2Max": 900,
  "lightMin": 300,
  "lightMax": 400,
  "description": "느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다.",
  "characteristics": "군생하며 갓은 회갈색~담회색을 띠고, 균사 성장 속도가 빠른 편입니다.",
  "healthBenefits": "식이섬유와 베타글루칸이 풍부해 면역력 강화와 콜레스테롤 감소에 도움을 줍니다.",
  "cultivationGuide": "다습한 환경을 선호하지만 환기가 부족하면 곰팡이가 발생하기 쉬우니 CO₂ 농도 관리에 유의해야 합니다.",
  "additionalInfo": null,
  "embedding": [
    0.124,
    -0.215,
    ...
  ]
}
```

---

## cultivation_insight

```json
{
  "cultivationId": "12",
  "mushroomType": "OYSTER",
  "avgTemperature": 21.8,
  "avgHumidity": 89.2,
  "avgCo2": 780.5,
  "avgLight": 360.0,
  "growthScore": 88,
  "harvestWeight": 3200,
  "summary": "느타리버섯, 평균 온도 21.8℃·습도 89% 환경에서 생육 점수 88점으로 3.2kg 수확",
  "embedding": [
    0.087,
    -0.192,
    ...
  ],
  "createdAt": "2026-08-16T00:00:00"
}
```

---

# 데이터 생성 흐름

## mushroom_environment

Cultivation Service

↓

관리자가 mushroom_reference 등록/수정 (characteristics/healthBenefits/cultivationGuide/additionalInfo 포함)

↓

RabbitMQ Publish (MushroomReferenceUpdatedEvent)

↓

Embedding Service 구독

↓

텍스트 결합 (characteristics + healthBenefits + cultivationGuide + additionalInfo)

↓

Embedding Model 호출 → Vector 생성

↓

Elasticsearch Upsert (mushroom_environment, mushroomType 기준)

버섯 종류가 5종으로 고정된 정적 데이터라, 이 흐름은 관리자가 참조 데이터를 등록/수정할 때만
드물게 발생합니다.

---

## cultivation_insight

AI Service (Insight Batch Scheduler, 매일 00시)

↓

Cultivation Service에서 미임베딩 건수 조회 → 20건 이상이면 목록 + 환경 평균 조회

↓

AI Service 자체 growth_record에서 생육 점수 병합

↓

Embedding Service 호출 (배치 임베딩 요청)

↓

각 case를 자연어 요약 문장으로 변환

↓

Embedding Model 호출 → Vector 생성

↓

Elasticsearch 저장 (cultivation_insight, cultivationId 기준 신규 문서)

↓

Embedding Service → AI Service (저장 완료) → Cultivation Service (임베딩 완료 처리)

`mushroom_environment`와 달리 이벤트 구독이 아닌 스케줄러 기반 배치 호출로 적재되며, "임계치에
도달한 만큼만" 적재하는 방식이라 매일 발생하지 않을 수 있습니다. 자세한 내용은
[insight.md](../04_sequence/insight.md) 참고.

---

# 검색 흐름

## mushroom_environment (챗봇 유사 사례 검색)

사용자 (AI 챗봇 질문)

↓

AI Service

↓

Embedding Service

↓

Embedding 생성

↓

Vector Search

↓

Top-K 검색

↓

AI Service

↓

LLM

↓

챗봇 답변 생성

---

## cultivation_insight (인사이트 조회, 사용자 요청 시점)

사용자 (인사이트 요청)

↓

AI Service

↓

Redis 캐시 조회 (ai:{cultivationId}:insight, Cache Miss 시에만 아래 진행)

↓

Cultivation Service (현재 재배의 환경 평균 조회)

↓

Embedding Service

↓

Elasticsearch (mushroomType term 필터 + avgTemperature range 필터)

↓

Top-K 매칭

↓

AI Service

↓

LLM

↓

인사이트 요약 생성 (매칭 사례 없으면 LLM 미호출, 고정 문구)

---

# 검색 방식

## Vector Search (mushroom_environment)

Cosine Similarity를 이용하여 가장 유사한 환경을 검색합니다.

예시

```
Top 5 Similar Environment
```

---

## Term/Range Filter (cultivation_insight)

Cosine Similarity 기반 Vector Search가 아니라, 명확한 조건으로 먼저 필터링하는 방식입니다.

- `mushroomType`: term 필터 (정확히 일치)
- `avgTemperature`: range 필터 (`요청 온도 - tolerance` ~ `요청 온도 + tolerance`)

인사이트는 "같은 버섯 종류 + 비슷한 온도"라는 조건 자체가 이미 명확하고 좁아서, 벡터 유사도
재랭킹 없이 필터 결과를 그대로(또는 최신순/topK 제한으로) 반환합니다. `embedding` 필드는
스키마에는 존재하지만 현재 검색 로직에서는 사용하지 않으며, 추후 매칭 사례가 많아지면
필터링 후 벡터 유사도로 재랭킹하는 확장을 고려할 수 있습니다(아래 "추후 개발 예정" 참고).

---

## Hybrid Search (추후)

Keyword Search

+

Vector Search

---

# 검색 결과 예시

## mushroom_environment

```json
[
  {
    "mushroomType": "OYSTER",
    "_score": 0.98
  },
  {
    "mushroomType": "KING_OYSTER",
    "_score": 0.94
  }
]
```

## cultivation_insight

```json
[
  {
    "cultivationId": "12",
    "avgTemperature": 21.8,
    "avgHumidity": 89.2,
    "growthScore": 88,
    "harvestWeight": 3200
  },
  {
    "cultivationId": "15",
    "avgTemperature": 22.3,
    "avgHumidity": 90.1,
    "growthScore": 91,
    "harvestWeight": 3400
  }
]
```

term/range 필터 기반이라 `_score`를 응답에 포함하지 않습니다.

---

# Index 관리

## 생성

```
mushroom_environment
cultivation_insight
```

---

## 삭제

개발 환경에서만 허용합니다.

운영 환경에서는 Reindex를 사용합니다.

---

## 재생성

### mushroom_environment

mushroom_reference 데이터가 변경될 경우(MushroomReferenceUpdatedEvent 수신)

Embedding Service가 해당 버섯 종류의 임베딩만 다시 생성합니다(Upsert).

전체 재생성이 필요하면(예: 임베딩 모델 교체) Embedding Service가 Cultivation Service의 전체
mushroom_reference 목록을 다시 조회해 일괄 재생성합니다.

### cultivation_insight

재생성 개념이 없습니다. 완료된 재배 사례가 한 번 임베딩되면 다시 갱신되지 않으며(불변 이력),
새 사례는 배치가 돌 때마다 신규 문서로 계속 추가됩니다. 임베딩 모델 교체 시에는 전체
재생성이 필요할 수 있으나, 이 경우 원본 데이터(환경 평균 등)를 다시 계산해야 하므로
AI Service/Cultivation Service를 통한 재조회가 필요합니다(추후 개발 예정).

---

# 장애 대응

Elasticsearch 장애 발생 시

- Vector Search 불가 (AI 챗봇의 유사 재배 사례 검색만 영향)
- LLM RAG 기능 비활성화
- 인사이트 배치 임베딩 적재 실패 (해당 배치는 완료 처리되지 않고 다음 배치 때 재시도됨)
- 인사이트 조회 시 유사 사례 검색 불가 (AI Service가 고정 안내 문구로 대체)

재배 생성 시 환경 추천(mushroom_reference 조회)은 Cultivation Service PostgreSQL을 직접
조회하는 별개의 경로이므로 Elasticsearch 장애의 영향을 받지 않습니다. 챗봇은 기본 프롬프트를
사용하여 AI 응답을 생성합니다. 일일 피드백(daily_feedback)도 Elasticsearch를 사용하지 않는
별개의 기능이라 영향받지 않습니다.

---

# 고려 사항

- 관계형 데이터의 원본(source of truth)은 Cultivation DB의 `mushroom_reference`이며, 이 인덱스는 그 텍스트를 임베딩과 함께 보관하는 읽기 최적화 사본입니다.
- Embedding Vector는 Cultivation DB(PostgreSQL)에는 저장하지 않고 이 인덱스에만 저장합니다. 데이터 중복 저장을 피하기 위함입니다.
- 텍스트 필드(characteristics/healthBenefits/cultivationGuide/description 등)는 검색/컨텍스트 표시용으로만 사용하며, Cultivation DB의 값과 다르면 안 됩니다(MushroomReferenceUpdatedEvent로 동기화).
- Embedding은 재생성이 가능하므로 영속 데이터가 아닙니다.
- Dense Vector 검색을 위해 KNN Index를 활성화합니다.
- 버섯 종류가 5종으로 고정되어 있어 문서 수도 5개로 매우 작습니다. 대량 데이터를 위한 인덱스가 아니라, "정확한 벡터 검색 인프라를 아키텍처에 갖춘다"는 목적과 챗봇의 유사 사례 검색 기능이 결합된 구조입니다.
- `cultivation_insight`는 `mushroom_environment`와 달리 계속 쌓이는 이력성 인덱스입니다. `environment_setting`(PostgreSQL, INSERT-only 이력)과 비슷한 성격이지만, 임베딩/벡터 검색 인프라가 필요해 Elasticsearch에 둡니다.
- `cultivation_insight`의 원본 데이터(환경 평균, 생육 점수)는 Cultivation DB/AI DB에 이미 존재하는 값을 조합한 것입니다. Elasticsearch 문서는 검색을 위한 사본이며, 데이터가 유실되어도 이론적으로는 재생성할 수 있습니다(다만 위 "재생성" 항목대로 아직 자동화된 재생성 경로는 없음).
- `cultivation_insight`의 검색은 현재 벡터 유사도가 아닌 term/range 필터만 사용합니다. `embedding` 필드는 스키마에 존재하지만 실제 검색 로직에서 사용되지 않는 "예약된" 필드입니다.

---

# 추후 개발 예정

- Hybrid Search
- Metadata Filtering
- Embedding Version 관리
- 다국어 Embedding
- 자동 Reindex
- cultivation_insight 필터링 후 벡터 유사도 기반 재랭킹
- cultivation_insight 오래된 이력에 대한 보관 주기 정책 (environment_setting과 유사한 고민)