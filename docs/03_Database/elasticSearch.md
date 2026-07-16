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
```

---

# Document 구조

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

# Mapping 예시

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

# 저장 예시

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

# 데이터 생성 흐름

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

# 검색 흐름

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

# 검색 방식

## Vector Search

Cosine Similarity를 이용하여 가장 유사한 환경을 검색합니다.

예시

```
Top 5 Similar Environment
```

---

## Hybrid Search (추후)

Keyword Search

+

Vector Search

---

# 검색 결과 예시

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

---

# Index 관리

## 생성

```
mushroom_environment
```

---

## 삭제

개발 환경에서만 허용합니다.

운영 환경에서는 Reindex를 사용합니다.

---

## 재생성

mushroom_reference 데이터가 변경될 경우(MushroomReferenceUpdatedEvent 수신)

Embedding Service가 해당 버섯 종류의 임베딩만 다시 생성합니다(Upsert).

전체 재생성이 필요하면(예: 임베딩 모델 교체) Embedding Service가 Cultivation Service의 전체
mushroom_reference 목록을 다시 조회해 일괄 재생성합니다.

---

# 장애 대응

Elasticsearch 장애 발생 시

- Vector Search 불가 (AI 챗봇의 유사 재배 사례 검색만 영향)
- LLM RAG 기능 비활성화

재배 생성 시 환경 추천(mushroom_reference 조회)은 Cultivation Service PostgreSQL을 직접
조회하는 별개의 경로이므로 Elasticsearch 장애의 영향을 받지 않습니다. 챗봇은 기본 프롬프트를
사용하여 AI 응답을 생성합니다.

---

# 고려 사항

- 관계형 데이터의 원본(source of truth)은 Cultivation DB의 `mushroom_reference`이며, 이 인덱스는 그 텍스트를 임베딩과 함께 보관하는 읽기 최적화 사본입니다.
- Embedding Vector는 Cultivation DB(PostgreSQL)에는 저장하지 않고 이 인덱스에만 저장합니다. 데이터 중복 저장을 피하기 위함입니다.
- 텍스트 필드(characteristics/healthBenefits/cultivationGuide/description 등)는 검색/컨텍스트 표시용으로만 사용하며, Cultivation DB의 값과 다르면 안 됩니다(MushroomReferenceUpdatedEvent로 동기화).
- Embedding은 재생성이 가능하므로 영속 데이터가 아닙니다.
- Dense Vector 검색을 위해 KNN Index를 활성화합니다.
- 버섯 종류가 5종으로 고정되어 있어 문서 수도 5개로 매우 작습니다. 대량 데이터를 위한 인덱스가 아니라, "정확한 벡터 검색 인프라를 아키텍처에 갖춘다"는 목적과 챗봇의 유사 사례 검색 기능이 결합된 구조입니다.

---

# 추후 개발 예정

- Hybrid Search
- Metadata Filtering
- Embedding Version 관리
- 다국어 Embedding
- 자동 Reindex