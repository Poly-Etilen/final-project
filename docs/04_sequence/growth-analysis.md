# 생육 분석 시퀀스

## 개요

사용자가 재배 중인 버섯을 직접 촬영하여 업로드하면, AI가 학습된 Vision 모델로
사진을 분석해 생육 상태를 판단하는 과정입니다.

카메라 센서가 자동으로 촬영하는 방식이 아니라, 사용자가 앱/웹에서 사진을 찍어 업로드하는 방식입니다.

Vision 모델은 사전에 다양한 성장 단계의 사진으로 학습되어 있으며,
업로드된 사진을 그 학습된 패턴과 비교하여 균사 성장률, 갓 크기, 색상, 병충해 여부를 판단합니다.

LLM은 Vision 모델이 산출한 지표를 새로 추정하지 않고, 그 지표를 해석하는 설명과 개선 방안만 생성합니다.

harvest.md의 수확 전 생육 분석과 동일한 분석 로직을 사용하지만,
이 시퀀스는 수확 여부와 관계없이 재배 상세 화면에서 사용자가 원할 때마다 사진을 찍어 실행할 수 있습니다.

> ℹ️ **변경 이력**: 분석 결과가 `ai:{cultivationId}:analysis` Redis 캐시(TTL 6시간)뿐 아니라
> `growth_record` 테이블(PostgreSQL, AI DB)에도 영구 저장되도록 바뀌었습니다. "일일
> 피드백"(사용자가 환경을 수정했을 때 생육이 실제로 어떻게 달라졌는지 매일 비교해 알려주는
> 기능)이 여러 날짜에 걸친 생육 추이를 비교해야 하는데, 6시간짜리 캐시만으로는 하루만
> 지나도 비교할 데이터가 사라지기 때문입니다. Redis 캐시는 "방금 분석한 결과를 재요청 없이
> 즉시 재조회"하는 성능 캐시로 그대로 유지되고, `growth_record`는 만료되지 않는 이력
> 저장소입니다. (자세한 내용은 [ai-db.md](../03_Database/ai-db.md),
> [daily-feedback.md](./daily-feedback.md) 참고)

---

# Sequence

```text
Client (사진 촬영)

↓

API Gateway

↓

Cultivation Service

↓

MinIO 저장

↓

photo 메타데이터 저장

↓

PostgreSQL

↓

AI Service (image_url 전달)

↓

Vision 모델 분석

↓

생육 점수 계산

↓

LLM

↓

결과 해석 및 개선 방안 생성

↓

Redis 저장 + growth_record 저장 (PostgreSQL)

↓

Cultivation Service

↓

Client
```

---

# 상세 과정

## 1. 사진 업로드 요청

사용자가 카메라로 촬영한 사진을 업로드합니다.

예시

```http
POST /cultivations/{cultivationId}/photos
```

Multipart Form

```
image: mushroom.jpg
```

---

## 2. MinIO 저장

Cultivation Service

↓

MinIO PUT Object

↓

image_url 발급

예시

```
https://minio/mushroom-photos/3/20260815-090000.jpg
```

---

## 3. photo 메타데이터 저장

Cultivation Service는

photo 테이블에 저장합니다.

```json
{
    "cultivationId":3,
    "imageUrl":"https://minio/mushroom-photos/3/20260815-090000.jpg",
    "uploadedAt":"2026-08-15T09:00:00"
}
```

---

## 4. AI Service 호출

Cultivation Service

↓

OpenFeign

↓

AI Service

전달 데이터

```json
{
    "cultivationId":3,
    "mushroomType":"OYSTER",
    "imageUrl":"https://minio/mushroom-photos/3/20260815-090000.jpg"
}
```

---

## 5. Vision 모델 분석

AI Service의 Vision 모델이 image_url로 사진을 조회하여 분석합니다.

Vision 모델은 사전에 학습된 성장 단계별 이미지 패턴과 비교하여 판단합니다.

분석 항목

- 균사 성장률
- 갓 크기
- 색상 분석
- 병충해 분석

출력

```json
{
    "myceliumGrowthRate":82,
    "capSize":"중(3.2cm)",
    "colorStatus":"정상",
    "diseaseStatus":"정상"
}
```

---

## 6. 생육 점수 계산

AI Service가 4가지 지표를 종합하여 점수를 계산합니다.

```
생육 점수 = 균사 성장률 40% + 갓 크기 점수 30% + 색상 점수 15% + 병충해 점수 15%
```

병충해가 감지된 경우 별도로 감점합니다.

성장 단계(균사기 / 자실체 형성기 / 성장기 / 수확 적기)를 기준으로 예상 수확 시기를 추정합니다.

---

## 7. LLM 해석

AI Service

↓

LLM

입력

- 버섯 종류
- 생육 점수
- 균사 성장률 / 갓 크기 / 색상 / 병충해 여부
- 성장 단계

LLM은 위 지표를 새로 추정하지 않고, 해석 문장과 개선 방안만 생성합니다.

출력

```json
{
    "growthScore":85,
    "myceliumGrowthRate":82,
    "capSize":"중(3.2cm)",
    "colorStatus":"정상",
    "diseaseStatus":"정상",
    "growthStage":"자실체 형성기",
    "expectedHarvestDate":"2026-08-20",
    "improvement":"균사 성장은 양호하나 조도가 다소 낮게 유지되고 있어 갓 형성이 더딜 수 있습니다. LED 점등 시간을 늘려보세요."
}
```

---

## 8. Redis 저장 + growth_record 저장

Redis

Key

```
ai:{cultivationId}:analysis
```

TTL

```
6시간
```

이후 같은 사진에 대한 재조회는 Redis에서 즉시 반환됩니다.

PostgreSQL (AI DB)

이와 별개로 분석 결과를 `growth_record`에 한 행으로 영구 저장합니다. Redis 캐시는 TTL이
지나면 사라지지만, `growth_record`는 만료되지 않고 계속 쌓여 "일일 피드백"이 여러 날짜의
생육 추이를 비교하는 데 사용됩니다. (자세한 내용은 [daily-feedback.md](./daily-feedback.md) 참고)

---

## 9. 결과 반환

AI Service

↓

Cultivation Service

↓

Client

사용자는 방금 찍은 사진에 대한 생육 점수와 분석 결과를 즉시 확인합니다.

---

# 최근 분석 결과만 조회 (재촬영 없이)

새 사진을 찍지 않고 마지막 분석 결과만 다시 보고 싶은 경우

```http
GET /cultivations/{cultivationId}/analysis
```

Redis에 저장된 마지막 결과를 반환하며, Vision 모델을 다시 실행하지 않습니다.

---

# 사용 Database

## MinIO

사용자가 업로드한 사진 저장 (Cultivation Service가 저장, AI Service가 읽기 전용 조회)

---

## PostgreSQL

```
photo (Cultivation DB)
growth_record (AI DB)
```

---

## Redis

AI 생육 분석 결과 Cache

---

# OpenFeign

```
Cultivation

↓

AI
```

---

# RabbitMQ

사용하지 않습니다.

사진 업로드와 생육 분석은 사용자 요청 기반의 동기 처리입니다.

---

# 예외 상황

- 존재하지 않는 재배
- 이미 종료된 재배
- 지원하지 않는 파일 형식
- 사진 업로드 실패 (MinIO 저장 실패)
- Vision 모델 분석 실패
- LLM 응답 실패
- Redis 장애
- growth_record 저장 실패 (PostgreSQL) — 저장에 실패해도 분석 응답 자체는 사용자에게 반환합니다

---

# 고려 사항

- 생육 점수, 균사 성장률, 갓 크기, 색상, 병충해 여부, 성장 단계는 Vision 모델이 산출한 결과이며 LLM이 추정하지 않습니다.
- LLM은 계산된 지표를 해석하는 설명과 개선 방안 생성에만 사용합니다.
- 사진은 카메라 센서가 아닌 사용자가 앱/웹에서 직접 촬영하여 업로드합니다.
- 사진 업로드와 분석은 하나의 요청(POST /photos)으로 함께 처리됩니다.
- FINISHED 상태인 재배는 생육 분석 대신 재배 이력 정보를 제공합니다.
- 분석 결과는 Redis(`ai:{cultivationId}:analysis`, TTL 6시간, 빠른 재조회용)와 `growth_record`
  (PostgreSQL, 영구 보관, 일일 피드백용) 두 곳에 함께 저장됩니다. 사진 자체(photo 테이블)는
  Cultivation DB에 이력으로 계속 보관됩니다.
- 캐시 TTL은 리포트(24시간)보다 짧게 설정하여 비교적 최신 상태를 반영합니다.
