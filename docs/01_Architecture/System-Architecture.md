# System Architecture

## 서비스 구성

### API Gateway

모든 클라이언트 요청의 진입점이다.

역할

- Routing
- JWT 인증
- Load Balancing
- Logging

---

### Auth Service

인증 전용 서비스

기능

- 로그인
- JWT 발급
- Refresh Token
- OAuth2
- 이메일 인증

---

### User Service

사용자 정보를 관리한다.

기능

- 사용자 조회
- 회원정보 수정
- 프로필 조회

---

### Workspace Service

프로젝트의 핵심 서비스이다.

기능

- Workspace 생성
- Workspace 수정
- Device 등록
- Environment 저장
- AI 추천 요청

---

### AI Service

LLM을 이용하여 환경을 추천한다.

기능

- 자연어 처리
- RAG
- AI Report 생성

---

### Embedding Service

CSV 데이터를 임베딩하여 Elasticsearch에 저장한다.

---

### Datasource Service

CSV 데이터를 파싱하여 MQTT Broker로 Publish한다.

---

### Collector Service

MQTT 데이터를 수신한다.

---

### Rule Engine

센서 데이터를 분석한다.

- 임계치 검사
- 이벤트 생성
- RabbitMQ Publish

---

### Storage Service

RabbitMQ 데이터를 저장한다.

- InfluxDB 저장
- Dashboard 조회
- AI Report 집계

---

### Notification Service

이벤트 알림

- WebSocket
- Telegram
- Discord