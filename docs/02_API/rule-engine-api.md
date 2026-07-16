# Rule Engine API

## 개요

Rule Engine Service는 REST API를 제공하지 않습니다.

MQTT 수신(Collector 역할 포함), 규칙 평가, 자동 제어, 센서 오류 감지를 담당하며,
MQTT Subscribe와 RabbitMQ Publish만으로 동작하는 이벤트/메시지 기반 서비스입니다.

> ℹ️ **변경 이력**: 한때 Sensor Service(측정값 저장·조회)와 통합하면서 `GET /sensors/current` 등의
> 조회 API를 이 문서에서 함께 제공했었지만, 두 서비스를 다시 분리하면서 조회 API는
> [sensor-api.md](./sensor-api.md)로 옮겼습니다.

센서 "장치" 자체의 등록/조회/삭제는 [cultivation-api.md](./cultivation-api.md)를,
측정값 현재값/통계/차트/리포트 조회는 [sensor-api.md](./sensor-api.md)를 참고하세요.
(DatasourceGenerator는 데이터 소스 관리와 MQTT 발행만 담당하며, 센서 장치 CRUD는 더 이상
담당하지 않습니다. 자세한 내용은 [datasource-generator-api.md](./datasource-generator-api.md) 참고)

---

# MQTT

Subscribe Topic

```
sensor/+
```

DatasourceGenerator가 발행한 센서 데이터를 구독하여 검증과 규칙 평가에 사용합니다.

---

# RabbitMQ

Publish

```
EnvironmentMeasuredEvent → Sensor Service (저장용, 수신할 때마다 매번 발행)
EnvironmentControlEvent → Notification Service (자동 제어가 발생했을 때만 발행)
SensorErrorEvent → Cultivation Service (sensor.status 갱신), Notification Service (센서 오류 감지 시 발행)
```

Subscribe

```
EnvironmentRangeUpdatedEvent (Cultivation Service 발행) → Redis 캐시(cultivation:{cultivationId}:range) 갱신
```

---

# Redis

목표 환경 범위 캐시 전용입니다. 측정값 저장은 Sensor Service의 Redis(별도 키 공간)를 사용합니다.

Key

```
cultivation:{cultivationId}:range
```

TTL

```
24시간 (EnvironmentRangeUpdatedEvent 수신 시마다 연장)
```

자세한 값 구조는 [rule-Engine.md](../01_Domain/rule-Engine.md)의 Redis 섹션을 참고하세요.

---

# OpenFeign

호출하는 서비스

```
Cultivation Service (Redis 캐시 미스 시에만 호출하는 fallback — 목표 환경 범위(min~max) 조회)
```

호출받는 서비스

```
없음 (REST API가 없어 다른 서비스가 직접 호출하지 않습니다)
```

---

# Error Code

Rule Engine Service의 내부 처리 오류는 로그/모니터링으로 관리하며, REST API가 없어 HTTP Error Code 체계를 사용하지 않습니다.

| 상황 | 설명 |
|------|------|
| MQTT Broker 연결 실패 | 센서 데이터 수신 불가 |
| 규칙 평가 실패 | 목표 환경 범위 조회 실패 등 |
| 장치 제어 실패 | 자동 제어 명령 전달 실패 |
| Redis 캐시 조회/저장 실패 | 목표 환경 범위 캐시 사용 불가, Cultivation Service fallback 호출 증가 |
| RabbitMQ 발행 실패 | Sensor Service/Notification Service 전달 실패 |
| RabbitMQ 구독 실패 | EnvironmentRangeUpdatedEvent를 못 받아 캐시가 최신화되지 않음 |
