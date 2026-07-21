# Embedding Service

## 역할

Embedding Service는 버섯 재배 데이터를 벡터(Embedding)로 변환하고,
Elasticsearch에 저장하여 AI가 유사한 재배 환경을 검색할 수 있도록 지원하는 서비스입니다.

LLM은 직접 데이터를 검색하지 않으며,
Embedding Service를 통해 검색된 결과를 기반으로 응답을 생성합니다.

> ℹ️ **변경 이력**: "인사이트" 기능(같은 버섯 종류 + 유사한 온도로 재배했던 타인의 사례 기반
> 피드백) 추가를 위해, 기존 `mushroom_environment`(버섯 종류별 고정 참조 데이터) 인덱스와는
> 별개로 `cultivation_insight`(완료된 재배 사례) 인덱스가 새로 생겼습니다. 임베딩 대상 원본
> 데이터도 다릅니다 — `mushroom_environment`는 관리자가 등록하는 5종 고정 텍스트이지만,
> `cultivation_insight`는 AI Service가 00시 배치로 넘겨주는 완료된 재배들의 요약(버섯 종류,
> 기간 가중 평균 환경값, 생육 점수, 수확량)입니다. 검색 방식도 다릅니다 — 기존 유사 사례
> 검색은 챗봇을 위한 벡터 검색이지만, 인사이트 검색은 버섯 종류(정확히 일치) + 온도(오차
> 범위 내)로 먼저 필터링한 뒤 매칭된 사례를 반환하는 방식입니다. (자세한 내용은
> [insight.md](../04_sequence/insight.md), [elasticSearch.md](../03_Database/elasticSearch.md)의
> `cultivation_insight` 참고)

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
- 인사이트 사례 배치 임베딩 (완료된 재배 요약)
- 인사이트 유사 사례 검색 (버섯 종류 + 온도 필터)

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

## 인사이트 사례 배치 임베딩

AI Service의 Insight Batch Scheduler(00시)가 미임베딩 수확 건이 임계치(20건) 이상 쌓였을 때
호출합니다. 완료된 재배 각각을 하나의 "사례" 문서로 만듭니다.

임베딩 대상 데이터 (AI Service가 배치로 전달)

- mushroomType (버섯 종류)
- avgTemperature / avgHumidity / avgCo2 / avgLight (기간 가중 평균 환경값)
- growthScore (생육 점수)
- harvestWeight (수확량)

이 값들을 사람이 읽을 수 있는 한 문장 요약(예: "느타리버섯, 평균 온도 21.8℃·습도 89% 환경에서
생육 점수 88점으로 3.2kg 수확")으로 먼저 만든 뒤 임베딩하고, 원본 수치 필드도 함께 문서에
저장합니다(검색 결과 필터링/표시용). `mushroom_environment`와 같은 원칙입니다 — 비텍스트·짧은
필드는 그대로 저장하고, 자연어로 풀어쓴 요약만 벡터화합니다.

Elasticsearch에는 `cultivation_insight` 인덱스에 저장됩니다. 저장이 성공적으로 끝나야 AI
Service가 Cultivation Service에 해당 건들을 임베딩 완료로 표시하도록 요청합니다(이 완료
처리 자체는 Embedding Service가 아닌 AI Service의 책임입니다).

---

## 인사이트 유사 사례 검색

사용자가 인사이트를 요청하면(사용자 요청 시점) AI Service가 호출합니다. 챗봇의 벡터 유사도
검색과 달리, 먼저 명확한 조건(버섯 종류 정확히 일치 + 온도 오차 범위 내)으로 필터링한 뒤
매칭된 사례들을 반환합니다.

검색 조건

- mushroomType (정확히 일치)
- avgTemperature (요청한 온도 ± 허용 오차 범위 내)

매칭된 사례들의 평균 환경값/생육 점수/수확량을 AI Service에 반환하며, 자연어 요약(LLM
호출)은 AI Service의 책임입니다. Embedding Service는 검색 결과를 그대로 전달할 뿐 스스로
요약하지 않습니다.

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

## 인사이트 사례 배치 임베딩 (내부용)

POST /api/v1/embeddings/insights

AI Service의 Insight Batch Scheduler만 호출하는 내부용 엔드포인트입니다.

---

## 인사이트 유사 사례 검색 (내부용)

POST /api/v1/embeddings/insights/search

AI Service가 인사이트 조회 시(사용자 요청 시점, 캐시 미스 시에만) 호출하는 내부용
엔드포인트입니다.

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

Index (인사이트, 신규)

```
cultivation_insight
```

주요 필드

- cultivationId (문서 ID로도 사용)
- mushroomType
- avgTemperature / avgHumidity / avgCo2 / avgLight
- growthScore / harvestWeight
- summary (자연어 요약 원문)
- embedding(Vector) — summary를 임베딩해 생성

`mushroom_environment`와 별개의 인덱스이며, 원본 데이터의 성격(고정 참조 데이터 vs. 계속
쌓이는 재배 사례)과 소유 서비스(관리자가 등록 vs. AI Service가 배치로 전달)가 다릅니다. 자세한
스키마는 [elasticSearch.md](../03_Database/elasticSearch.md)의 `cultivation_insight` 참고.

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
- 인사이트 사례 배치 임베딩 요청 (Insight Batch Scheduler 실행 시, `POST /api/v1/embeddings/insights`)
- 인사이트 유사 사례 검색 요청 (사용자 요청 시점, `POST /api/v1/embeddings/insights/search`)

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

## 인사이트 사례 배치 임베딩

AI Service (Insight Batch Scheduler, 임계치 20건 도달 시)

↓

Embedding Service (POST /api/v1/embeddings/insights)

↓

각 사례를 자연어 요약 문장으로 변환

↓

Embedding Model

↓

Elasticsearch 저장 (cultivation_insight, 신규 문서)

↓

Embedding Service → AI Service (저장 완료 응답)

↓

AI Service → Cultivation Service (임베딩 완료 처리)

자세한 내용은 [insight.md](../04_sequence/insight.md) 참고.

---

## 인사이트 유사 사례 검색

AI Service (사용자 요청 시점)

↓

Embedding Service (POST /api/v1/embeddings/insights/search, mushroomType + 온도 오차 범위)

↓

Elasticsearch (mushroomType term 필터 + avgTemperature range 필터)

↓

Embedding Service → AI Service (매칭된 사례 목록)

↓

AI Service → LLM (자연어 요약)

자세한 내용은 [insight.md](../04_sequence/insight.md) 참고.

---

# 예외 상황

- Elasticsearch 연결 실패
- Embedding 생성 실패
- 검색 결과 없음
- Index 생성 실패
- 임베딩 데이터 손상
- MushroomReferenceUpdatedEvent 구독 실패 (Elasticsearch가 최신 상태를 반영하지 못함)
- 전체 재생성 시 Cultivation Service 호출 실패
- 인사이트 배치 임베딩 저장 실패 (cultivation_insight 인덱스) — 저장 실패 시 AI Service에 실패를 알리며, AI Service는 해당 건들을 임베딩 완료로 표시하지 않음
- 인사이트 검색 시 매칭되는 사례 없음 (충분히 이상한 상황은 아니며, AI Service가 고정 안내 문구로 처리)

---

# 추후 개발 예정

- Hybrid Search (Keyword + Vector)
- 임베딩 버전 관리
- 재배 데이터 자동 동기화
- 다국어 임베딩 지원