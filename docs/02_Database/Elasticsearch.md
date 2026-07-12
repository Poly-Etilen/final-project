# Elasticsearch

## 목적

Embedding된 식물 환경 데이터를 저장한다.

AI Service는 Elasticsearch를 이용하여 RAG를 수행한다.

---

# Index

plant_environment

---

# Document

```json
{
  "plant_name":"몬스테라",

  "temperature":24,

  "humidity":65,

  "co2":700,

  "ph":6.5,

  "embedding":[]
}
```

---

# 검색 과정

```text
사용자 질문

↓

Embedding

↓

Vector Search

↓

Top-K

↓

LLM

↓

추천 결과
```

---

# 저장 데이터

CSV

↓

Embedding Service

↓

Elasticsearch

---

# 특징

- KNN Search

- Cosine Similarity

- Dense Vector