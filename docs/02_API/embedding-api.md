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

> ℹ️ **변경 이력**: 임베딩 원본 데이터가 CSV/DatasourceGenerator에서 Cultivation DB의
> `mushroom_reference`로 바뀌었습니다. 평상시 임베딩 생성/갱신은 Cultivation Service가 발행하는
> `MushroomReferenceUpdatedEvent`를 구독해 자동으로 이루어지며, 아래 `POST /`는 수동/관리자
> 트리거용으로 남겨둡니다.

---

# 임베딩 생성

## POST /

버섯 재배 참조 데이터를 임베딩으로 변환하여 저장합니다. 평상시에는 이 API 대신
`MushroomReferenceUpdatedEvent` 구독으로 자동 처리되며, 이 엔드포인트는 수동 재시도/관리자
트리거 용도입니다.

### Request

```json
{
    "mushroomType": "OYSTER",
    "mushroomNameKo": "느타리버섯",
    "mushroomNameEn": "Oyster Mushroom",
    "mushroomScientificName": "Pleurotus ostreatus",
    "temperature": {"min": 15.0, "max": 18.0},
    "humidity": {"min": 85, "max": 95},
    "co2": {"min": 700, "max": 900},
    "light": {"min": 300, "max": 400},
    "description": "느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다.",
    "characteristics": "군생하며 갓은 회갈색~담회색을 띠고, 균사 성장 속도가 빠른 편입니다.",
    "healthBenefits": "식이섬유와 베타글루칸이 풍부해 면역력 강화와 콜레스테롤 감소에 도움을 줍니다.",
    "cultivationGuide": "다습한 환경을 선호하지만 환기가 부족하면 곰팡이가 발생하기 쉬우니 CO₂ 농도 관리에 유의해야 합니다.",
    "additionalInfo": null
}
```

---

### Process

Embedding Service

↓

characteristics/healthBenefits/cultivationGuide/additionalInfo 결합

↓

Embedding Model 호출

↓

Vector 생성

↓

Elasticsearch Upsert (mushroom_environment, mushroomType 기준)

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
    { "mushroomType": "OYSTER", "mushroomNameKo": "느타리버섯", "score": 0.98 },
    { "mushroomType": "KING_OYSTER", "mushroomNameKo": "새송이버섯", "score": 0.94 }
]
```

---

# 임베딩 재생성

## POST /rebuild

Cultivation DB의 `mushroom_reference` 참조 데이터가 변경된 경우 전체 임베딩을 다시 생성합니다.
내부적으로 Cultivation Service의 `GET /api/v1/mushroom-references`(전체 목록)를 OpenFeign으로
조회합니다. 평상시에는 `MushroomReferenceUpdatedEvent`로 건별 동기화되므로, 이 API는 임베딩
모델 교체 등 예외적인 경우에만 사용합니다.

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
```

호출하는 서비스

```
Cultivation Service (전체 재생성 시에만, GET /api/v1/mushroom-references 전체 목록 조회)
```

---

# Event

## 구독 이벤트

- MushroomReferenceUpdatedEvent (Cultivation Service 발행) — 해당 mushroomType의 임베딩을 재생성해 Elasticsearch에 Upsert

## 발행 이벤트

- EmbeddingCreatedEvent
- EmbeddingUpdatedEvent

현재는 내부 로깅 목적으로만 사용하며, 구독하는 서비스는 없습니다.
