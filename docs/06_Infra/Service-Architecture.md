# Service Architecture

## 서비스 목록

### API Gateway

외부 요청 진입점

---

### Auth Service

- 로그인
- JWT
- OAuth
- 이메일 인증

---

### User Service

- 사용자 관리
- 내 정보 조회

---

### Workspace Service

- Workspace 관리
- Device 관리
- Member 관리

---

### AI Service

- 환경 추천
- AI Report 생성

---

### Embedding Service

- CSV Embedding
- Vector Search

---

### Datasource Service

- CSV Parsing
- MQTT Publish

---

### Collector Service

- MQTT Subscribe
- DTO 생성

---

### Rule Engine Service

- 임계치 검사
- Event 생성

---

### Storage Service

- InfluxDB 저장
- Dashboard API

---

### Notification Service

- WebSocket
- Telegram
- Discord