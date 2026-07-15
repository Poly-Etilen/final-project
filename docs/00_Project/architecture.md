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

### MinIO

사용자가 업로드한 생육 사진 저장

- Cultivation Service가 업로드
- AI Service가 Vision 분석을 위해 읽기 전용으로 조회

---

## 메시지 브로커

### MQTT

센서 데이터 수집

### RabbitMQ

서비스 간 비동기 이벤트 전달

---

## AI

### 환경 추천 / 챗봇 / 리포트

Spring AI

↓

Embedding Service

↓

Elasticsearch

↓

LLM

↓

환경 추천 / 챗봇 답변 / AI 리포트

---

### 생육 분석 (Vision)

사용자가 촬영한 사진 업로드 (Cultivation Service → MinIO)

↓

AI Service Vision 모델

↓

균사 성장률 / 갓 크기 / 색상 / 병충해 판별

↓

LLM (결과 해석 및 개선 방안 생성)