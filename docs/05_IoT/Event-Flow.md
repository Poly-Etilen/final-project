# Event Flow

## 정상 데이터

Datasource

↓

MQTT

↓

Collector

↓

Rule Engine

↓

RabbitMQ

↓

Storage

↓

InfluxDB

↓

Dashboard

---

## 이상 데이터

Datasource

↓

MQTT

↓

Collector

↓

Rule Engine

├── RabbitMQ → Storage → InfluxDB

└── Notification → WebSocket / Telegram / Discord