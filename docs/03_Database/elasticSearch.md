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

| Field | Type | Description |
|--------|------|-------------|
| id | Keyword | 문서 ID |
| mushroomType | Keyword | 버섯 종류 |
| temperature | Float | 권장 온도 |
| humidity | Float | 권장 습도 |
| co2 | Integer | 권장 CO₂ |
| light | Integer | 권장 조도 |
| description | Text | 재배 환경 설명 |
| embedding | Dense Vector | 임베딩 벡터 |

---

# Mapping 예시

```json
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "mushroomType": {
        "type": "keyword"
      },
      "temperature": {
        "type": "float"
      },
      "humidity": {
        "type": "float"
      },
      "co2": {
        "type": "integer"
      },
      "light": {
        "type": "integer"
      },
      "description": {
        "type": "text"
      },
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
  "id": "ENV-001",
  "mushroomType": "OYSTER",
  "temperature": 22.0,
  "humidity": 91.0,
  "co2": 850,
  "light": 420,
  "description": "느타리버섯 생육에 적합한 환경",
  "embedding": [
    0.124,
    -0.215,
    ...
  ]
}
```

---

# 데이터 생성 흐름

CSV

↓

DatasourceGenerator

↓

Embedding Service

↓

Embedding Model

↓

Vector 생성

↓

Elasticsearch 저장

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

CSV 데이터가 변경될 경우

Embedding Service가 전체 임베딩을 다시 생성합니다.

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

- 관계형 데이터는 저장하지 않습니다.
- Embedding Vector만 저장합니다.
- PostgreSQL과 데이터를 중복 저장하지 않습니다.
- Embedding은 재생성이 가능하므로 영속 데이터가 아닙니다.
- Dense Vector 검색을 위해 KNN Index를 활성화합니다.

---

# 추후 개발 예정

- Hybrid Search
- Metadata Filtering
- Embedding Version 관리
- 다국어 Embedding
- 자동 Reindex