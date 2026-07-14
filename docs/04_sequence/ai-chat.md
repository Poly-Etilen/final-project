# AI 챗봇 시퀀스

## 개요

사용자가 자연어로 재배 관련 질문을 하면 AI가 현재 센서 데이터와 유사 재배 사례를 참고하여
답변을 생성하는 과정입니다.

동일한 질문에 대한 반복 호출을 줄이기 위해 Redis에 응답을 캐싱합니다.

---

# Sequence

```text
Client

↓

API Gateway

↓

AI Service

↓

Redis 조회

↓

Cache Miss

↓

Sensor Service

↓

현재 환경 / 통계 조회

↓

Embedding Service

↓

Elasticsearch

↓

유사 재배 사례 검색

↓

AI Service

↓

LLM

↓

답변 생성

↓

Redis 저장

↓

Client
```

---

# 상세 과정

## 1. 챗봇 질문 요청

사용자가 자연어로 질문합니다.

예시

```http
POST /ai/chat
```

```json
{
    "cultivationId":3,
    "message":"왜 성장이 느린가요?"
}
```

---

## 2. Redis 캐시 조회

AI Service는 먼저 Redis를 조회합니다.

Key

```
ai:{hash}
```

hash는 cultivationId와 질문 내용을 조합하여 생성합니다.

↓

존재하면

↓

즉시 반환

---

## 3. Cache Miss

Redis에 없으면

↓

Sensor Service 호출

(OpenFeign)

---

## 4. 센서 데이터 조회

Sensor Service

↓

현재 환경 (Redis)

↓

최근 통계 (InfluxDB)

조회 항목

- 현재 온도 / 습도 / CO₂ / 조도
- 최근 7일 평균값
- 목표 환경 대비 유지율

---

## 5. 유사 재배 사례 검색

질문이 환경/생육과 관련된 경우

AI Service

↓

Embedding Service

↓

Elasticsearch

↓

Vector Search

↓

Top-K 유사 사례 반환

환경과 무관한 질문(예: 단순 인사)은 이 단계를 건너뜁니다.

---

## 6. LLM 질의

AI Service

↓

LLM

입력

- 사용자 질문
- 버섯 종류
- 현재 센서 데이터
- 목표 환경
- 유사 재배 사례 (선택)

---

## 7. 답변 생성

예시

```
현재 습도가 목표보다 5% 낮은 상태가
지속되고 있어 생육 속도가 느려졌을 수 있습니다.

가습기 가동 주기를 조금 더 짧게
설정하는 것을 권장합니다.
```

---

## 8. Redis 저장

Key

```
ai:{hash}
```

TTL

```
24시간
```

---

## 9. 응답 반환

Client

↓

챗봇 답변 출력

---

# 사용 Database

## Redis

AI 응답 Cache

---

## InfluxDB

센서 통계 조회 (Sensor Service 경유)

---

## Elasticsearch

유사 재배 사례 검색 (Embedding Service 경유)

---

# OpenFeign

```
AI Service

↓

Sensor Service

↓

Embedding Service
```

---

# RabbitMQ

사용하지 않습니다.

챗봇은 사용자 요청 기반의 동기 처리입니다.

---

# 예외 상황

- Redis 장애
- Sensor Service 호출 실패
- Embedding 검색 실패
- LLM 응답 실패
- 부적절한 질문 입력

---

# 고려 사항

- 챗봇 질문은 매번 다를 수 있어 캐시 히트율이 낮을 수 있습니다.
- 유사 사례 검색은 환경/생육 관련 질문에서만 수행하여 불필요한 호출을 줄입니다.
- 대화 이력은 별도로 저장하지 않으며, 매 질문은 독립적으로 처리됩니다.
- InfluxDB 원본 데이터가 아닌 집계된 통계만 LLM에 전달하여 토큰 사용량을 줄입니다.
