# AI Service API

## 개요

Workspace Service에서만 호출한다.

외부 Client는 직접 접근하지 않는다.

---

# 환경 추천

POST /internal/ai/recommend

```json
{
    "prompt":"몬스테라 키우기 좋은 환경"
}
```

---

### Response

```json
{
    "temperature":24,

    "humidity":65,

    "co2":700,

    "ph":6.5
}
```

---

# AI Report 생성

POST /internal/ai/report

```json
{
    "workspaceId":1,

    "summaryPeriod":"WEEKLY",

    "sensorData":[]
}
```

---

### Response

```json
{
    "summary":"..."
}
```