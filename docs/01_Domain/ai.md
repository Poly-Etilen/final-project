# AI Service

## 역할

AI Service는 LLM과 RAG(Retrieval-Augmented Generation)를 활용하여 사용자에게 지능형 서비스를 제공합니다.

사용자의 재배 환경을 추천하고, 센서 데이터를 분석하여 생육 상태를 예측하며, AI 리포트와 챗봇 기능을 제공합니다.

---

# 책임

- AI 환경 추천
- AI 생육 분석
- AI 챗봇
- AI 리포트 생성
- 프롬프트 관리
- AI 응답 캐싱

---

# 주요 기능

## AI 재배 환경 추천

사용자가 재배하려는 버섯 종류를 입력하면
Embedding Service를 통해 유사한 재배 데이터를 검색합니다.

검색된 데이터를 기반으로 LLM이 최적의 환경을 생성합니다.

추천 항목

- 목표 온도
- 목표 습도
- 목표 CO₂ 농도
- 목표 조도
- 환기 주기

---

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

## 환경 추천

POST /ai/environment

---

## 생육 분석

POST /ai/analysis

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

- 환경 추천 결과
- AI 분석 결과
- AI 리포트

---

# MinIO

AI Service는 Vision 분석을 위해 사진을 조회합니다.

사진을 직접 저장하지는 않으며, Cultivation Service가 저장한 이미지를 읽기 전용으로 사용합니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Embedding Service

- 유사 환경 검색

---

### Sensor Service

- 센서 데이터 조회
- 주간 데이터 조회
- 월간 데이터 조회

---

## 호출받는 서비스

### Cultivation Service

- 환경 추천 요청
- 생육 사진 Vision 분석 요청 (사진 URL 포함)

### API Gateway

- AI 챗봇 요청

---

# Event

현재 이벤트를 발행하지 않습니다.

---

# Sequence

## AI 환경 추천

Client

↓

Gateway

↓

Cultivation Service

↓

AI Service

↓

Embedding Service

↓

Elasticsearch

↓

유사 환경 검색

↓

LLM

↓

환경 추천 생성

↓

Cultivation Service

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
- LLM 응답 실패
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