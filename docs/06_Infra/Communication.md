# Service Communication

| From | To | 방식 |
|------|----|------|
| Gateway | Auth | REST |
| Gateway | User | REST |
| Gateway | Workspace | REST |
| Workspace | AI | REST |
| AI | Embedding | REST |
| Embedding | Elasticsearch | Client |
| Datasource | MQTT | MQTT Publish |
| Collector | MQTT | MQTT Subscribe |
| Collector | Rule Engine | REST (또는 내부 호출) |
| Rule Engine | RabbitMQ | Publish |
| RabbitMQ | Storage | Consume |
| Rule Engine | Notification | Event |
| Storage | AI | REST |
| Dashboard | Storage | REST |