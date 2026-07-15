# InfluxDB

## 개요

InfluxDB는 버섯 재배 과정에서 발생하는 센서 데이터를 시계열(Time Series) 형태로 저장합니다.

Rule Engine Service가 MQTT로 수신한 데이터를 InfluxDB에 직접 저장하며,
대시보드 차트, 통계 조회, AI 분석 데이터 생성에 활용됩니다.

---

# 사용 서비스

| Service | 역할 |
|----------|------|
| Rule Engine Service | 센서 데이터 저장 및 조회 (기존 Sensor Service 역할 포함) |
| AI Service | 주간/월간 데이터 분석 |

---

# Measurement

```
environment
```

---

# Tags

Tags는 조회 성능 향상을 위한 인덱스 역할을 합니다.

| Tag | 설명 |
|------|------|
| cultivationId | 재배 ID |
| sensorId | 센서 ID |

---

# Fields

실제 측정값입니다.

| Field | Type | 설명 |
|--------|------|------|
| temperature | Float | 온도(℃) |
| humidity | Float | 습도(%) |
| co2 | Integer | CO₂(ppm) |
| light | Integer | 조도(lux) |

---

# Timestamp

```
measuredAt
```

UTC 기준으로 저장합니다.

---

# 예시 데이터

```text
Measurement

environment

Tag

cultivationId = 3
sensorId = 1

Field

temperature = 22.4
humidity = 91.2
co2 = 850
light = 430

Timestamp

2026-08-15T12:30:00Z
```

---

# 데이터 저장 흐름

DatasourceGenerator

↓

MQTT

↓

Rule Engine Service (수신·규칙평가·저장을 하나의 서비스가 처리)

↓

InfluxDB

---

# 주요 조회

## 현재 시간 기준 최근 데이터

```
최근 1건 조회
```

---

## 최근 1시간

```
Temperature

Humidity

CO₂

Light
```

---

## 최근 24시간

환경 변화 그래프

---

## 최근 7일

환경 통계

---

## 최근 30일

환경 분석

---

# 집계(Aggregation)

주간 및 월간 AI 리포트를 위해 집계 데이터를 생성합니다.

주요 집계 항목

- 평균 온도
- 평균 습도
- 평균 CO₂
- 평균 조도
- 최고 온도
- 최저 온도
- 최고 습도
- 최저 습도

---

# Dashboard

실시간 카드

```
Redis 사용
```

차트

```
InfluxDB 사용
```

---

# Retention Policy

기본 정책

```
2년
```

2년 이후 데이터는 자동 삭제합니다.

※ 운영 환경에 따라 정책은 변경될 수 있습니다.

---

# Bucket

```
mushroom-monitoring
```

---

# 장애 대응

InfluxDB 장애 발생 시

- 실시간 데이터는 Redis에서 조회
- 차트 조회 불가
- 통계 집계 불가
- AI 리포트 생성 불가

---

# 고려 사항

- 센서 데이터는 수정하지 않습니다.
- 모든 데이터는 Append Only 방식으로 저장합니다.
- Redis는 현재 상태(Current State)를 관리합니다.
- InfluxDB는 과거 이력(Historical Data)을 관리합니다.

---

# 추후 개발 예정

- Downsampling
- Continuous Query
- Grafana 연동
- 장기 통계 저장