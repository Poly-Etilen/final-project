# 환경 자동 제어 시퀀스

## 개요

센서에서 수집한 환경 데이터를 기반으로 Rule Engine Service가 목표 환경 범위와 비교하여
자동 제어 여부를 판단합니다.

목표 환경은 단일값이 아닌 범위(min~max)로 저장되어 있습니다. 제어를 시작하는 기준은 이 범위의
경계(min/max)이지만, 제어를 멈추는 기준은 범위 경계가 아니라 범위의 중앙값(mid =
(min+max)/2)입니다. 즉 한 번 장치가 켜지면 값이 범위 안으로 돌아온 즉시가 아니라 중앙값에
도달할 때까지 계속 동작합니다. 불필요하게 장치를 자주 켜고 끄는 것(채터링)을 막기 위한
히스테리시스 목적입니다.

> ℹ️ **변경 이력**: 제어 정지 기준을 "범위 안으로 복귀"에서 "범위의 중앙값 도달"로 명확히
> 했습니다. 경계를 살짝 넘기자마자 바로 꺼지면 경계 부근에서 반복 On/Off가 발생할 수 있어
> 정지 기준을 중앙값으로 한 단계 더 밀었습니다. (자세한 내용은
> [rule-Engine.md](../01_Domain/rule-Engine.md) 참고)

Rule Engine Service는 MQTT 수신, 검증, 규칙 평가, 자동 제어까지 담당하고, 저장은 직접 하지 않습니다.
검증을 마친 데이터를 RabbitMQ(EnvironmentMeasuredEvent)로 발행하면 Sensor Service가 구독하여
Redis/InfluxDB에 저장합니다. 센서가 1초 주기로 값을 보내면 이 이벤트도 매초 발행되지만,
Sensor Service는 Redis는 매초 그대로 갱신하고 InfluxDB는 재배별 10초 간격으로 스로틀링하여
기록합니다. 규칙 평가/자동 제어는 원본(매초) 데이터를 그대로 사용하므로 저장 스로틀링과
무관하게 반응 속도가 유지됩니다. (자세한 내용은 [sensor-data.md](./sensor-data.md) 참고)

> ℹ️ **변경 이력**: 한때 Rule Engine과 저장(Sensor Service)을 하나의 서비스로 통합해 내부 로직으로
> 즉시 저장하는 방식을 검토했었지만, 책임 분리를 위해 다시 별도 서비스로 나누고 RabbitMQ로 연결했습니다.

목표 환경 범위는 Rule Engine Service의 Redis 캐시(`cultivation:{cultivationId}:range`)에서 먼저 조회합니다.
Sensor Service를 매번 동기 호출하지 않기 위한 것으로, 캐시가 없을 때만 Sensor Service를
OpenFeign으로 호출해 값을 채웁니다.

> ℹ️ **변경 이력**: 팀 회의 결과 `environment_setting` 테이블이 Cultivation Service에서
> Sensor Service로 이관되면서, 이 fallback 호출 대상과 `EnvironmentRangeUpdatedEvent` 발행
> 주체가 모두 Sensor Service로 바뀌었습니다.

제어가 필요한 경우 제어 명령을 생성하고, 사용자에게 알림을 전송합니다.

---

# Sequence

```text
Sensor

↓

DatasourceGenerator

↓

MQTT Broker

↓

Rule Engine Service

├── 목표 환경 범위 조회 (Redis 캐시 우선, 미스 시 Sensor Service OpenFeign fallback)
├── 범위 비교 (min/max)
└── 제어 여부 판단

↓

RabbitMQ

├── EnvironmentMeasuredEvent (매초) → Sensor Service (Redis는 매초, InfluxDB는 10초 스로틀링)
└── EnvironmentControlEvent (제어 발생 시만) → Notification Service

↓

WebSocket
Telegram
Discord
```

---

# 상세 과정

## 1. 센서 데이터 측정

센서가 다음 데이터를 측정합니다.

- Temperature
- Humidity
- CO₂
- Light

예시

```json
{
    "temperature":20.5,
    "humidity":82,
    "co2":980,
    "light":310
}
```

---

## 2. MQTT Publish

DatasourceGenerator가

MQTT Broker로 데이터를 Publish합니다.

Topic

```
sensor/{cultivationId}
```

---

## 3. Rule Engine Service 수신

Rule Engine Service가

MQTT를 Subscribe합니다.

↓

환경 데이터를 수신하고 검증합니다.

---

## 4. 목표 환경 범위 조회

Rule Engine Service는

Redis 캐시(`cultivation:{cultivationId}:range`)를 먼저 조회합니다.

```
Cache Hit → 즉시 사용

Cache Miss → Sensor Service OpenFeign 호출 → Redis에 캐시 저장 후 사용
```

캐시는 평상시 Sensor Service가 environment_setting을 저장/수정할 때 발행하는
EnvironmentRangeUpdatedEvent를 구독해 미리 채워져 있으므로, OpenFeign 호출은 캐시가 없는
예외적인 상황에만 발생합니다.

예시

```
Temperature

temp_min 20.5℃ ~ temp_max 23.5℃
```

```
Humidity

humidity_min 85% ~ humidity_max 95%
```

이 범위는 Sensor Service가 사용자의 단일 목표값(예: 온도 22℃)에 허용 오차를 적용해
저장해 둔 값입니다.

---

## 5. 범위 비교

현재값

↓

min ~ max 범위

↓

비교 수행

예시

```
현재 습도

82%
```

↓

```
범위

85% ~ 95%
```

↓

판정

```
82% < min(85%) → 범위 미달
```

---

## 6. 자동 제어 판단

Rule Engine Service

↓

규칙 확인

예시

```
Humidity < humidity_min

↓

Humidifier ON
```

또는

```
Temperature > temp_max

↓

Cooling Fan ON
```

장치가 원래 OFF 상태였고 현재값이 min ~ max 범위 안에 있으면 아무 것도 제어하지 않습니다.
장치가 이미 ON 상태라면, 범위 안에 들어왔더라도 중앙값(mid = (min+max)/2)에 도달하기 전까지는
계속 ON을 유지합니다. mid에 도달하면 그때 OFF로 전환합니다.

---

## 7. RabbitMQ Publish

Rule Engine Service는 두 종류의 이벤트를 발행합니다.

### EnvironmentMeasuredEvent (저장용, 매초 발행)

```json
{
    "cultivationId":3,
    "deviceEui":"24e124128c067999",
    "temperature":20.5,
    "humidity":82,
    "co2":980,
    "light":310,
    "measuredAt":"2026-08-15T12:30:00"
}
```

구독: Sensor Service — Redis는 받을 때마다 매번 갱신하고, InfluxDB는 재배별 10초 간격으로 스로틀링해서 기록합니다.

### EnvironmentControlEvent (제어 발생 시만 발행)

```json
{
    "cultivationId":3,
    "temperature":20.5,
    "humidity":82,
    "co2":980,
    "light":310,
    "action":"HUMIDIFIER_ON",
    "reason":"Humidity below humidity_min"
}
```

구독: Notification Service

범위 안에서 정상 상태로 유지되는 경우 EnvironmentControlEvent는 발행하지 않습니다.

---

## 8. Sensor Service 저장

RabbitMQ Subscribe (EnvironmentMeasuredEvent)

↓

Redis 저장 (최신값)

↓

InfluxDB 저장 (이력)

---

## 9. Notification Service

RabbitMQ Subscribe (EnvironmentControlEvent)

↓

사용자에게 알림 전송

예시

```
🍄 자동 제어

습도가 목표 범위(85~95%) 아래로 떨어져

가습기를 실행했습니다.
```

---

## 부가 흐름: 목표 환경 범위 캐시 갱신

사용자가 환경 설정을 저장/수정할 때마다 아래 흐름으로 Rule Engine Service의 캐시가 갱신됩니다.

```
Sensor Service

↓

environment_setting 생성/수정

↓

RabbitMQ Publish (EnvironmentRangeUpdatedEvent)

↓

Rule Engine Service

↓

Redis 캐시 갱신 (cultivation:{cultivationId}:range, TTL 24시간 연장)
```

---

# 사용 Database

## Redis

- 현재 환경 저장 (Sensor Service, 매초 갱신)
- 목표 환경 범위 캐시 (Rule Engine Service, TTL 24시간)

---

## InfluxDB

환경 이력 저장 (Sensor Service, 재배별 10초 간격 스로틀링)

---

# MQTT

Subscribe

```
sensor/{cultivationId}
```

---

# RabbitMQ

Publish

```
EnvironmentMeasuredEvent
EnvironmentControlEvent
```

Subscribe

- Sensor Service (EnvironmentMeasuredEvent)
- Notification Service (EnvironmentControlEvent)
- Rule Engine Service (EnvironmentRangeUpdatedEvent, Sensor Service가 발행)

---

# OpenFeign

```
Rule Engine Service

↓

Sensor Service (Redis 캐시 미스 시에만 호출하는 fallback — 목표 환경 범위 조회)
```

---

# 자동 제어 예시

| 조건 | 제어 시작 | 제어 종료 |
|------|-----------|-----------|
| Temperature > temp_max | 냉각팬 ON | temperature가 temp_mid 도달 시 OFF |
| Temperature < temp_min | 히터 ON | temperature가 temp_mid 도달 시 OFF |
| Humidity < humidity_min | 가습기 ON | humidity가 humidity_mid 도달 시 OFF |
| Humidity > humidity_max | 제습기 ON | humidity가 humidity_mid 도달 시 OFF |
| CO₂ > co2_max | 환풍기 ON | co2가 co2_mid 도달 시 OFF |
| Light < light_min | LED ON | light가 light_mid 도달 시 OFF |
| 모든 항목이 범위 안 (장치가 원래 OFF) | 제어 없음 | - |

---

# 예외 상황

- MQTT 연결 실패
- 규칙 평가 오류
- Redis 캐시 조회/저장 실패 (Rule Engine Service)
- 목표 환경 범위 조회 실패 (캐시 미스 시 Sensor Service fallback 호출 실패)
- RabbitMQ 발행 실패
- Sensor Service 저장 실패
- Notification 전송 실패

---

# 고려 사항

- 목표 환경은 단일값이 아닌 범위(min~max)로 저장되어 있어, 장치가 원래 OFF 상태였다면 범위 안에서는 장치를 켜지 않습니다.
- 장치가 이미 ON 상태라면 범위 안으로 복귀해도 즉시 끄지 않고, 범위의 중앙값(mid)에 도달할 때까지 계속 제어합니다. 이 제어를 위해서는 재배별/장치별 현재 ON/OFF 상태를 Rule Engine Service가 추적해야 합니다(구체적인 저장 방식은 추후 확정, [rule-Engine.md](../01_Domain/rule-Engine.md) 참고).
- Rule Engine Service는 저장을 직접 하지 않고 RabbitMQ로 Sensor Service에 위임합니다.
- Rule Engine Service와 Sensor Service는 RabbitMQ로만 연결되며 서로 직접 호출하지 않습니다.
- Notification Service는 EnvironmentControlEvent만 구독하며, 저장용 EnvironmentMeasuredEvent는 구독하지 않습니다.
- 목표 환경 범위는 Redis 캐시를 우선 사용해 Sensor Service에 매 센서 수신마다 동기 호출이 몰리는 것을 방지합니다. 캐시는 EnvironmentRangeUpdatedEvent로 write-through 갱신되며, TTL(24시간)은 이벤트 유실에 대비한 안전장치입니다.
- 규칙 평가는 매초 원본 데이터로 수행되어 자동 제어 반응 속도에는 영향이 없지만, Sensor Service의 InfluxDB 저장은 재배별 10초 간격으로 스로틀링됩니다. Redis "현재값"은 매초 갱신되어 대시보드 체감 실시간성은 유지됩니다.
