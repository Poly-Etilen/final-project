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

> ℹ️ **변경 이력**: 환경 추천 API(`POST /environment`)는 제거되었습니다. 버섯 종류가 공공데이터
> 기준 5가지로 고정되어 있어 Vector Search/LLM 없이도 항상 동일한 값이 나오므로, Cultivation
> Service가 자체 참조 테이블(`mushroom_reference`)을 직접 조회하는 방식으로 이전했습니다.
> 자세한 내용은 [cultivation-api.md](./cultivation-api.md)의 "재배 생성" 참고.

> ℹ️ **변경 이력**: `POST /mushroom-guide`가 새로 추가되었습니다. 재배 생성 직후 버섯의
> 효능/재배 주의사항을 자연어로 보여주는 기능으로, Client가 재배 생성 응답을 받은 뒤 이
> 엔드포인트를 직접 호출합니다(Cultivation Service를 거치지 않음). 환경 추천과 달리 이 기능은
> "설명 문서 생성"이 목적이라 LLM을 그대로 사용하되, `mushroomType`(5종 고정) 기준으로
> 캐싱해 동일 종류에 대한 반복 호출을 피합니다.

> ℹ️ **변경 이력**: `mushroom_reference`(Cultivation DB)에 특성/효능/재배 가이드/추가 정보
> 원문이 추가되면서, 캐시 미스 시 AI Service가 Cultivation Service를 OpenFeign으로 호출해
> (`GET /api/v1/mushroom-references/{mushroomType}`) 이 원문을 RAG 컨텍스트로 가져온 뒤
> LLM에 전달합니다. LLM은 원문을 그대로 반환하지 않고 자연스러운 문장으로 재구성합니다.

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

# 버섯 가이드

## POST /mushroom-guide

재배 생성 직후, 선택한 버섯 종류의 효능/재배 시 주의사항을 자연어로 생성합니다. Client가
재배 생성 응답(`cultivationId`, `recommendedEnvironment`)을 받은 뒤 바로 호출하는 것을
가정하지만, `cultivationId`는 필요하지 않습니다(버섯 종류에만 의존).

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

Redis 캐시 조회 (ai:mushroom:{mushroomType}:guide)

↓

Cache Miss 시 Cultivation Service OpenFeign 호출 (`GET /api/v1/mushroom-references/{mushroomType}`, RAG 컨텍스트 조회)

↓

LLM 호출 (characteristics/healthBenefits/cultivationGuide/additionalInfo를 참고해 효능/주의사항 생성) → Redis에 캐시 저장 (TTL 7일)

---

### Response

```json
{
    "mushroomType": "OYSTER",
    "benefits": "느타리버섯은 식이섬유와 베타글루칸이 풍부해 면역력 강화에 도움을 줍니다.",
    "precautions": "다습한 환경을 선호하지만 환기가 부족하면 곰팡이가 발생하기 쉬우니 CO₂ 농도 관리에 유의하세요."
}
```

`benefits`/`precautions`는 `mushroom_reference`의 원문(characteristics/healthBenefits/
cultivationGuide/additionalInfo)을 LLM이 참고해 재구성한 결과이며, 원문을 그대로 반환하지
않습니다.

버섯 종류가 5가지로 고정되어 있어 같은 `mushroomType`이면 항상 같은 응답이 캐시에서
반환됩니다. `mushroom_reference.description`(짧은 한 줄 참고 문구, 재배 생성 응답에 포함)과는
별개의 콘텐츠입니다.

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

Cache Miss 시 Sensor Service 조회 + Embedding Service 유사 사례 검색 (선택)

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

Cache Miss 시 Sensor Service 통계 조회 (InfluxDB)

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
| AI001 | Embedding 검색 실패 (챗봇 유사 사례 검색) |
| AI002 | LLM 응답 실패 |
| AI003 | Redis Cache 조회 실패 |
| AI004 | Sensor 데이터 부족 |
| AI005 | 등록된 사진 없음 |
| AI006 | Vision 모델 분석 실패 |
| AI007 | API 호출 시간 초과 |
| AI008 | 버섯 가이드 생성 실패 (LLM 응답 실패, Cultivation Service RAG 컨텍스트 조회 실패 포함) |

---

# OpenFeign

호출하는 서비스

```
Embedding Service (챗봇 유사 재배 사례 검색, 선택적 호출)
Sensor Service (센서 데이터/통계 조회)
Cultivation Service (버섯 가이드 RAG 컨텍스트 조회, GET /api/v1/mushroom-references/{mushroomType}, 캐시 미스 시에만)
```

호출받는 서비스

```
Cultivation Service (생육 사진 Vision 분석 요청)
API Gateway (AI 챗봇 요청, 버섯 가이드 요청)
```

---

# Redis

캐시 대상

- AI 생육 분석 결과 (ai:{cultivationId}:analysis, TTL 6시간)
- AI 챗봇 응답 (ai:{hash}, TTL 24시간)
- AI 리포트 (report:{cultivationId}:{period}, TTL 24시간)
- 버섯 가이드 (ai:mushroom:{mushroomType}:guide, TTL 7일) — cultivationId가 아닌 mushroomType 기준으로 캐싱됩니다.

---

# MinIO

사용자가 업로드한 사진을 읽기 전용으로 조회합니다. (Cultivation Service가 저장)
