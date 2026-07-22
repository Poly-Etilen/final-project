# 생육 분석 시퀀스

## 개요

사용자가 재배 중인 버섯을 직접 촬영해 업로드하면, AI가 Vision 모델로 사진을 분석해
생육 상태를 판단하는 과정입니다. 카메라 센서가 자동으로 촬영하는 방식이 아니라, 사용자가
앱/웹에서 사진을 찍어 업로드하는 방식입니다.

Vision 모델은 사전에 학습된 성장 단계별 이미지 패턴과 업로드된 사진을 비교해 균사
성장률/갓 크기/색상/병충해 여부를 판단합니다. LLM은 이 지표를 새로 추정하지 않고, 해석
문장과 개선 방안만 생성합니다.

[harvest.md](./harvest.md)의 수확 전 생육 분석과 동일한 로직을 사용하지만, 이 시퀀스는
수확 여부와 관계없이 재배 상세 화면에서 사용자가 원할 때마다 실행할 수 있습니다.

---

# Sequence

```text
Client (사진 촬영)
↓
API Gateway
↓
Cultivation Service
↓
Photo Storage 저장 (MinIO 또는 Local)
↓
photo 메타데이터 저장 (PostgreSQL, Cultivation DB)
↓
AI Service (objectKey/storageType 전달)
↓
Vision 모델 분석
↓
생육 점수 계산
↓
LLM 해석
↓
Redis 저장 + growth_record 저장 (PostgreSQL, AI DB)
↓
Cultivation Service
↓
Client
```

---

# 상세 과정

## 1. 사진 업로드 요청

```http
POST /api/v1/cultivations/{cultivationId}/photos
```

Multipart Form으로 이미지를 전달합니다.

---

## 2. Photo Storage 저장

Cultivation Service → Photo Storage(`storage_type` 설정에 따라 MinIO 또는 로컬) 저장

저장 후 `object_key`(예: `3/20260815-090000.jpg`)를 발급받습니다. 전체 URL이 아닌
저장소 내 상대 경로만 발급받으며, 실제 접근 URL/경로 변환은 조회 시점에 서비스 설정
기준으로 계산합니다.

---

## 3. photo 메타데이터 저장

`photo` 테이블(Cultivation DB)에 저장합니다.

```json
{
    "cultivationId": 3,
    "objectKey": "3/20260815-090000.jpg",
    "storageType": "MINIO",
    "uploadedAt": "2026-08-15T09:00:00"
}
```

---

## 4. AI Service 호출

Cultivation Service → OpenFeign → AI Service (`POST /api/v1/ai/analysis`)

```json
{
    "cultivationId": 3,
    "photoId": 7,
    "objectKey": "3/20260815-090000.jpg",
    "storageType": "MINIO"
}
```

---

## 5. Vision 모델 분석

AI Service의 Vision 모델이 `objectKey`로 Photo Storage에서 사진을 조회해 분석합니다.

분석 항목: 균사 성장률, 갓 크기, 색상, 병충해 여부

```json
{
    "myceliumGrowthRate": 82.0,
    "capSize": "MEDIUM",
    "color": "정상",
    "diseaseDetected": false
}
```

---

## 6. 생육 점수 계산

AI Service가 4가지 지표를 종합해 점수를 계산합니다.

```
생육 점수 = 균사 성장률 40% + 갓 크기 점수 30% + 색상 점수 15% + 병충해 점수 15%
```

병충해가 감지된 경우 별도로 감점합니다. 성장 단계(균사기/자실체 형성기/생장기/수확
적기)를 기준으로 예상 수확일을 추정합니다.

---

## 7. LLM 해석

AI Service → LLM

입력: 버섯 종류, 생육 점수, 균사 성장률/갓 크기/색상/병충해 여부, 성장 단계

LLM은 위 지표를 새로 추정하지 않고 해석 문장과 개선 방안만 생성합니다.

```json
{
    "growthScore": 85,
    "growthStage": "자실체 형성기",
    "expectedHarvestDate": "2026-08-20",
    "improvement": "균사 성장은 양호하나 조도가 다소 낮게 유지되고 있어 갓 형성이 더딜 수 있습니다. LED 점등 시간을 늘려보세요."
}
```

---

## 8. Redis 저장 + growth_record 저장

Redis: `ai:{cultivationId}:analysis`, TTL 6시간 — 이후 같은 결과의 재조회는 즉시 반환됩니다.

PostgreSQL(AI DB): 이와 별개로 분석 결과를 `growth_record`에 한 행으로 영구 저장합니다.
Redis 캐시는 TTL이 지나면 사라지지만 `growth_record`는 만료되지 않고 계속 쌓여, 일일
피드백([daily-feedback.md](./daily-feedback.md))이 여러 날짜의 생육 추이를 비교하는 데
사용됩니다.

---

## 9. 결과 반환

AI Service → Cultivation Service → Client

사용자는 방금 찍은 사진에 대한 생육 점수와 분석 결과를 즉시 확인합니다.

---

# 사용 Database

## Photo Storage

사용자가 업로드한 사진 저장 (Cultivation Service가 저장, AI Service가 읽기 전용 조회)

## PostgreSQL

```
photo (Cultivation DB)
growth_record (AI DB)
```

## Redis

AI 생육 분석 결과 캐시 (`ai:{cultivationId}:analysis`, TTL 6시간)

---

# OpenFeign

```
Cultivation Service → AI Service
```

---

# RabbitMQ

사용하지 않습니다. 사진 업로드와 생육 분석은 사용자 요청 기반의 동기 처리입니다.

---

# 예외 상황

- 존재하지 않는 재배 / 이미 종료된 재배
- 지원하지 않는 파일 형식
- Photo Storage 저장 실패
- Vision 모델 분석 실패 / LLM 응답 실패
- Redis 장애
- `growth_record` 저장 실패 (저장에 실패해도 분석 응답 자체는 사용자에게 반환)

---

# 고려 사항

- 생육 점수, 균사 성장률, 갓 크기, 색상, 병충해 여부, 성장 단계는 Vision 모델이 산출한
  결과이며 LLM이 새로 추정하지 않습니다. LLM은 해석 문장과 개선 방안 생성에만 사용합니다.
- 사진은 카메라 센서가 아닌 사용자가 앱/웹에서 직접 촬영해 업로드합니다.
- 사진 업로드와 분석은 하나의 요청(`POST .../photos`)으로 함께 처리됩니다.
- 분석 결과는 Redis(빠른 재조회용, TTL 6시간)와 `growth_record`(영구 보관, 일일 피드백용)
  두 곳에 함께 저장됩니다. 사진 자체(`photo`)는 Cultivation DB에 이력으로 계속 보관됩니다.
- `object_key`+`storage_type`만 저장하므로, 추후 저장소를 MinIO에서 로컬로(또는 그
  반대로) 바꾸더라도 기존 행을 다시 쓸 필요가 없습니다.
