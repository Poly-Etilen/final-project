# 🌱 EcoSphere

> AI 기반 스마트 식물 생육 환경 관리 플랫폼

EcoSphere는 IoT 센서와 AI(LLM), 시계열 데이터 분석 기술을 활용하여 식물의 생육 환경을 실시간으로 관리하고, AI가 최적의 환경을 추천하는 스마트 식물 관리 플랫폼입니다.

사용자는 Workspace를 생성하여 자신만의 식물을 관리할 수 있으며, Workspace를 다른 사용자와 공유하여 함께 관리할 수 있습니다.

---

# 프로젝트 목표

- 초보자도 AI의 도움을 받아 식물을 쉽게 관리할 수 있는 환경 제공
- MQTT 기반 실시간 센서 데이터 수집
- 시계열 데이터 분석을 통한 환경 모니터링
- AI 기반 식물 환경 추천 및 리포트 생성
- MSA 기반 확장 가능한 시스템 구축

---

# 주요 기능

- 사용자 인증(JWT, OAuth2)
- Workspace 생성 및 공유
- AI 기반 환경 추천
- MQTT 기반 센서 데이터 수집
- Rule Engine 기반 이상 상태 감지
- 실시간 Dashboard
- Telegram / Discord / WebSocket 알림
- AI 주간 / 월간 리포트

---

# 기술 스택

## Backend

- Java 21
- Spring Boot
- Spring Security
- Spring AI

## Database

- PostgreSQL
- InfluxDB
- Redis
- Elasticsearch(Vector Search)

## Messaging

- MQTT
- RabbitMQ

## Infra

- Docker
- Kubernetes
- GitHub Actions

---

# 서비스 구성

- API Gateway
- Auth Service
- User Service
- Workspace Service
- AI Service
- Notification Service
- Collector Service
- Rule Engine Service
- Storage Service
- Embedding Service
- Datasource Service

---

# 프로젝트 구조

```
docs/
src/
docker/
k8s/
```

---

# 문서

프로젝트 설계 문서는 docs 폴더에서 확인할 수 있습니다.

```
docs/
├── 00_Project
├── 01_Architecture
├── 02_Database
├── 03_API
├── 04_AI
├── 05_IoT
├── 06_Infra
└── 07_Frontend
```