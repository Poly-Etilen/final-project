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

Sensor Service에서 제공하는 센서 데이터를 기반으로 현재 재배 상태를 분석합니다.

제공 정보

- 생육 점수
- 환경 유지율
- 예상 수확 시기
- 예상 품질
- 환경 개선 사항

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
- 생육 분석 요청

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
- API 호출 시간 초과

---

# 추후 개발 예정

- 사용자 맞춤형 프롬프트
- AI 응답 품질 평가
- 모델 교체(OpenAI, Gemini 등)
- 다국어 지원
- AI 재배 코칭