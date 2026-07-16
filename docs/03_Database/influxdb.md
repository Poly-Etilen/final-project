# InfluxDB

## 개요

InfluxDB는 버섯 재배 과정에서 발생하는 센서 데이터를 시계열(Time Series) 형태로 저장합니다.

Rule Engine Service가 MQTT로 수신·검증한 데이터를 RabbitMQ(EnvironmentMeasuredEvent)로 전달하면,
Sensor Service가 이를 구독하여 InfluxDB에 저장합니다.
대시보드 차트, 통계 조회, AI 분석 데이터 생성에 활용됩니다.

센서가 1초 주기로 값을 보내더라도 InfluxDB에는 매초 기록하지 않고 **재배별로 10초 간격으로
스로틀링**하여 저장합니다. 재배실 환경(온도/습도/CO₂/조도)은 물리적으로 초 단위로 급변하지
않기 때문에 10초 해상도로도 이력/통계/AI 분석 품질에 문제가 없으며, 이를 통해 InfluxDB 저장
용량을 약 1/10로 줄일 수 있습니다. 실시간성이 중요한 "현재값" 조회는 Redis가 매초 갱신을
그대로 유지하므로 대시보드 체감 실시간성에는 영향이 없습니다.

---

# 사용 서비스

| Service | 역할 |
|----------|------|
| Sensor Service | 센서 데이터 저장 및 조회 |
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
| deviceEui | 센서 장치 ID |

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
deviceEui = 1

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

MQTT (센서 발행 주기: 1초)

↓

Rule Engine Service (수신·검증·규칙평가)

↓

RabbitMQ (EnvironmentMeasuredEvent, 매초 발행)

↓

Sensor Service

├── Redis 저장 (매초, 항상)
└── InfluxDB 저장 (재배별 10초 이상 경과했을 때만 — 스로틀링)

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
- Redis는 현재 상태(Current State)를 관리하며 매초 갱신됩니다.
- InfluxDB는 과거 이력(Historical Data)을 관리하며 10초 간격으로 스로틀링되어 기록됩니다.
- 스로틀링 기준(10초)은 초기값이며 재배/센서 타입별로 조정될 수 있습니다.
- 스로틀링은 Sensor Service가 재배별 마지막 기록 시각을 메모리에서 관리하며 적용합니다. (자세한 내용은 [sensor.md](../01_Domain/sensor.md) 참고)

---

# 추후 개발 예정

- Downsampling (10초로 스로틀링된 데이터를 더 오래된 구간에서는 1분/1시간 단위로 추가 압축)
- Continuous Query
- Grafana 연동
- 장기 통계 저장