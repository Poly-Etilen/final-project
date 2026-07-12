# Infrastructure

## 개요

EcoSphere는 MSA 기반으로 구성되며 각 서비스는 독립적으로 배포된다.

모든 서비스는 Kubernetes 환경에서 실행되며 API Gateway를 통해 외부 요청을 처리한다.

---

## 구성 요소

- API Gateway
- Auth Service
- User Service
- Workspace Service
- AI Service
- Embedding Service
- Datasource Service
- Collector Service
- Rule Engine Service
- Storage Service
- Notification Service

---

## 사용 기술

| 영역 | 기술 |
|------|------|
| API Gateway | Spring Cloud Gateway |
| Service Discovery | Eureka |
| Container | Docker |
| Orchestration | Kubernetes |
| CI/CD | GitHub Actions |
| Cache | Redis |
| Message Queue | RabbitMQ |
| Time Series DB | InfluxDB |
| Vector DB | Elasticsearch |
| RDB | PostgreSQL |