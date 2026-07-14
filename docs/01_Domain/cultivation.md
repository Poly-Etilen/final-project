# Cultivation Service

## 역할

Cultivation Service는 사용자의 버섯 재배 정보를 관리하는 핵심 서비스입니다.

사용자는 새로운 재배를 생성하고, AI가 추천한 재배 환경을 수정하여 저장할 수 있으며,
재배 진행 상황과 수확 결과를 관리할 수 있습니다.

---

# 책임

- 재배 생성
- 재배 조회
- 재배 수정
- 재배 삭제
- 환경 설정 저장
- 재배 종료
- 수확 정보 저장
- 재배 이력 관리

---

# 주요 기능

## 재배 생성

새로운 버섯 재배를 생성합니다.

사용자는

- 재배 이름
- 버섯 종류

를 입력합니다.

이후 AI Service를 호출하여 최적의 환경을 추천받습니다.

사용자가 추천 환경을 수정한 후 저장하면
최종 환경 설정이 Database에 저장됩니다.

---

## 재배 목록 조회

사용자가 생성한 모든 재배 목록을 조회합니다.

조회 정보

- 재배 이름
- 버섯 종류
- 현재 상태
- 생성일

---

## 재배 상세 조회

재배의 상세 정보를 조회합니다.

조회 정보

- 환경 설정
- 센서 상태
- 생육 상태
- 생성일
- 수정일

---

## 재배 환경 수정

사용자는 현재 재배 환경을 수정할 수 있습니다.

수정 항목

- 목표 온도
- 목표 습도
- 목표 CO₂
- 목표 조도

---

## 재배 종료

재배를 종료합니다.

종료 시

- 종료일 저장
- 상태 변경
- 수확 정보 입력

---

## 수확 기록 저장

재배 종료 후

- 수확량
- 메모

를 저장합니다.

---

## 재배 이력 조회

완료된 재배 목록을 조회합니다.

---

# API

## 재배 생성

POST /cultivations

---

## 재배 목록 조회

GET /cultivations

---

## 재배 상세 조회

GET /cultivations/{cultivationId}

---

## 재배 수정

PATCH /cultivations/{cultivationId}

---

## 재배 삭제

DELETE /cultivations/{cultivationId}

---

## 환경 설정 저장

PATCH /cultivations/{cultivationId}/environment

---

## 재배 종료

PATCH /cultivations/{cultivationId}/finish

---

## 수확 정보 저장

PATCH /cultivations/{cultivationId}/harvest

---

## 재배 이력 조회

GET /cultivations/history

---

# Database

Cultivation Service는 별도의 PostgreSQL Database를 사용합니다.

### Table

- cultivation
- cultivation_environment
- harvest_history

---

# Redis

현재 사용하지 않습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### AI Service

환경 추천 요청

생육 분석 요청

AI 리포트 조회

---

### Sensor Service

현재 센서 상태 조회

환경 통계 조회

실시간 센서 데이터 조회

---

## 호출받는 서비스

API Gateway

---

# Event

## 발행 이벤트

### CultivationCreatedEvent

재배 생성 완료

---

### CultivationFinishedEvent

재배 종료

---

### HarvestCompletedEvent

수확 완료

---

# Sequence

## 재배 생성

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

LLM

↓

환경 추천

↓

Client

↓

사용자 수정

↓

Cultivation Service

↓

PostgreSQL 저장

---

## 재배 종료

Client

↓

Gateway

↓

Cultivation Service

↓

Harvest 저장

↓

재배 종료

↓

Event 발행

---

# 예외 상황

- 존재하지 않는 재배
- 이미 종료된 재배
- 권한 없는 재배 접근
- AI 추천 실패
- 환경 설정 저장 실패

---

# 추후 개발 예정

- 재배 템플릿 저장
- 즐겨찾는 환경 저장
- 자동 재배 스케줄 설정
- 재배 목표 설정