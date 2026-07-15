# Embedding API

## 개요

Embedding Service에서 제공하는 REST API 명세입니다.

버섯 재배 데이터를 벡터로 변환하여 Elasticsearch에 저장하고, 유사 환경 검색을 제공합니다.
주로 AI Service가 내부적으로 호출하며, 관리자 도구를 통한 직접 호출도 가능합니다.

Base URL

```
/api/v1/embeddings
```

인증 방식

```
Bearer JWT (관리자 권한 필요 - 생성/재생성/삭제)
```

---

# 임베딩 생성

## POST /

버섯 재배 참조 데이터를 임베딩으로 변환하여 저장합니다.

### Request

```json
{
    "mushroomType": "OYSTER",
    "temperature": 22.0,
    "humidity": 91.0,
    "co2": 850,
    "light": 420,
    "description": "느타리버섯 생육에 적합한 환경"
}
```

---

### Process

Embedding Service

↓

Embedding Model 호출

↓

Vector 생성

↓

Elasticsearch 저장 (mushroom_environment)

---

### Response

```json
{
    "id": "ENV-001",
    "message": "임베딩이 생성되었습니다."
}
```

---

# 벡터 검색

## POST /search

### Request

```json
{
    "mushroomType": "OYSTER",
    "topK": 5
}
```

---

### Process

Embedding Service

↓

Elasticsearch Vector Search (Cosine Similarity)

---

### Response

```json
[
    { "mushroomType": "OYSTER", "temperature": 22.0, "humidity": 91.0, "co2": 850, "light": 420, "score": 0.98 },
    { "mushroomType": "KING_OYSTER", "temperature": 21.0, "humidity": 88.0, "co2": 800, "light": 400, "score": 0.94 }
]
```

---

# 임베딩 재생성

## POST /rebuild

CSV/DB의 참조 데이터가 변경된 경우 전체 임베딩을 다시 생성합니다.

### Response

```json
{
    "message": "임베딩 재생성이 시작되었습니다.",
    "targetCount": 42
}
```

---

# 임베딩 삭제

## DELETE /{embeddingId}

### Response

```json
{
    "message": "임베딩이 삭제되었습니다."
}
```

---

# Error Code

| Code | Description |
|------|-------------|
| E001 | Elasticsearch 연결 실패 |
| E002 | Embedding 생성 실패 |
| E003 | 검색 결과 없음 |
| E004 | Index 생성 실패 |
| E005 | 존재하지 않는 임베딩 |

---

# OpenFeign

호출받는 서비스

```
AI Service (챗봇 유사 재배 사례 검색, 선택적 호출)
DatasourceGenerator (선택 - 새 재배 데이터 등록 시 임베딩 생성 요청)
```

호출하는 서비스

```
없음
```

---

# Event

## 발행 이벤트

- EmbeddingCreatedEvent
- EmbeddingUpdatedEvent

현재는 내부 로깅 목적으로만 사용하며, 구독하는 서비스는 없습니다.
