# Embedding Service

## 역할

Embedding Service는 버섯 재배 데이터를 벡터(Embedding)로 변환하고,
Elasticsearch에 저장하여 AI가 유사한 재배 환경을 검색할 수 있도록 지원하는 서비스입니다.

LLM은 직접 데이터를 검색하지 않으며,
Embedding Service를 통해 검색된 결과를 기반으로 응답을 생성합니다.

> ℹ️ **변경 이력**: 임베딩 원본 데이터가 CSV/DatasourceGenerator에서 Cultivation DB의
> `mushroom_reference`로 바뀌었습니다. Cultivation Service가 관리자의 등록/수정에 따라
> `MushroomReferenceUpdatedEvent`를 발행하면 이를 구독해 characteristics/healthBenefits/
> cultivationGuide/additionalInfo 텍스트를 임베딩합니다. DatasourceGenerator는 센서 데이터
> 생성/발행만 담당하며 이 데이터와 무관합니다.

---

# 책임

- 재배 데이터 임베딩 생성
- Elasticsearch 벡터 저장
- 벡터 검색
- 임베딩 데이터 관리
- 임베딩 재생성

---

# 주요 기능

## 데이터 임베딩

Cultivation Service의 `mushroom_reference`에 있는 텍스트 데이터를
Embedding Model을 이용하여 벡터로 변환합니다.

임베딩 대상 텍스트

- characteristics (특성)
- health_benefits (효능)
- cultivation_guide (재배 가이드)
- additional_info (추가 정보)

위 네 필드를 결합한 텍스트로 벡터를 생성하며, mushroomType/이름/환경 범위/description 같은
비텍스트·짧은 필드는 임베딩하지 않고 문서에 그대로 함께 저장합니다(검색 결과 표시/필터링용).

---

## 벡터 저장

생성된 임베딩을 Elasticsearch에 저장합니다.

---

## 유사 환경 검색

사용자가 재배하려는 버섯과 가장 유사한 환경 데이터를
Vector Search를 통해 검색합니다.

검색 결과는 AI Service로 전달됩니다.

---

## 임베딩 재생성

`mushroom_reference`가 변경되면(MushroomReferenceUpdatedEvent 수신) 해당 버섯 종류의 임베딩만
다시 생성해 Elasticsearch를 최신 상태로 유지합니다(Upsert). 임베딩 모델 교체 등으로 전체
재생성이 필요하면 Cultivation Service의 전체 목록(`GET /api/v1/mushroom-references`)을
OpenFeign으로 조회해 일괄 재생성합니다.

---

# API

## 임베딩 생성

POST /embeddings

---

## 벡터 검색

POST /embeddings/search

---

## 임베딩 재생성

POST /embeddings/rebuild

---

## 임베딩 삭제

DELETE /embeddings/{embeddingId}

---

# Database

Embedding Service는 관계형 데이터베이스를 사용하지 않습니다.

---

# Elasticsearch

Index

```
mushroom_environment
```

주요 필드

- mushroomType (문서 ID로도 사용)
- mushroomNameKo / mushroomNameEn / mushroomScientificName
- tempMin / tempMax / humidityMin / humidityMax / co2Min / co2Max / lightMin / lightMax
- description
- characteristics / healthBenefits / cultivationGuide / additionalInfo
- embedding(Vector) — characteristics/healthBenefits/cultivationGuide/additionalInfo를 결합해 생성

---

# Redis

사용하지 않습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Cultivation Service

- 전체 재생성 시에만 `GET /api/v1/mushroom-references`(전체 목록) 호출. 평상시 동기화는 MushroomReferenceUpdatedEvent 구독으로 처리하므로 이 호출은 드뭅니다.

---

## 호출받는 서비스

### AI Service

- AI 챗봇의 유사 재배 사례 검색 요청 (선택적 호출)

> ℹ️ **변경 이력**: 재배 생성 시의 환경 추천 요청은 더 이상 들어오지 않습니다. Cultivation
> Service가 자체 참조 테이블(mushroom_reference)을 직접 조회하는 방식으로 대체되었습니다.

> ℹ️ **변경 이력**: DatasourceGenerator로부터의 임베딩 생성 요청은 더 이상 존재하지 않습니다.
> DatasourceGenerator는 센서 데이터 생성/발행만 담당하며 버섯 참조 데이터와 무관합니다.
> 임베딩 대상 데이터는 이제 Cultivation Service의 `mushroom_reference`이며,
> `MushroomReferenceUpdatedEvent` 구독을 통해 동기화합니다.

---

# Event

## 구독 이벤트

### MushroomReferenceUpdatedEvent

Cultivation Service가 `mushroom_reference`를 등록/수정할 때 발행합니다. 해당 mushroomType의
텍스트(characteristics/healthBenefits/cultivationGuide/additionalInfo)를 임베딩해
Elasticsearch의 mushroom_environment 인덱스를 Upsert합니다. 버섯 종류가 5종 고정이라 매우
드물게 발생합니다.

---

## 발행 이벤트

### EmbeddingCreatedEvent

새로운 임베딩 생성

---

### EmbeddingUpdatedEvent

기존 임베딩 갱신

---

# Sequence

## 챗봇 유사 사례 검색

AI Service

↓

Embedding Service

↓

Elasticsearch

↓

Top-K Vector Search

↓

Embedding Service

↓

AI Service

↓

LLM

↓

챗봇 답변

---

## 임베딩 생성

Cultivation Service (관리자가 mushroom_reference 등록/수정)

↓

RabbitMQ Publish (MushroomReferenceUpdatedEvent)

↓

Embedding Service 구독

↓

Embedding Model

↓

Elasticsearch Upsert (mushroom_environment)

---

# 예외 상황

- Elasticsearch 연결 실패
- Embedding 생성 실패
- 검색 결과 없음
- Index 생성 실패
- 임베딩 데이터 손상
- MushroomReferenceUpdatedEvent 구독 실패 (Elasticsearch가 최신 상태를 반영하지 못함)
- 전체 재생성 시 Cultivation Service 호출 실패

---

# 추후 개발 예정

- Hybrid Search (Keyword + Vector)
- 임베딩 버전 관리
- 재배 데이터 자동 동기화
- 다국어 임베딩 지원