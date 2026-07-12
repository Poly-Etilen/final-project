# MSA Service

| Service | DB | 역할 |
|----------|----|------|
| Gateway | X | Routing |
| Auth | PostgreSQL / Redis | 인증 |
| User | PostgreSQL | 사용자 |
| Workspace | PostgreSQL | Workspace |
| AI | ElasticSearch | 환경 추천 |
| Embedding | ElasticSearch | Vector 생성 |
| Datasource | X | CSV Parser |
| Collector | X | MQTT Consumer |
| Rule Engine | X | 임계치 검사 |
| Storage | InfluxDB | 시계열 저장 |
| Notification | Redis | 실시간 알림 |

---

## 서비스 통신

### REST

- Gateway → Auth
- Gateway → User
- Gateway → Workspace
- Workspace → AI

---

### MQTT

- Datasource → Collector

---

### RabbitMQ

- Rule Engine → Storage

---

### WebSocket

- Notification → Client

---

## Database Per Service

```text
Auth
 ├── PostgreSQL
 └── Redis

User
 └── PostgreSQL

Workspace
 └── PostgreSQL

AI
 └── ElasticSearch

Storage
 └── InfluxDB

Notification
 └── Redis
```

---

## 이벤트 기반 처리

동기 처리

- 로그인
- Workspace 생성
- AI 추천

비동기 처리

- 센서 데이터
- 알림
- AI Report
- 데이터 저장