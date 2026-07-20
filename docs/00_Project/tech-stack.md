# 기술 스택

## Backend

- Java 21
- Spring Boot 4.0.7 (Spring Framework 7.0.x 기반)
- Spring MVC
- Spring Security
- Spring Data JPA
- Spring AI

Spring MVC / Spring Security / Spring Data JPA / Spring AI는 별도 버전을 명시하지 않고
Spring Boot 4.0.7이 관리하는 BOM(Bill of Materials) 버전을 그대로 따릅니다.

---

## Database

### PostgreSQL

- 사용자 정보
- 재배 정보
- 센서 장치 정보
- 알림 이력
- 챗봇 대화 이력

### Redis

- Refresh Token
- 이메일 인증
- AI 응답 캐시

### InfluxDB

- 센서 시계열 데이터 저장

### Elasticsearch

- 버섯 재배 환경 임베딩(Vector Search)

### MinIO

- 사용자가 업로드한 생육 사진(이미지) 저장

---

## AI

- Spring AI
- OpenAI API (또는 Gemini API)
- RAG (Retrieval-Augmented Generation)
- Embedding
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

Spring Cloud 2025.1.x (Oakwood) 릴리스 트레인 사용 — Spring Boot 4.0.x와 호환되는 버전입니다.

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

---

## Monitoring (예정)

- OpenObserve
- Grafana