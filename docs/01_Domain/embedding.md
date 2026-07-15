# Embedding Service

## 역할

Embedding Service는 버섯 재배 데이터를 벡터(Embedding)로 변환하고,
Elasticsearch에 저장하여 AI가 유사한 재배 환경을 검색할 수 있도록 지원하는 서비스입니다.

LLM은 직접 데이터를 검색하지 않으며,
Embedding Service를 통해 검색된 결과를 기반으로 응답을 생성합니다.

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

CSV 또는 데이터베이스에 저장된 버섯 재배 데이터를
Embedding Model을 이용하여 벡터로 변환합니다.

생성 항목

- 버섯 종류
- 적정 온도
- 적정 습도
- 적정 CO₂ 농도
- 적정 조도
- 설명

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

재배 데이터가 변경되거나 추가될 경우
임베딩을 다시 생성하여 Elasticsearch를 최신 상태로 유지합니다.

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

- mushroomType
- temperature
- humidity
- co2
- light
- description
- embedding(Vector)

---

# Redis

사용하지 않습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

없음

---

## 호출받는 서비스

### AI Service

- AI 챗봇의 유사 재배 사례 검색 요청 (선택적 호출)

> ℹ️ **변경 이력**: 재배 생성 시의 환경 추천 요청은 더 이상 들어오지 않습니다. Cultivation
> Service가 자체 참조 테이블(mushroom_reference)을 직접 조회하는 방식으로 대체되었습니다.

### DatasourceGenerator (선택)

- 새로운 재배 데이터 등록 시 임베딩 생성 요청

---

# Event

## EmbeddingCreatedEvent

새로운 임베딩 생성

---

## EmbeddingUpdatedEvent

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

DatasourceGenerator

↓

Embedding Service

↓

Embedding Model

↓

Elasticsearch 저장

---

# 예외 상황

- Elasticsearch 연결 실패
- Embedding 생성 실패
- 검색 결과 없음
- Index 생성 실패
- 임베딩 데이터 손상

---

# 추후 개발 예정

- Hybrid Search (Keyword + Vector)
- 임베딩 버전 관리
- 재배 데이터 자동 동기화
- 다국어 임베딩 지원