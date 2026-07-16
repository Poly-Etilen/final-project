# Rule Engine Service

## 역할

Rule Engine Service는 MQTT로 수신한 센서 데이터를 검증하고, 규칙(Rule) 기반으로 환경을 평가하여
자동 제어를 수행하는 서비스입니다.

> ℹ️ **변경 이력**: 한때 "Collector + Rule Engine + Sensor(Storage)"를 하나의 서비스로 통합하는
> 방안을 검토했었지만, 저장·조회 책임의 크기와 배포 주기가 규칙 평가/제어 로직과 달라 다시
> **Rule Engine Service(수신·평가·제어)** 와 **Sensor Service(저장·조회)** 로 분리했습니다.
> Rule Engine Service가 MQTT 수신(Collector 역할 포함)까지 담당하고, 저장은 RabbitMQ를 통해
> Sensor Service에 위임합니다.

> ℹ️ **변경 이력**: 처음에는 규칙 평가 때마다 Cultivation Service를 OpenFeign으로 매번 호출해
> 목표 환경 범위를 조회했지만, 센서 데이터가 수신될 때마다(수 초~수십 초 주기) 동기 호출이
> 발생해 Cultivation Service에 부하가 몰리고 장애 시 자동 제어 자체가 막히는 문제가 있었습니다.
> 이를 해결하기 위해 Redis에 목표 환경 범위를 캐싱하는 방식을 도입했습니다.

---

# 책임

- MQTT 센서 데이터 수신 (Collector)
- 센서 데이터 검증
- 환경 상태 분석 / 규칙(Rule) 평가
- 장치 자동 제어
- 목표 환경 범위 캐시 관리 (Redis)
- 센서 오류 / 연결 해제 감지
- 이벤트 발행 (저장용 EnvironmentMeasuredEvent, 제어 알림용 EnvironmentControlEvent, 오류용 SensorErrorEvent)

측정값 저장(Redis/InfluxDB)과 조회(현재값/통계/차트/리포트 API)는 Sensor Service의 책임입니다.
Rule Engine Service의 Redis는 측정값이 아닌 "목표 환경 범위 캐시" 전용입니다.

---

# 주요 기능

## 센서 데이터 수신 및 검증

MQTT Broker로부터 센서 데이터를 수신합니다.

수신 항목

- Temperature
- Humidity
- CO₂
- Light

---

## 목표 환경 범위 조회 (Redis 캐시 우선)

규칙 평가 전에 목표 환경 범위(min~max)를 조회합니다.

```
Redis 캐시 조회 (cultivation:{cultivationId}:range)

↓

Cache Hit → 즉시 사용

↓

Cache Miss → Cultivation Service OpenFeign 호출 → Redis에 캐시 저장 후 사용
```

캐시는 기본적으로 Cultivation Service가 환경을 저장/수정할 때 발행하는 EnvironmentRangeUpdatedEvent를
구독해 미리 채워둡니다(write-through). OpenFeign 호출은 캐시가 아직 없는 예외 상황(서비스 재시작 직후 등)에만
발생하는 fallback입니다.

---

## 규칙 평가 및 자동 제어

조회한 목표 환경 범위와 현재 센서 값을 비교하여 규칙을 평가하고, 필요 시 장치를 자동으로 제어합니다.

목표 환경은 단일값이 아닌 범위로 저장되어 있습니다. (예: 습도 85~95%)
현재값이 범위 안에 있으면 아무 것도 제어하지 않고, 범위를 벗어난 경우에만 장치를 동작시킵니다.
불필요하게 장치를 자주 켜고 끄는 것(채터링)을 막기 위한 히스테리시스 목적입니다.

제어 대상

- 가습기
- 환풍기
- 냉각기
- 히터
- LED 조명

판단 규칙

```
현재값 < min

↓

부족 방향 장치 ON (예: 습도 < min → 가습기 ON)
```

```
현재값 > max

↓

과다 방향 장치 ON (예: 습도 > max → 제습기 ON)
```

```
min ≤ 현재값 ≤ max

↓

제어하지 않음 (이미 켜져 있던 장치는 OFF)
```

예시

```
현재 습도 82% < 습도 min 85%

↓

가습기 ON
```

```
현재 습도 90% (min 85% ~ max 95% 범위 안)

↓

제어 없음
```

---

## 센서 데이터 전달 (저장 위임)

Rule Engine Service는 센서 데이터를 직접 저장하지 않습니다.

검증을 마친 센서 데이터를 RabbitMQ로 EnvironmentMeasuredEvent 발행 → Sensor Service가 구독하여
Redis(최신값)/InfluxDB(이력)에 저장합니다.

---

## 센서 오류 감지

수신 주기와 데이터 유효성을 검사하여 센서 오류(OFFLINE/ERROR)를 감지하고,
Cultivation Service의 센서 상태를 갱신하도록 이벤트를 발행합니다.

> ℹ️ **변경 이력**: 센서 장치 CRUD가 DatasourceGenerator에서 Cultivation Service로 이전되면서,
> SensorErrorEvent의 구독 주체도 DatasourceGenerator에서 Cultivation Service로 함께 옮겨졌습니다.

---

# API

Rule Engine Service는 REST API를 제공하지 않습니다.

MQTT 수신과 RabbitMQ 발행만으로 동작하는 이벤트/메시지 기반 서비스입니다.

현재 환경 조회, 통계, 차트, 주간/월간 리포트 데이터가 필요하면 [sensor-api.md](../02_API/sensor-api.md)를 참고하세요.

---

# Database

Rule Engine Service는 PostgreSQL/InfluxDB 같은 영구 저장소는 갖지 않으며,
Redis를 목표 환경 범위 캐시 용도로만 사용합니다.

## Redis

### Key

```
cultivation:{cultivationId}:range
```

### Value

```json
{
  "tempMin": 20.5,
  "tempMax": 23.5,
  "humidityMin": 85,
  "humidityMax": 95,
  "co2Min": 750,
  "co2Max": 850,
  "lightMin": 320,
  "lightMax": 380,
  "updatedAt": "2026-08-15T09:00:00"
}
```

### TTL

```
24시간
```

EnvironmentRangeUpdatedEvent를 받을 때마다 값을 갱신하며 TTL도 24시간으로 다시 연장합니다.
TTL은 이벤트 유실 등으로 캐시가 오래 방치되는 것을 막기 위한 안전장치이며,
정상 흐름에서는 이벤트로 계속 갱신되어 만료 전에 항상 최신값을 유지합니다.

캐시가 없을 때(TTL 만료, 서비스 재시작 직후 등)는 Cultivation Service를 OpenFeign으로 호출해
값을 조회한 뒤 Redis에 채워 넣습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Cultivation Service

- 재배별 목표 환경 범위(environment_setting의 min~max) 조회 (Redis 캐시 미스 시에만 호출하는 fallback)

---

## 호출받는 서비스

### MQTT Broker

- DatasourceGenerator가 발행한 센서 데이터 구독

---

# MQTT

## Subscribe Topic

```
sensor/+
```

수신 데이터

```json
{
  "sensorId": 1,
  "temperature": 22.4,
  "humidity": 88.1,
  "co2": 1050,
  "light": 420,
  "measuredAt": "2026-08-15T12:30:00"
}
```

---

# RabbitMQ

Rule Engine Service는 저장(Sensor Service)과 알림(Notification Service)처럼
분리된 다른 서비스로 데이터를 전달할 때, 그리고 Cultivation Service로부터 목표 환경 범위 변경을
전달받을 때 RabbitMQ를 사용합니다.

## Subscribe Event

### EnvironmentRangeUpdatedEvent

Cultivation Service가 environment_setting을 생성/수정할 때 발행합니다.

```json
{
  "cultivationId": 3,
  "tempMin": 20.5,
  "tempMax": 23.5,
  "humidityMin": 85,
  "humidityMax": 95,
  "co2Min": 750,
  "co2Max": 850,
  "lightMin": 320,
  "lightMax": 380,
  "updatedAt": "2026-08-15T09:00:00"
}
```

수신 시 Redis 캐시(`cultivation:{cultivationId}:range`)를 즉시 갱신합니다.
발행 서비스: Cultivation Service

---

## Publish Event

### EnvironmentMeasuredEvent

검증을 마친 센서 측정값을 저장용으로 발행합니다. (수신할 때마다 매번 발행)

```json
{
  "cultivationId": 3,
  "sensorId": 1,
  "temperature": 22.4,
  "humidity": 88.1,
  "co2": 1050,
  "light": 420,
  "measuredAt": "2026-08-15T12:30:00"
}
```

구독 서비스: Sensor Service

---

### EnvironmentControlEvent

자동 제어가 발생했을 때만 발행합니다. (범위 안이라 제어하지 않은 경우 발행하지 않음)

```json
{
  "cultivationId": 3,
  "device": "HUMIDIFIER",
  "action": "ON",
  "reason": "Humidity below humidity_min",
  "timestamp": "2026-08-15T12:31:00"
}
```

구독 서비스: Notification Service

---

### SensorErrorEvent

센서 오류/연결 해제가 감지되었을 때 발행합니다.

구독 서비스: Cultivation Service (sensor.status 갱신), Notification Service (알림)

---

# 자동 제어 예시

| 조건 | 제어 |
|------|------|
| Temperature > temp_max | 냉각팬 ON |
| Temperature < temp_min | 히터 ON |
| Humidity < humidity_min | 가습기 ON |
| Humidity > humidity_max | 제습기 ON |
| CO₂ > co2_max | 환풍기 ON |
| Light < light_min | LED ON |
| 모든 항목이 범위 안 | 제어 없음 |

---

# Sequence

## 센서 데이터 처리 및 자동 제어

DatasourceGenerator

↓

MQTT Publish

↓

MQTT Broker

↓

Rule Engine Service

├── 검증
├── 목표 환경 범위 조회 (Redis 캐시 우선, 미스 시 Cultivation Service OpenFeign fallback)
├── 규칙 평가 → 장치 자동 제어 (범위 벗어난 경우만)
└── RabbitMQ Publish

↓

EnvironmentMeasuredEvent → Sensor Service (Redis/InfluxDB 저장)

EnvironmentControlEvent (제어 발생 시만) → Notification Service

---

## 목표 환경 범위 캐시 갱신

Cultivation Service

↓

environment_setting 생성/수정

↓

RabbitMQ Publish (EnvironmentRangeUpdatedEvent)

↓

Rule Engine Service

↓

Redis 캐시 갱신 (cultivation:{cultivationId}:range)

---

# 예외 상황

- MQTT Broker 연결 실패
- 규칙 평가 실패
- 장치 제어 실패
- 잘못된 센서 데이터
- Redis 캐시 조회/저장 실패
- Cultivation Service 호출 실패 (Redis 캐시 미스 시 fallback 호출)
- EnvironmentRangeUpdatedEvent 구독 실패 (캐시가 최신 값으로 갱신되지 않음, TTL 만료 전까지 이전 값 사용)
- RabbitMQ 발행 실패

---

# 추후 개발 예정

- 사용자 정의 Rule 지원
- Rule 우선순위 설정
- Rule 활성화/비활성화
- Rule 시뮬레이션 기능
- AI 기반 Rule 자동 생성
- 이상 데이터 탐지 고도화
- 환경 예측
