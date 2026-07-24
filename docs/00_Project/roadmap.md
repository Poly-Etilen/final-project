# 프로젝트 로드맵

## 1단계 - 프로젝트 기획

### 목표

프로젝트의 전체 방향성과 요구사항을 정의합니다.

### 작업 내용

- 프로젝트 주제 선정
- 요구사항 분석
- 기능 정의
- 기술 스택 선정
- MSA 아키텍처 설계
- DB 선정
- 서비스 분리

---

## 2단계 - 프로젝트 설계

### 목표

서비스별 구조와 데이터 모델을 설계합니다.

### 작업 내용

- ERD 설계
- API 명세 작성
- 서비스 간 통신 설계
- OpenFeign 설계
- RabbitMQ 이벤트 설계
- MQTT Topic 설계
- Rule Engine 규칙 설계

---

## 3단계 - 개발 환경 구축

### 목표

프로젝트 개발을 위한 공통 환경을 구축합니다.

### 작업 내용

- GitHub Repository 구성
- GitHub Actions 설정
- Docker 환경 구축
- Kubernetes 환경 구성
- Eureka Server 구축
- API Gateway 구축
- 공통 라이브러리 작성

---

## 4단계 - 핵심 서비스 개발

### 목표

각 MSA 서비스를 구현합니다.

### 개발 대상

7개 서비스(API Gateway 포함)로 구성합니다.

- API Gateway
- Auth Service
- Cultivation Service (재배/수확/사진/상품 등급, 생육/수확 모드 자동 전환, 재배 멤버
  관리(공유), 센서 장치/목표 환경/버섯 참조 데이터 관리, 측정값 저장/조회, 통계·차트,
  일일 피드백용 일간 통계 집계, 문의(Inquiry))
- AI Service
- Rule Engine Service (MQTT 수신/Collector, 검증, 규칙 평가, 자동 제어, 센서 오류 감지)
- Notification Service
- DatasourceGenerator

---

## 5단계 - AI 기능 구현

### 목표

AI 기반 기능을 구현합니다.

### 작업 내용

- RAG (참조 데이터/인사이트 사례 직접 조회 기반, 임베딩·벡터 검색 없음)
- AI 챗봇 (웹/앱 + Telegram/Discord)
- Vision 모델 연동 (생육 사진 분석)
- 생육 분석 / 일일 피드백(생육 추이 비교 + 환경 통계) / 인사이트

환경 추천(`mushroom_reference` 참조 테이블 조회)은 AI가 아닌 Cultivation/Sensor
Service의 기능이므로 4단계(핵심 서비스 개발)에서 함께 구현합니다.

---

## 6단계 - IoT 기능 구현

### 목표

실시간 센서 데이터 처리 및 자동 제어를 구현합니다.

### 작업 내용

- MQTT Broker 연동
- 센서 데이터 수집
- Rule Engine 구현
- 자동 환경 제어
- RabbitMQ 이벤트 처리

---

## 7단계 - 프론트엔드 연동

### 목표

사용자 인터페이스를 구현합니다.

### 작업 내용

- 로그인 화면 (LOCAL + 구글)
- 대시보드
- 재배 생성 (환경 추천 포함)
- 실시간 차트
- 생육 사진 업로드/분석 화면
- AI 챗봇 / 일일 피드백 / 인사이트

---

## 8단계 - 테스트

### 목표

서비스의 안정성을 확보합니다.

### 작업 내용

- 단위 테스트
- 통합 테스트
- API 테스트
- 성능 테스트
- 장애 테스트

---

## 9단계 - 배포

### 목표

서비스를 운영 환경에 배포합니다.

### 작업 내용

- Docker Image 생성
- GitHub Actions CI
- Kubernetes 배포
- Rolling Update
- 모니터링 환경 구축

---

# 마일스톤

| 단계 | 목표 |
|------|------|
| Milestone 1 | 요구사항 및 설계 완료 |
| Milestone 2 | MSA 환경 구축 |
| Milestone 3 | 핵심 서비스 구현 완료 |
| Milestone 4 | AI 및 IoT 기능 구현 완료 |
| Milestone 5 | 통합 테스트 완료 |
| Milestone 6 | 최종 배포 및 발표 |

---

# 최종 목표

- 공공데이터 기반 버섯 재배 환경 추천
- IoT 기반 실시간 센서 모니터링
- Rule Engine 기반 자동 환경 제어
- AI 생육 분석 및 수확 예측
- 일일 피드백(생육 추이 비교 + 환경 통계), 인사이트(타인의 유사 재배 사례 기반
  피드백) 제공
- Kubernetes 기반 MSA 서비스 운영
