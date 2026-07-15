# AI API

## 개요

AI Service에서 제공하는 REST API 명세입니다.

Base URL

```
/api/v1/ai
```

인증 방식

```
Bearer JWT
```

내부 서비스(Cultivation Service 등) 호출은 OpenFeign을 통해 이루어집니다.

---

# 환경 추천

## POST /environment

버섯 종류를 기반으로 최적의 재배 환경을 추천합니다. (Vector Search 기반)

### Request

```json
{
    "mushroomType": "OYSTER"
}
```

---

### Process

AI Service

↓

Embedding Service

↓

Elasticsearch (Vector Search)

↓

Top-K 유사 환경 검색

↓

LLM

---

### Response

```json
{
    "temperature": 22,
    "humidity": 90,
    "co2": 800,
    "light": 350,
    "reason": "느타리버섯은 서늘하고 다습한 환경에서 균사 활착이 빠릅니다."
}
```

---

# 생육 분석 (Vision)

## POST /analysis

사용자가 업로드한 사진을 Vision 모델로 분석합니다.

카메라 센서가 아닌, 사용자가 직접 촬영하여 업로드한 사진을 대상으로 합니다.

### Request

```json
{
    "cultivationId": 3,
    "mushroomType": "OYSTER",
    "imageUrl": "https://minio/mushroom-photos/3/20260815-090000.jpg"
}
```

---

### Process

AI Service

↓

MinIO에서 이미지 조회 (imageUrl)

↓

Vision 모델 분석

↓

균사 성장률 / 갓 크기 / 색상 / 병충해 판별

↓

생육 점수 계산 (균사 40% + 갓크기 30% + 색상 15% + 병충해 15%)

↓

LLM (결과 해석 및 개선 방안 생성)

---

### Response

```json
{
    "growthScore": 85,
    "myceliumGrowthRate": 82,
    "capSize": "중(3.2cm)",
    "colorStatus": "정상",
    "diseaseStatus": "정상",
    "growthStage": "자실체 형성기",
    "expectedHarvestDate": "2026-08-20",
    "improvement": "균사 성장은 양호하나 조도가 다소 낮게 유지되고 있어 갓 형성이 더딜 수 있습니다."
}
```

Vision 모델이 산출한 지표(growthScore, myceliumGrowthRate, capSize, colorStatus, diseaseStatus, growthStage, expectedHarvestDate)는
결정론적으로 계산되며, LLM은 improvement(해석/개선 방안)만 생성합니다.

---

# AI 챗봇

## POST /chat

### Request

```json
{
    "cultivationId": 3,
    "message": "왜 성장이 느린가요?"
}
```

---

### Process

AI Service

↓

Redis 캐시 조회 (ai:{hash})

↓

Cache Miss 시 Rule Engine Service 조회 + Embedding Service 유사 사례 검색 (선택)

↓

LLM

---

### Response

```json
{
    "answer": "현재 습도가 목표보다 5% 낮은 상태가 지속되고 있어 생육 속도가 느려졌을 수 있습니다. 가습기 가동 주기를 조금 더 짧게 설정하는 것을 권장합니다."
}
```

---

# AI 리포트 생성

## POST /report

### Request

```json
{
    "cultivationId": 3,
    "period": "weekly"
}
```

`period`는 `weekly` 또는 `monthly` 입니다.

---

### Process

AI Service

↓

Redis 캐시 조회 (report:{cultivationId}:{period})

↓

Cache Miss 시 Rule Engine Service 통계 조회 (InfluxDB)

↓

LLM

---

### Response

```json
{
    "period": "weekly",
    "averageTemperature": 22.1,
    "averageHumidity": 90.3,
    "environmentMaintainRate": 95,
    "autoControlCount": 6,
    "report": "지난 7일 동안 평균 온도는 22.1℃로 적정 범위를 유지했습니다. 전체적으로 매우 안정적인 재배 환경을 유지하고 있습니다."
}
```

---

# Error Code

| Code | Description |
|------|-------------|
| AI001 | Embedding 검색 실패 |
| AI002 | LLM 응답 실패 |
| AI003 | Redis Cache 조회 실패 |
| AI004 | Sensor 데이터 부족 |
| AI005 | 등록된 사진 없음 |
| AI006 | Vision 모델 분석 실패 |
| AI007 | API 호출 시간 초과 |

---

# OpenFeign

호출하는 서비스

```
Embedding Service (유사 환경 검색)
Rule Engine Service (센서 데이터/통계 조회)
```

호출받는 서비스

```
Cultivation Service (환경 추천, 생육 사진 Vision 분석 요청)
API Gateway (AI 챗봇 요청)
```

---

# Redis

캐시 대상

- 환경 추천 결과
- AI 생육 분석 결과 (ai:{cultivationId}:analysis, TTL 6시간)
- AI 챗봇 응답 (ai:{hash}, TTL 24시간)
- AI 리포트 (report:{cultivationId}:{period}, TTL 24시간)

---

# MinIO

사용자가 업로드한 사진을 읽기 전용으로 조회합니다. (Cultivation Service가 저장)
