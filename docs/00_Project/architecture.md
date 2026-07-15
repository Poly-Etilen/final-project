# 시스템 아키텍처

## 서비스 구성

8개 서비스(API Gateway 포함)로 구성합니다. 강사 피드백("합칠 수 있는 건 합쳐라")에 따라
기존 9개 서비스에서 Auth+User, Collector+RuleEngine+Storage를 각각 하나로 통합했습니다.

- API Gateway
- Auth Service (기존 Auth+User 통합)
- Cultivation Service
- AI Service
- Embedding Service
- Rule Engine Service (기존 Collector+RuleEngine+Sensor/Storage 통합)
- Notification Service
- DatasourceGenerator (기존 Datasource Service 리네임)

---

## 데이터베이스

### PostgreSQL

- Auth (users 단일 테이블)
- Cultivation
- DatasourceGenerator

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