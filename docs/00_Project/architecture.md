# 시스템 아키텍처

## 서비스 구성

- API Gateway
- Auth Service
- User Service
- Cultivation Service
- AI Service
- Embedding Service
- Sensor Service
- Datasource Service
- Rule Engine Service
- Notification Service

---

## 데이터베이스

### PostgreSQL

- Auth
- User
- Cultivation
- Datasource

---

### Redis

- Refresh Token
- 이메일 인증
- AI 응답 캐시

---

### InfluxDB

센서 시계열 데이터 저장

- Temperature
- Humidity
- CO₂
- Light

---

### Elasticsearch

버섯 환경 임베딩 저장

---

## 메시지 브로커

### MQTT

센서 데이터 수집

### RabbitMQ

서비스 간 비동기 이벤트 전달

---

## AI

Spring AI

↓

Embedding Service

↓

Elasticsearch

↓

LLM

↓

환경 추천

↓

생육 분석

↓

AI 리포트