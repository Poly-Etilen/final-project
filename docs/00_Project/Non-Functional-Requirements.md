# Non Functional Requirements

## Performance

- Dashboard 조회 응답시간 1초 이하
- AI 추천 응답시간 5초 이하
- MQTT 데이터 처리 지연 1초 이하

---

## Availability

- RabbitMQ Cluster
- Service Replication
- Rolling Update

---

## Scalability

서비스는 독립적으로 Scale Out 가능해야 한다.

- AI Service
- Collector Service
- Notification Service

---

## Security

- JWT Authentication
- OAuth2
- BCrypt
- HTTPS

---

## Monitoring

- Grafana
- InfluxDB
- Application Log

---

## Maintainability

- MSA Architecture
- Domain Driven Design
- Database Per Service