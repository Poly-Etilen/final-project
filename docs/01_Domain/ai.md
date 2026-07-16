# AI Service

## 역할

AI Service는 LLM과 RAG(Retrieval-Augmented Generation)를 활용하여 사용자에게 지능형 서비스를 제공합니다.

센서 데이터를 분석하여 생육 상태를 예측하며, AI 리포트와 챗봇 기능을 제공합니다.

> ℹ️ **변경 이력**: 재배 환경 추천은 원래 AI Service가 Embedding/Vector Search/LLM을 통해
> 생성했지만, 공공데이터 기준 버섯 종류가 5가지로 고정되어 있어 항상 같은 값이 나오는 조회에는
> AI가 불필요하다고 판단해 Cultivation Service의 `mushroom_reference` 참조 테이블 조회로
> 이전했습니다. AI Service는 더 이상 환경 추천을 담당하지 않습니다.

> ℹ️ **변경 이력**: 재배 생성 직후 사용자에게 버섯 효능/재배 시 주의사항을 자연어로 보여주는
> "버섯 가이드" 기능이 추가되었습니다. 환경 추천(수치, 범위)과 달리 이 기능은 사람이 읽기 좋은
> 설명 문서를 생성하는 것이 목적이라 LLM을 그대로 활용하기로 했습니다. 다만 버섯 종류가
> 5가지로 고정되어 있는 것은 환경 추천 때와 동일하므로, 재배(cultivation)가 아닌
> **버섯 종류(mushroomType) 기준으로 캐싱**하여 동일 종류에 대해 매번 LLM을 호출하지 않도록
> 했습니다.

> ℹ️ **변경 이력**: `mushroom_reference`(Cultivation DB)에 특성/효능/재배 가이드/추가 정보
> 원문 텍스트가 추가되면서, "버섯 가이드"는 이 원문을 무(無)에서 생성하지 않고 **RAG 컨텍스트로
> 활용**합니다. AI Service가 Cultivation Service를 OpenFeign으로 호출해(`GET
> /api/v1/mushroom-references/{mushroomType}`) 원문 텍스트를 가져온 뒤, LLM이 이를 참고해
> 더 자연스러운 문장의 `benefits`/`precautions`로 다듬어 응답합니다. 원문을 그대로 반환하지
> 않는 이유는, 공공데이터 원문이 항목별로 파편화되어 있어 사용자에게는 자연어로 통합된 설명이
> 더 읽기 좋기 때문입니다.

---

# 책임

- AI 생육 분석
- AI 챗봇
- AI 리포트 생성
- 버섯 가이드(효능/주의사항) 생성
- 프롬프트 관리
- AI 응답 캐싱

---

# 주요 기능

## AI 생육 분석

환경 데이터만으로 추정하지 않고, 사용자가 직접 촬영하여 업로드한 사진을 Vision 모델로 분석하여
실제 생육 상태를 판단합니다.

카메라 센서가 자동으로 촬영하는 방식이 아니라, 사용자가 앱/웹에서 사진을 찍어 업로드하면
그 사진을 AI가 학습된 기준(성장 단계별 학습 데이터)과 비교하여 분석하는 방식입니다.

Vision 모델은 AI Service 내부에 포함되며, LLM과는 별도로 동작합니다.

### 분석 흐름

1. Cultivation Service가 사용자로부터 업로드받은 사진의 URL을 AI Service에 전달합니다.

사진 원본은 Cultivation Service가 MinIO에 저장한 상태이며, AI Service는 전달받은 URL로 이미지를 조회합니다.

---

2. AI Service의 Vision 모델이 사진을 분석합니다.

Vision 모델은 사전에 다양한 성장 단계의 사진으로 학습되어 있으며,
입력된 사진을 학습된 패턴과 비교하여 아래 지표를 산출합니다.

```json
{
    "myceliumGrowthRate":82,
    "capSize":"중(3.2cm)",
    "colorStatus":"정상",
    "diseaseStatus":"정상"
}
```

분석 항목

- 균사 성장률 : 균사가 배지를 덮은 비율
- 갓 크기 : 버섯 갓의 지름
- 색상 분석 : 정상 / 변색 / 갈변 등 색상 상태
- 병충해 분석 : 정상 / 의심 / 감염 여부

---

3. AI Service가 4가지 지표를 종합하여 생육 점수를 계산합니다.

```
생육 점수 = 균사 성장률 40% + 갓 크기 점수 30% + 색상 점수 15% + 병충해 점수 15%
```

병충해가 감지된 경우 생육 점수를 별도로 감점합니다.

---

4. 성장 단계(균사기 / 자실체 형성기 / 성장기 / 수확 적기)를 기준으로 예상 수확 시기를 추정합니다.

---

5. LLM은 Vision 모델이 산출한 지표(생육 점수, 성장 단계, 병충해 여부)를 입력받아
   결과를 해석하는 자연어 설명과 개선 방안만 생성합니다.
   지표 자체를 새로 추정하지 않습니다.

### 제공 정보

- 생육 점수
- 균사 성장률
- 갓 크기
- 색상 상태
- 병충해 여부
- 성장 단계
- 예상 수확 시기
- 환경 개선 사항 (LLM 생성)

---

## 버섯 가이드 생성 (효능/주의사항)

재배를 생성한 직후, 선택한 버섯 종류의 효능과 재배 시 주의사항을 자연어 문서로 보여줍니다.

`mushroom_reference.description`(짧은 한 줄 참고 문구, Cultivation Service가 재배 생성 응답에
그대로 포함)과는 별개입니다. 버섯 가이드는 그보다 훨씬 자세한 설명(효능, 주의사항)을 LLM으로
생성하는 별도 기능이며, Client가 재배 생성 이후 AI Service를 직접 호출해서 받습니다
(Cultivation Service를 거치지 않습니다. 다만 AI Service는 응답을 만들기 위해 내부적으로
Cultivation Service를 OpenFeign으로 호출합니다).

`mushroom_reference`에 저장된 characteristics(특성)/health_benefits(효능)/cultivation_guide
(재배 가이드)/additional_info(추가 정보) 원문을 Cultivation Service로부터 가져와 LLM
프롬프트의 RAG 컨텍스트로 사용합니다. LLM은 이 원문을 그대로 반환하지 않고, 자연스러운 문장의
`benefits`/`precautions`로 재구성합니다.

버섯 종류는 공공데이터 기준 5가지로 고정되어 있어 같은 종류라면 항상 같은 내용이 나오므로,
`mushroomType` 기준으로 캐싱해 동일 종류에 대한 반복 LLM 호출을 피합니다. (재배 환경 추천을
AI에서 뺀 이유와 같은 문제이지만, 이번에는 수치가 아닌 "설명 문서"를 만드는 것이라 LLM을
그대로 사용하기로 했습니다.)

### 제공 정보

- 효능 (buffs)
- 재배 시 주의사항 (precautions)

---

## AI 챗봇

사용자는 자연어로 재배 관련 질문을 할 수 있습니다.

예시

- 왜 성장이 느린가요?
- 현재 환경은 적절한가요?
- 언제 수확하면 좋을까요?
- 생산량을 늘리려면 어떻게 해야 하나요?

---

## AI 리포트

Sensor Service에서 집계한
주간 및 월간 데이터를 분석하여
AI 리포트를 생성합니다.

제공 내용

- 환경 유지율
- 평균 환경 데이터
- 자동 제어 횟수
- 이상 환경 발생 내역
- 개선 제안

---

# API

## 생육 분석

POST /ai/analysis

---

## 버섯 가이드 생성

POST /ai/mushroom-guide

---

## AI 챗봇

POST /ai/chat

---

## AI 리포트 생성

POST /ai/report

---

# Database

AI Service는 별도의 관계형 데이터베이스를 사용하지 않습니다.

---

# Redis

AI 응답 캐시를 저장합니다.

캐시 대상

- AI 분석 결과
- AI 리포트
- AI 챗봇 응답
- 버섯 가이드 (mushroomType 기준, cultivationId와 무관)

---

# MinIO

AI Service는 Vision 분석을 위해 사진을 조회합니다.

사진을 직접 저장하지는 않으며, Cultivation Service가 저장한 이미지를 읽기 전용으로 사용합니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Embedding Service

- AI 챗봇의 유사 재배 사례 검색 (선택적 호출)

---

### Sensor Service

- 센서 데이터 조회
- 주간 데이터 조회
- 월간 데이터 조회

---

### Cultivation Service

- 버섯 가이드 생성 시 `GET /api/v1/mushroom-references/{mushroomType}` 호출 (RAG 컨텍스트 조회, 캐시 미스 시에만)

---

## 호출받는 서비스

### Cultivation Service

- 생육 사진 Vision 분석 요청 (사진 URL 포함)

### API Gateway

- AI 챗봇 요청
- 버섯 가이드 요청 (Client가 직접 호출, Cultivation Service를 거치지 않음)

---

# Event

## 발행 이벤트

### WeeklyReportCompletedEvent

Sensor Service의 Weekly Scheduler가 집계한 데이터를 받아 AI 리포트 생성이 완료되면 발행합니다.

구독 서비스: Notification Service

---

### MonthlyReportCompletedEvent

Monthly Scheduler 기반 AI 리포트 생성이 완료되면 발행합니다.

구독 서비스: Notification Service

---

생육 분석(Vision), 챗봇은 사용자 요청에 대한 동기 응답으로 결과가 즉시 전달되므로 별도 이벤트를 발행하지 않습니다.

---

# Sequence

## 버섯 가이드 생성

Client (재배 생성 완료 직후)

↓

AI Service

↓

Redis 캐시 조회 (ai:mushroom:{mushroomType}:guide)

↓

Cache Hit → 즉시 반환

↓

Cache Miss → Cultivation Service OpenFeign 호출 (`GET /api/v1/mushroom-references/{mushroomType}`, RAG 컨텍스트 조회)

↓

LLM 호출 (원문을 참고해 효능/주의사항 생성) → Redis 캐시 저장 (TTL 7일) → 반환

↓

Client

---

## AI 리포트 생성

Scheduler

↓

Sensor Service

↓

주간 데이터 집계

↓

AI Service

↓

LLM

↓

AI 리포트 생성

↓

Client

---

# 예외 상황

- Embedding 검색 실패
- LLM 응답 실패 (버섯 가이드 생성 실패 포함)
- 버섯 가이드 생성 시 Cultivation Service 호출 실패 (RAG 컨텍스트 조회 실패)
- Redis Cache 조회 실패
- Sensor 데이터 부족
- 등록된 사진 없음
- Vision 모델 분석 실패
- API 호출 시간 초과

---

# 추후 개발 예정

- 사용자 맞춤형 프롬프트
- AI 응답 품질 평가
- 모델 교체(OpenAI, Gemini 등)
- 다국어 지원
- AI 재배 코칭