### device_source
* 가상의 센서를 관리하는 테이블

```sql
CREATE TABLE device_source (
    device_source_id BIGSERIAL PRIMARY KEY,

    device_code VARCHAR(100) NOT NULL UNIQUE,

    device_name VARCHAR(100) NOT NULL,

    sensor_type VARCHAR(50) NOT NULL,

    mqtt_topic VARCHAR(255) NOT NULL,

    location VARCHAR(100),

    enabled BOOLEAN NOT NULL DEFAULT TRUE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### sensor_simulation_data
* CSV를 Import하여 저장하는 테이블
```sql
CREATE TABLE sensor_simulation_data (
    simulation_data_id BIGSERIAL PRIMARY KEY,

    device_source_id BIGINT NOT NULL,

    temperature DECIMAL(4,1),

    humidity DECIMAL(4,1),

    co2 INTEGER,

    ph DECIMAL(3,1),

    measured_at TIMESTAMP NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

ALTER TABLE sensor_simulation_data
    ADD CONSTRAINT fk_simulation_device
        FOREIGN KEY (device_source_id)
            REFERENCES device_source(device_source_id);
```

### 연관관계
* device_source ---- sensor_simulation_data

---
### flow
```text
CSV

↓

sensor_simulation_data

↓

Scheduler (1초마다)

↓

device_source 조회

↓

MQTT Publish

↓

Collector Service
```

### 데이터 예시
```json
{
  "deviceCode": "TEMP-001",
  "temperature": 24.8,
  "humidity": 63.5,
  "co2": 780,
  "ph": 6.4,
  "timestamp": "2026-07-13T10:30:00"
}
```