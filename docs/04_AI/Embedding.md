# Embedding Service

## 목적

Embedding Service는 식물 생육 데이터를 벡터화하여 Elasticsearch에 저장한다.

AI Service는 이 데이터를 검색하여 RAG(Retrieval-Augmented Generation)를 수행한다.

---

## 데이터 출처

CSV

예시

```csv
plant,temperature,humidity,co2,ph

몬스테라,24,65,700,6.5

산세베리아,22,50,500,6.8
```

---

## 처리 과정

```mermaid
flowchart LR

CSV

Parser

Embedding

VectorDB

CSV --> Parser

Parser --> Embedding

Embedding --> VectorDB
```

---

## 저장 데이터

```json
{
    "plant":"몬스테라",

    "temperature":24,

    "humidity":65,

    "co2":700,

    "ph":6.5,

    "embedding":[]
}
```

---

## 검색 방식

- KNN Search
- Cosine Similarity
- Top-K Search