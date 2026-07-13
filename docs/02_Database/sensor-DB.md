### sensor_report
```sql
CREATE TABLE sensor_report (
    report_id BIGSERIAL PRIMARY KEY,

    workspace_id BIGINT NOT NULL,

    report_type VARCHAR(20) NOT NULL,

    summary TEXT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

* report_type : DAILY, WEEKLY, MONTHLY

### sensor_event_log
```sql
CREATE TABLE sensor_event_log (
    event_id BIGSERIAL PRIMARY KEY,

    workspace_id BIGINT NOT NULL,

    device_id BIGINT,

    event_type VARCHAR(30) NOT NULL,

    message TEXT,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```
* event_type : WARNING, ERROR, DEVICE_OFFLINE, HIGH_TEMP, LOW_HUMIDITY

---
# Redis
* 현재 센서 상태
* 대시보드 요약 정보
* 주간 AI 리포트
