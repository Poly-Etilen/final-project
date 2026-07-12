# Storage Service API

## Dashboard 조회

GET /api/storage/dashboard/{workspaceId}

---

### Response

```json
{
    "temperature":24,

    "humidity":65,

    "co2":650,

    "ph":6.4
}
```

---

# 차트 조회

GET /api/storage/chart

### Query

```
workspaceId

type

start

end
```

---

# AI Report 조회

GET /api/storage/report/{workspaceId}

---

# 최근 이벤트

GET /api/storage/events/{workspaceId}