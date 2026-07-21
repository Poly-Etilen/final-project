# 시스템 아키텍처

## 서비스 구성

9개 서비스(API Gateway 포함)로 구성합니다. 강사 피드백("합칠 수 있는 건 합쳐라")에 따라
Auth+User는 하나로 통합했습니다. Collector+RuleEngine+Sensor(Storage)는 한때 통합을 검토했지만,
저장·조회 책임의 크기와 변경 주기가 규칙 평가/제어 로직과 달라 Rule Engine Service(수신·평가·제어)와
Sensor Service(저장·조회)로 다시 분리했습니다.

- API Gateway
- Auth Service (기존 Auth+User 통합)
- Cultivation Service
- AI Service
- Embedding Service
- Rule Engine Service (MQTT 수신/Collector, 검증, 규칙 평가, 자동 제어, 센서 오류 감지)
- Sensor Service (측정값 저장/조회, 통계·차트, 주간 리포트 집계)
- Notification Service
- DatasourceGenerator (기존 Datasource Service 리네임)

Rule Engine Service와 Sensor Service는 RabbitMQ(EnvironmentMeasuredEvent)로만 연결되며
서로 직접 호출하지 않습니다.

---

## 데이터베이스

### PostgreSQL

- Auth (users 단일 테이블)
- Cultivation
- Notification (알림 이력)
- AI (챗봇 대화 이력, 생육 분석 이력, 일일 피드백 이력)

---

### Redis

- Refresh Token
- 이메일 인증
- AI 응답 캐시
- 목표 환경 범위 캐시 (Rule Engine Service)
- 최신 센서 데이터 (Sensor Service)

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

### 환경 추천 (Cultivation Service, AI 미사용)

Cultivation Service

↓

mushroom_reference 조회 (PostgreSQL)

↓

환경 추천

버섯 종류가 공공데이터 기준 5가지로 고정되어 있어 Vector Search/LLM 없이 Cultivation Service가
직접 조회합니다.

---

### 챗봇 / 리포트

Spring AI

↓

Embedding Service (챗봇 유사 사례 검색, 선택적)

↓

Elasticsearch

↓

LLM

↓

챗봇 답변 / AI 리포트

---

### 생육 분석 (Vision)

사용자가 촬영한 사진 업로드 (Cultivation Service → MinIO)

↓

AI Service Vision 모델

↓

균사 성장률 / 갓 크기 / 색상 / 병충해 판별

↓

LLM (결과 해석 및 개선 방안 생성)