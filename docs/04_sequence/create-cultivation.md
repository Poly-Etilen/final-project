# 재배 생성 시퀀스

## 개요

사용자가 새로운 버섯 재배를 시작하는 과정입니다.

재배 생성 시 AI는 버섯 종류에 맞는 최적의 환경을 추천하며,
사용자가 추천값을 수정하거나 그대로 저장하면 재배가 시작됩니다.

---

# Sequence

```text
Client

↓

API Gateway

↓

Cultivation Service

↓

PostgreSQL

↓

Cultivation 생성

↓

AI Service

↓

Embedding Service

↓

Elasticsearch

↓

유사 환경 검색

↓

AI Service

↓

LLM

↓

환경 추천 생성

↓

Cultivation Service

↓

Client

↓

사용자 확인

↓

환경 저장

↓

Cultivation Service

↓

Environment Setting 저장

↓

재배 시작
```

---

# 상세 과정

## 1. 재배 생성 요청

사용자는

- 재배 이름
- 버섯 종류

를 입력합니다.

예시

```json
{
    "name":"느타리 1호기",
    "mushroomType":"OYSTER"
}
```

---

## 2. Cultivation 생성

Cultivation Service는

재배 정보를 생성합니다.

초기 상태

```
CREATED
```

아직 환경 정보는 저장하지 않습니다.

---

## 3. AI 추천 요청

Cultivation Service

↓

OpenFeign

↓

AI Service

전달 데이터

```json
{
    "mushroomType":"OYSTER"
}
```

---

## 4. Embedding 검색

AI Service

↓

Embedding Service

↓

Embedding 생성

↓

Elasticsearch

↓

Vector Search

↓

Top K 검색

예시

```
Top 5
```

유사 환경 반환

---

## 5. LLM 환경 추천

AI Service

↓

LLM

입력

- 버섯 종류
- Vector Search 결과

출력

```json
{
    "temperature":22,
    "humidity":91,
    "co2":850,
    "light":420
}
```

---

## 6. 추천 결과 반환

AI Service

↓

Cultivation Service

↓

Client

사용자는 추천 환경을 확인합니다.

---

## 7. 환경 수정

사용자는

- Temperature
- Humidity
- CO₂
- Light

를 수정할 수 있습니다.

---

## 8. 환경 저장

사용자가

저장 버튼을 누르면

Cultivation Service

↓

단일 목표값을 허용 오차만큼 확장하여 범위(min~max)로 변환

↓

Environment Setting이 생성됩니다.

```text
environment_setting
```

예시 (허용 오차 적용)

```
Temperature 22℃ → temp_min 20.5 / temp_max 23.5
Humidity 91% → humidity_min 86 / humidity_max 96
```

API 요청/응답에는 단일 목표값만 노출되며, 범위 변환은 Cultivation Service 내부 저장 로직입니다.

---

## 9. 재배 시작

Cultivation 상태 변경

```
CREATED

↓

RUNNING
```

재배가 시작됩니다.

---

# 사용 Database

## PostgreSQL

```
cultivation

environment_setting
```

---

## Elasticsearch

Vector Search

---

# OpenFeign

```
Cultivation

↓

AI

↓

Embedding
```

---

# RabbitMQ

사용하지 않습니다.

재배 생성은 사용자 응답이 필요한 기능이므로
동기 방식(OpenFeign)으로 처리합니다.

---

# 상태 변화

초기

```
CREATED
```

↓

환경 저장

↓

```
RUNNING
```

↓

수확 완료

↓

```
FINISHED
```

---

# 예외 상황

- 존재하지 않는 버섯 종류
- AI 추천 실패
- Elasticsearch 검색 실패
- LLM 응답 실패
- Environment 저장 실패
- DB 저장 실패

---

# 고려 사항

- AI 추천 환경은 Database에 저장하지 않습니다.
- 사용자가 최종 저장한 환경만 저장합니다.
- 저장 시 단일 목표값은 허용 오차만큼 확장된 범위(min~max)로 변환되어 저장됩니다. API 스펙은 단일값을 그대로 유지합니다.
- 재배는 환경 저장 이후 RUNNING 상태가 됩니다.
- Embedding 검색 실패 시 기본 프롬프트를 사용하여 AI 추천을 수행합니다.