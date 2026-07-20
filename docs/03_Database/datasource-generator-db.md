# DatasourceGenerator Database

## 개요

DatasourceGenerator는 별도의 Database(PostgreSQL)를 사용하지 않습니다.

센서 데이터 생성/발행에 필요한 최소한의 정보(`sensor_cache`)는 서비스 메모리(In-Memory)에서만
관리합니다. 실제 센서 데이터(측정값)도 이 서비스에는 저장하지 않으며, MQTT를 통해 Rule Engine
Service로 전달될 뿐입니다.

> ℹ️ **변경 이력**: 센서 "장치"의 등록/조회/삭제(CRUD)는 원래 이 DB의 `sensor` 테이블이
> 전담했지만, 센서가 항상 특정 재배에 종속되는 정보라는 점 때문에 소유권을 Cultivation
> Service(Cultivation DB)로 옮겼습니다. (자세한 내용은 [cultivation-db.md](./cultivation-db.md) 참고)

> ℹ️ **변경 이력**: 센서 등록 시 사용자가 입력하는 위치 정보(place/location)가 별도 엔티티
> 없이 센서 레코드에 직접 저장되는 방식으로 바뀌면서, 이 DB가 소유하던 `datasource` 테이블도
> 폐지되었습니다.

> ℹ️ **변경 이력**: 위 두 변경 이후 이 DB에는 `sensor_cache` 테이블 하나만 남았는데,
> device_eui/cultivationId/sensorType 세 값만 갖는 단순 조회용 캐시라 조인·트랜잭션이
> 필요 없었습니다. 이런 용도로 PostgreSQL(관계형 DB)을 유지하는 것이 과하다고 판단해,
> **DB 자체를 없애고 메모리(In-Memory, 예: ConcurrentHashMap)로 전환**했습니다.
> `device_eui`는 정수가 아니라 LoRaWAN 64비트 DevEUI를 16자리 hex 문자열로 표현한 값이라
> `Map` 키 타입도 `String`입니다(자세한 내용은 [cultivation-db.md](./cultivation-db.md) 참고).
> 이 문서는
> 과거 테이블 설계를 참고할 수 있도록 유지하되, 실제로는 더 이상 PostgreSQL을 사용하지
> 않습니다.

---

# sensor_cache (메모리 구조)

DatasourceGenerator가 "어떤 센서에 대해 MQTT 데이터를 생성/발행해야 하는지" 판단하기 위한
읽기 전용 캐시입니다. Cultivation Service의 `sensor` 테이블이 원본(source of truth)이며,
아래 두 가지 방식으로만 채워집니다.

- 평상시: Cultivation Service가 발행하는 `SensorRegisteredEvent`(Upsert)/`SensorDeletedEvent`(삭제) 구독
- 서비스 시작 시: Cultivation Service의 `GET /api/v1/sensors`를 OpenFeign으로 호출해 전체 목록을 한 번에 채움 (재시작하면 메모리가 비기 때문)

## 구조 (참고용, 실제 DDL 아님)

```
Map<String, SensorCacheEntry>

SensorCacheEntry
──────────────────────────────────────────────
    deviceEui       (Key, Cultivation Service의 sensor.device_eui와 동일)
    cultivationId
    sensorType
```

place/location/deviceModel/status 같은 상세 메타데이터는 갖지 않습니다. (필요하면 Cultivation
Service API를 조회)

---

# Status

sensor_cache는 상태(ONLINE/OFFLINE/ERROR 등)를 갖지 않습니다. 센서 상태는 Cultivation
Service의 `sensor.status`가 원본이며, Rule Engine Service가 발행하는 SensorErrorEvent도
Cultivation Service가 구독합니다. DatasourceGenerator는 상태 정보가 필요 없습니다(데이터
생성/발행 여부는 sensor_cache에 존재하는지 여부로만 판단).

---

# 데이터 흐름

## 평상시 (이벤트 기반)

센서 등록 (Cultivation Service)

↓

RabbitMQ Publish (SensorRegisteredEvent)

↓

DatasourceGenerator 구독 → 메모리 캐시(sensor_cache) Upsert

↓

MQTT Publish (sensor_cache에 있는 센서만 데이터 생성/발행)

↓

Rule Engine Service

삭제 시에는 SensorDeletedEvent를 구독해 메모리 캐시에서 해당 항목을 삭제합니다.

## 서비스 시작 시 (재구성)

DatasourceGenerator 시작

↓

Cultivation Service에 OpenFeign 호출 (`GET /api/v1/sensors`)

↓

메모리 캐시(sensor_cache) 일괄 채움

↓

이후 이벤트 기반으로 갱신

---

# 관계

```
Cultivation Service (sensor, 원본)

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

- DatasourceGenerator는 PostgreSQL을 포함해 어떤 영구 저장소도 사용하지 않습니다. 실제 센서 데이터(측정값)도 저장하지 않습니다.
- 센서 "장치" 메타데이터의 원본(source of truth)은 Cultivation Service의 `sensor` 테이블입니다. DatasourceGenerator는 시뮬레이션/발행에 필요한 최소 정보만 메모리에 보관합니다.
- 별도 `datasource` 테이블/엔티티는 존재하지 않습니다. 위치 정보(place/location)는 Cultivation Service의 `sensor` 테이블에만 저장되며, 이 서비스는 알 필요가 없습니다.
- 메모리 캐시이므로 서비스 재시작 시 데이터가 사라지지만, 시작 시점에 Cultivation Service를 OpenFeign으로 호출해 전체 목록을 다시 받아와 즉시 복구합니다.
- 평상시에는 이벤트만으로 동기화하므로, 이벤트가 유실되면(RabbitMQ 장애 등) 재시작 전까지는 캐시가 최신 상태를 반영하지 못할 수 있습니다. 주기적 재동기화(reconciliation) 배치는 추후 개발 예정입니다.
- 센서값은 MQTT를 통해 Rule Engine Service로 전달됩니다.
- 시계열 데이터는 Sensor Service(InfluxDB)에서 관리합니다. Rule Engine Service는 수신·검증·규칙평가만 담당하고 RabbitMQ로 Sensor Service에 저장을 위임합니다.
