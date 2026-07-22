# DatasourceGenerator Database

## 개요

DatasourceGenerator는 별도의 Database(PostgreSQL)를 사용하지 않습니다. 센서 데이터
생성/발행에 필요한 최소한의 정보(`sensor_cache`)는 서비스 메모리(In-Memory)에서만
관리합니다. 실제 센서 데이터(측정값)도 이 서비스에는 저장하지 않으며, MQTT를 통해
Rule Engine Service로 전달될 뿐입니다.

---

# sensor_cache (메모리 구조)

DatasourceGenerator가 "어떤 센서에 대해 MQTT 데이터를 생성/발행해야 하는지" 판단하기
위한 읽기 전용 캐시입니다. Sensor Service의 `sensor` 테이블이 원본(source of
truth)이며, 아래 두 가지 방식으로만 채워집니다.

- 평상시: Sensor Service가 발행하는 `SensorRegisteredEvent`(Upsert)/`SensorDeletedEvent`(삭제) 구독
- 서비스 시작 시: Sensor Service의 `GET /api/v1/sensors`를 OpenFeign으로 호출해 전체 목록을 한 번에 채움 (재시작하면 메모리가 비기 때문)

## 구조 (참고용, 실제 DDL 아님)

```
Map<String, SensorCacheEntry>

SensorCacheEntry
──────────────────────────────────────────────
    deviceEui       (Key, Sensor Service의 sensor.device_eui와 동일)
    cultivationId
    sensorType
```

`location`/`locationDetail`/`deviceModel`/`status` 같은 상세 메타데이터는 갖지
않습니다. 필요하면 Sensor Service API를 조회합니다.

---

# Status

`sensor_cache`는 상태(ONLINE/OFFLINE/ERROR 등)를 갖지 않습니다. 센서 상태는 Sensor
Service의 `sensor.status`가 원본이며, Rule Engine Service가 발행하는
`SensorErrorEvent`도 Sensor Service가 구독합니다. DatasourceGenerator는 상태 정보가
필요 없습니다(데이터 생성/발행 여부는 `sensor_cache`에 존재하는지 여부로만 판단).

---

# 데이터 흐름

## 평상시 (이벤트 기반)

```
센서 등록 (Sensor Service)
↓
RabbitMQ Publish (SensorRegisteredEvent)
↓
DatasourceGenerator 구독 → 메모리 캐시(sensor_cache) Upsert
↓
MQTT Publish (sensor_cache에 있는 센서만 데이터 생성/발행)
↓
Rule Engine Service
```

삭제 시에는 `SensorDeletedEvent`를 구독해 메모리 캐시에서 해당 항목을 삭제합니다.

## 서비스 시작 시 (재구성)

```
DatasourceGenerator 시작
↓
Sensor Service에 OpenFeign 호출 (GET /api/v1/sensors)
↓
메모리 캐시(sensor_cache) 일괄 채움
↓
이후 이벤트 기반으로 갱신
```

---

# 관계

```
Sensor Service (sensor, 원본)
    │  SensorRegisteredEvent / SensorDeletedEvent (RabbitMQ)
    │  GET /api/v1/sensors (OpenFeign, 서비스 시작 시에만)
    ▼
DatasourceGenerator (sensor_cache, 메모리, 읽기 전용)
    │
    ▼
MQTT → Rule Engine Service
```

---

# 고려 사항

- DatasourceGenerator는 PostgreSQL을 포함해 어떤 영구 저장소도 사용하지 않습니다.
  실제 센서 데이터(측정값)도 저장하지 않습니다.
- 센서 "장치" 메타데이터의 원본(source of truth)은 Sensor Service의 `sensor`
  테이블입니다. DatasourceGenerator는 시뮬레이션/발행에 필요한 최소 정보만
  메모리에 보관합니다.
- 메모리 캐시이므로 서비스 재시작 시 데이터가 사라지지만, 시작 시점에 Sensor
  Service를 OpenFeign으로 호출해 전체 목록을 다시 받아와 즉시 복구합니다.
- 평상시에는 이벤트만으로 동기화하므로, 이벤트가 유실되면(RabbitMQ 장애 등)
  재시작 전까지는 캐시가 최신 상태를 반영하지 못할 수 있습니다. 주기적
  재동기화(reconciliation) 배치는 추후 개발 예정입니다.
- 시계열 데이터는 Sensor Service(InfluxDB)에서 관리합니다. Rule Engine Service는
  수신·검증·규칙평가만 담당하고 RabbitMQ로 Sensor Service에 저장을 위임합니다.
