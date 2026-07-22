# 기술 스택

## Backend

- Java 21
- Spring Boot 4.0.7 (Spring Framework 7.0.x 기반)
- Spring MVC
- Spring Security
- Spring Data JPA
- Spring AI

Spring MVC / Spring Security / Spring Data JPA / Spring AI는 별도 버전을 명시하지
않고 Spring Boot 4.0.7이 관리하는 BOM(Bill of Materials) 버전을 그대로 따릅니다.

---

## Database

### PostgreSQL (서비스별 전용 DB)

- 사용자/인증 정보 (Auth DB)
- 재배/수확/사진 정보 (Cultivation DB)
- 센서 장치/목표 환경/버섯 참조 정보 (Sensor DB)
- 알림 이력 (Notification DB)
- 챗봇 대화/생육 분석/일일 피드백/인사이트 이력 (AI DB)

### Redis

- Refresh Token / 이메일 인증
- AI 응답·생육 분석·리포트·인사이트·버섯 가이드 캐시
- 목표 환경 범위 캐시
- 최신 센서 데이터

### InfluxDB

- 센서 시계열 데이터 저장

### Photo Storage (MinIO / Local)

- 사용자가 업로드한 생육 사진 저장. `storage_type`으로 MinIO/로컬을 구분해
  추상화합니다.

---

## AI

- Spring AI
- OpenAI API (또는 Gemini API)
- RAG (버섯 참조 데이터/인사이트 사례를 정확한 값 매칭으로 직접 조회해 LLM 컨텍스트로
  사용 — 버섯 종류가 5종 고정이고 검색 조건이 정확한 값/범위 필터라 별도의 임베딩
  모델이나 벡터 검색은 사용하지 않습니다)
- Vision 모델 (생육 사진 분석 - 균사 성장률/갓 크기/색상/병충해 판별)

---

## Messaging

### MQTT

센서 데이터 수집

### RabbitMQ

서비스 간 비동기 메시지 처리

---

## MSA

- Spring Cloud Gateway
- Eureka Server
- OpenFeign

Spring Cloud 2025.1.x (Oakwood) 릴리스 트레인 사용 — Spring Boot 4.0.x와 호환되는
버전입니다.

---

## Infrastructure

- Docker
- Kubernetes
- Nginx

---

## API

- REST API
- Swagger (OpenAPI 3)

---

## Build Tool

- Maven

---

## CI/CD

- GitHub
- GitHub Actions
- Docker
- Kubernetes

CI/CD Pipeline

```
GitHub Push
↓
GitHub Actions
↓
Maven Build
↓
Docker Image Build
↓
Docker Registry
↓
Kubernetes Rolling Update
```

---

## Monitoring (예정)

- OpenObserve
- Grafana
