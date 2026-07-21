# Sensor Service

## 역할

Sensor Service는 Rule Engine Service로부터 전달받은 센서 측정값을 저장하고,
현재 환경/통계/차트/주간 리포트 데이터를 조회할 수 있도록 제공하는 서비스입니다. 또한 센서
"장치" 메타데이터(sensor)와 목표 환경(위험 한계값, environment_setting)을 PostgreSQL에
소유·관리합니다.

> ℹ️ **변경 이력**: 팀 회의 결과, `sensor`/`environment_setting` 두 테이블과 관련 API가
> Cultivation Service에서 Sensor Service로 완전히 이관되었습니다. Sensor Service가 이미
> 센서 측정값(InfluxDB/Redis)과 통계/차트/리포트를 전담하고 있어, 센서 장치 메타데이터와
> 목표 환경까지 함께 소유하는 것이 "database per service" 원칙에 더 맞는다고 판단했습니다.
> 이에 따라 Sensor Service는 이번에 처음으로 PostgreSQL을 갖게 되었습니다. (자세한 내용은
> [sensor-db.md](../03_Database/sensor-db.md), [cultivation.md](./cultivation.md),
> [README.md](../README.md)의 결정 사항 #24 참고)

> ℹ️ **변경 이력**: 한때 Rule Engine Service와 하나로 통합하는 방안을 검토했었지만,
> 저장·조회 책임(Redis/InfluxDB, 통계·차트 API, 주간 리포트 집계)의 크기와 변경 주기가
> 규칙 평가/자동 제어 로직과 달라 다시 별도 서비스로 분리했습니다.

> ℹ️ **변경 이력**: 월간 리포트를 폐기했습니다. 버섯 재배 기간이 한 달을 넘지 않아 "월간"
> 단위 자체가 의미가 없다고 판단했습니다. 이와 함께 리포트 생성 방식도 "사용자가 요청할 때
> 그때 생성"(pull)에서 **Weekly Scheduler가 주기적으로 먼저 집계해 AI Service에 전달하고
> AI Service가 리포트를 미리 만들어 두는 방식**(push)으로 정리했습니다. 이전 문서에는 두
> 방식이 섞여서(Scheduler 존재 + 동시에 요청 시 동기 생성) 서로 모순되게 적혀 있었습니다.
> (자세한 내용은 [ai.md](./ai.md), [ai-report.md](../04_sequence/ai-report.md) 참고)
> Rule Engine Service와는 RabbitMQ(EnvironmentMeasuredEvent)로만 연결되며, 직접 호출하지 않습니다.

> ℹ️ **변경 이력**: 센서가 1초 주기로 값을 보내는 경우, EnvironmentMeasuredEvent도 매초 발행되어
> InfluxDB에 그대로 다 기록하면 재배 1건 기준 한 달에 수백 MB, 2년 누적 시 상당한 용량이
> 필요해집니다. 반면 재배실 환경(온도/습도/CO₂/조도)은 물리적으로 초 단위로 급변하지 않으므로,
> **Redis(실시간 대시보드용)는 매초 그대로 갱신하고, InfluxDB(이력 저장용)만 10초 간격으로
> 스로틀링**하는 방식을 도입했습니다. 자동 제어 반응 속도(Rule Engine Service)에는 영향이 없습니다.

---

# 책임

- 센서 데이터 저장 (Redis 최신값 매초, InfluxDB 이력은 10초 간격 스로틀링)
- 실시간 환경 조회
- 환경 통계 / 차트 데이터 제공
- 주간 데이터 집계 및 AI Service 전달 (Scheduler, push)
- 센서 장치 등록/조회/삭제 (device CRUD)
- 목표 환경(위험 한계값) 저장/조회 (environment_setting)
- 환경 평균 조회 (단건 + 배치, AI Service 인사이트/일일피드백용)

센서 수신, 검증, 규칙 평가, 자동 제어는 Rule Engine Service의 책임입니다.
Rule Engine Service는 매초 EnvironmentMeasuredEvent를 발행하며, 저장 빈도 조절(스로틀링)은
전적으로 Sensor Service의 책임입니다.

---

# 주요 기능

## 센서 데이터 저장 (Redis 매초 / InfluxDB 10초 스로틀링)

Rule Engine Service가 RabbitMQ로 발행한 EnvironmentMeasuredEvent를 구독하여 저장합니다.
이벤트는 매초 들어오지만, 두 저장소를 다른 주기로 처리합니다.

1. **Redis** — 이벤트를 받을 때마다 매번 최신 데이터로 덮어씁니다. (실시간 대시보드용, 매초 갱신)
2. **InfluxDB** — 재배별로 마지막 기록 시각을 확인하여, 10초 이상 지났을 때만 이번 값을 기록합니다.
   10초가 지나지 않았다면 이번 이벤트는 InfluxDB에는 기록하지 않고 건너뜁니다. (이력 저장용, 10초 간격)

```
EnvironmentMeasuredEvent 수신

↓

Redis 저장 (항상)

↓

마지막 InfluxDB 기록 시각 확인 (재배별)

↓

10초 이상 경과? ─ No → InfluxDB 저장 건너뜀
              │
              Yes
              ↓
         InfluxDB 저장 + 마지막 기록 시각 갱신
```

마지막 InfluxDB 기록 시각은 재배별로 애플리케이션 메모리에 관리합니다.
서비스 재시작 시 초기화되지만, 이 경우 최초 1회 정도만 스로틀링 없이 기록되는 정도라 문제되지 않습니다.

10초라는 값은 초기 기준이며, 필요 시 재배/센서 타입별로 다르게 조정할 수 있습니다.

---

## 현재 환경 조회

Redis에 저장된 최신 센서 데이터를 조회합니다. 실시간 대시보드는 이 데이터를 사용합니다.

---

## 환경 통계 / 차트 조회

InfluxDB에서 기간별 통계와 차트 데이터를 조회합니다.

- 평균/최대/최소 온도·습도·CO₂·조도
- 시간별 변화 추이
- 목표 환경 범위 대비 유지율 (이 서비스가 소유한 environment_setting 참고)

---

## 주간 데이터 집계

Weekly Scheduler가 InfluxDB의 환경 데이터를 집계해 AI Service에 전달하고, AI Service가 그
자리에서 리포트를 생성하도록 트리거합니다. 사용자가 요청하는 시점이 아니라 Sensor Service가
먼저 능동적으로(push) 집계 데이터를 만들어 전달합니다.

---

## 센서 장치 등록/조회/삭제

재배에 연결되는 센서 "장치"를 관리합니다. (센서가 측정한 값 자체는 다루지 않습니다. 값은 위
"센서 데이터 저장"이 Redis/InfluxDB에 별도로 관리합니다.)

등록 정보

- device_eui (장치 고유 식별자, PK)
- place (설치 장소)
- location (세부 위치)
- device_model (장치 모델명)
- sensor_type (센서 종류)

개별 등록(재배 생성 이후 추가)과 배치 등록(재배 생성과 동시에, Cultivation Service가 호출)
두 가지 경로가 있습니다.

- **개별 등록**: 사용자가 재배 상세 화면 등에서 직접 요청합니다. 요청자가 해당 cultivation의
  소유자인지 Cultivation Service의 `GET /api/v1/cultivations/{cultivationId}/owner`를
  OpenFeign으로 호출해 확인한 뒤 등록합니다.
- **배치 등록**: Cultivation Service가 재배 생성 시 자기 자신의 이름으로 호출합니다. 호출자가
  이미 Cultivation Service 자신이므로 별도 소유권 확인을 하지 않습니다.

> ℹ️ **변경 이력**: 팀 회의 결과 `sensor` 테이블이 Cultivation Service에서 이관되면서, 등록
> 요청의 소유권을 더 이상 로컬 DB로 검증할 수 없게 되었습니다(`cultivation.user_id`에 직접
> 접근 불가). 이를 해결하기 위해 Cultivation Service에 내부용 소유권 확인 API를 신설했고,
> Sensor Service는 개별 등록/수정/삭제 시 이를 호출합니다. 배치 등록만 예외입니다. (자세한
> 내용은 [sensor-db.md](../03_Database/sensor-db.md) 참고)

등록/삭제 시 `SensorRegisteredEvent`/`SensorDeletedEvent`를 발행합니다. DatasourceGenerator가
이를 구독해 "어떤 센서에 대해 데이터를 시뮬레이션/발행할지" 판단하는 데 사용합니다.

센서 상태(ONLINE/OFFLINE/ERROR/MAINTENANCE)는 사용자가 직접 수정하지 않으며, Rule Engine
Service가 발행하는 `SensorErrorEvent`를 구독해 자동으로 갱신합니다.

재배가 삭제되면 Cultivation Service가 발행하는 `CultivationDeletedEvent`를 구독해 해당
cultivation_id의 sensor/environment_setting 행을 정리합니다(보상 삭제). 서로 다른 DB라서
`ON DELETE CASCADE`를 쓸 수 없기 때문입니다.

---

## 환경 설정 저장/조회

사용자가 추천 환경(mushroom_reference, Cultivation Service 소유)을 참고하여 실제 자동 제어
기준값(위험 한계값)을 저장합니다. 4개 항목(온도/습도/CO₂/조도)을 한 번에 저장할 수도 있고,
일부 항목만 선택적으로 수정할 수도 있습니다.

API로 주고받는 목표값은 단일값(예: 온도 22℃)이지만, 저장 시 허용 오차를 적용한 범위(예:
20.5~23.5℃)로 변환해 `environment_setting`에 항목별로 새 행을 INSERT합니다(UPDATE 없음).
Rule Engine Service가 값이 범위를 벗어날 때만 장치를 제어하도록 하여 불필요한 On/Off를
줄이기 위함입니다. 조회 시에는 항목별 최신 행의 범위에서 중간값 `(min+max)/2`를 계산해 단일값
으로 반환합니다. (자세한 변환 기준과 이력 설계는 [sensor-db.md](../03_Database/sensor-db.md) 참고)

쓰기 요청 시에는 위 센서 등록과 동일하게 Cultivation Service를 호출해 소유권을 확인합니다.

저장에 성공하면 재배의 4개 항목 전체(방금 수정한 것 + 기존 최신 값)를 모아 RabbitMQ로
`EnvironmentRangeUpdatedEvent`를 발행합니다(Rule Engine Service의 Redis 캐시는 항상 4개
항목 전체를 갖고 있어야 하므로, 이벤트 자체의 형태는 4개 항목을 모두 담습니다).

> ℹ️ **변경 이력**: "환경 설정 저장/조회"는 원래 Cultivation Service의 책임이었으나,
> `environment_setting` 테이블과 함께 Sensor Service로 완전히 이관되었습니다. 설계(단일값
> ↔ 범위 변환, INSERT-only 이력)는 그대로 유지했습니다.

---

## 환경 평균 조회 (단건 + 배치)

AI Service가 "인사이트"/"일일 피드백" 기능에서 사용할 환경 평균값(기간 가중 평균)을 제공합니다.

- **단건 조회**: 특정 재배 하나의 환경 평균을 조회합니다. 재배가 아직 `RUNNING` 상태여도
  (harvest가 아직 없어도) 호출할 수 있습니다 — 진행 중인 재배의 "지금까지의" 평균을 조회하는
  용도이기 때문입니다.
- **배치 조회**: 여러 cultivationId를 한 번에 보내 각각의 환경 평균을 한 번의 호출로 받아올 수
  있습니다. AI Service의 인사이트 배치 스케줄러가 미임베딩 수확 건마다 개별 호출하지 않고
  효율적으로 조회하기 위해 사용합니다.

두 API 모두 `environment_setting` 이력 전체를 기간 가중 평균(각 설정값이 적용되었던 기간의
길이로 가중)하여 조회 시점에 계산하며, 별도 컬럼으로 저장하지 않습니다.

> ℹ️ **변경 이력**: 이 기능은 원래 Cultivation Service가 `GET
> /api/v1/cultivations/{cultivationId}/environment-average`(단건)와 `GET
> /api/v1/harvests/unembedded` 응답의 평균 필드(배치성)로 제공했습니다.
> `environment_setting`이 이관되면서 단건 조회는 그대로 이전하고, 배치 조회는 신규
> 엔드포인트(`POST /api/v1/sensors/environment-averages`)로 새로 만들었습니다. AI Service의
> 인사이트 배치 스케줄러는 이제 Cultivation Service(미임베딩 목록)와 Sensor Service(환경
> 평균 일괄 조회)를 각각 호출해 조합합니다. (자세한 내용은 [ai.md](./ai.md),
> [insight.md](../04_sequence/insight.md) 참고)

---

## 환경 변경 이력 조회

특정 재배의 `environment_setting` 변경 이력(항목별 변경 시각, min/max)을 조회합니다. AI
Service의 "일일 피드백" 기능이 "사용자가 언제 몇 도에서 몇 도로 바꿨는지"를 파악하는 데
사용합니다.

> ℹ️ **변경 이력**: 원래 Cultivation Service가 제공하던 이 조회 기능이 `environment_setting`과
> 함께 이관되었습니다. (자세한 내용은 [daily-feedback.md](../04_sequence/daily-feedback.md) 참고)

---

# API

## 현재 환경 조회

GET /sensors/current

---

## 환경 통계 조회

GET /sensors/statistics

---

## 차트 데이터 조회

GET /sensors/chart

---

## 주간 데이터 조회

GET /sensors/report/weekly

---

## 센서 등록 (개별)

POST /sensors/cultivations/{cultivationId}

재배 생성 이후 센서를 추가로 등록할 때 사용합니다. Cultivation Service로 소유권을 확인합니다.

---

## 센서 일괄 등록 (내부용)

POST /sensors/cultivations/{cultivationId}/batch

Cultivation Service가 재배 생성 시 `devices`와 함께 등록하기 위해서만 호출하는 내부용
엔드포인트입니다. 소유권 확인을 하지 않습니다(호출자가 이미 Cultivation Service 자신).

---

## 센서 목록 조회

GET /sensors/cultivations/{cultivationId}

---

## 센서 상세 조회

GET /sensors/cultivations/{cultivationId}/{deviceEui}

---

## 센서 삭제

DELETE /sensors/cultivations/{cultivationId}/{deviceEui}

---

## 환경 설정 저장

PATCH /sensors/cultivations/{cultivationId}/environment

---

## 환경 설정 조회

GET /sensors/cultivations/{cultivationId}/environment

---

## 재배 환경 평균 조회 (내부용)

GET /sensors/cultivations/{cultivationId}/environment-average

AI Service가 "인사이트" 조회 시점에 현재 재배 조건을 파악하기 위해 호출합니다.

---

## 재배 환경 평균 일괄 조회 (내부용)

POST /sensors/environment-averages

AI Service의 인사이트 배치 스케줄러가 여러 cultivationId의 환경 평균을 한 번에 조회하기
위해 호출합니다.

---

## 환경 변경 이력 조회 (내부용)

GET /sensors/cultivations/{cultivationId}/environment-history

AI Service의 "일일 피드백" 기능이 환경 변경 이력을 조회하기 위해 호출합니다.

---

## 전체 센서 목록 조회 (내부용)

GET /api/v1/sensors

특정 재배로 한정하지 않고, 전체 재배의 센서 목록을 조회합니다. 사용자가 아닌
**DatasourceGenerator가 서비스 재시작 시 캐시(In-Memory)를 재구성하기 위해서만 호출**하는
내부용 엔드포인트입니다. (다른 센서 API처럼 `/cultivations/{cultivationId}` 하위 경로가
아닌 것에 유의)

> ℹ️ **변경 이력**: 이 엔드포인트는 원래 Cultivation Service가 제공했으나, `sensor` 테이블과
> 함께 Sensor Service로 이관되었습니다. (자세한 내용은
> [datasource-generator.md](./datasource-generator.md) 참고)

---

센서 데이터 수신, 규칙 평가, 자동 제어는 이 서비스가 아닌 Rule Engine Service가 담당하며 REST API로 노출하지 않습니다.

에러 코드 전체 목록은 [sensor-api.md](../02_API/sensor-api.md)를 참고하세요.

---

# Database

## PostgreSQL

`sensor`(센서 장치 메타데이터), `environment_setting`(목표 환경/위험 한계값 이력)을
소유합니다. Sensor Service가 PostgreSQL을 갖는 것은 이번이 처음입니다.

> ℹ️ **변경 이력**: 두 테이블 모두 Cultivation Service의 DB에서 이관되었습니다. `cultivation_id`를
> 갖지만 DB가 분리되어 있어 DB 레벨 외래키(FK)는 걸지 않습니다(순수 값 참조, `cultivation.user_id`
> 무-FK 참조와 같은 패턴). 자세한 테이블 구조/DDL/인덱스/설계 근거는
> [sensor-db.md](../03_Database/sensor-db.md)를 참고하세요.

---

## InfluxDB

### Measurement

```
environment
```

### Tag

- cultivationId
- deviceEui

### Field

- temperature
- humidity
- co2
- light

---

## Redis

최신 센서 데이터를 저장합니다.

### Key

```
cultivation:{cultivationId}:current
```

### Value

```json
{
  "temperature": 22.5,
  "humidity": 91.2,
  "co2": 820,
  "light": 430,
  "updatedAt": "2026-08-15T10:20:30"
}
```

TTL은 설정하지 않으며 EnvironmentMeasuredEvent를 받을 때마다(매초) 항상 최신 데이터로 덮어씁니다.
InfluxDB와 달리 스로틀링을 적용하지 않습니다 — Redis 덮어쓰기는 값이 늘어나는 게 아니라 항상 1건만
유지되므로 매초 갱신해도 저장 용량에 영향이 없습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### AI Service

- Weekly Scheduler가 집계한 주간 통계를 전달해 리포트 생성을 트리거 (push, OpenFeign)

### Cultivation Service

- 센서/환경 쓰기 요청(등록·수정·삭제) 처리 전 소유권 확인을 위해
  `GET /api/v1/cultivations/{cultivationId}/owner` 호출 (배치 등록은 예외 — 호출하지 않음)

> ℹ️ **변경 이력**: `sensor`/`environment_setting`이 이관되면서 Sensor Service가 더 이상
> `cultivation.user_id`에 직접 접근할 수 없게 되어 신설한 호출입니다.

---

## 호출받는 서비스

### API Gateway

- 현재 환경 / 통계 / 차트 조회 / 센서 CRUD / 환경 설정 CRUD

### AI Service

- 센서 데이터 / 통계 조회
- "인사이트" 기능을 위해 환경 평균 단건/배치 조회 (`GET /sensors/cultivations/{cultivationId}/environment-average`, `POST /sensors/environment-averages`)
- "일일 피드백" 기능을 위해 환경 변경 이력 조회 (`GET /sensors/cultivations/{cultivationId}/environment-history`)

### Cultivation Service

- 재배 생성 시 센서 일괄 등록을 위해 `POST /sensors/cultivations/{cultivationId}/batch` 호출
  (실패 시 Cultivation Service가 방금 생성한 cultivation을 보상 삭제)

### Rule Engine Service

- 목표 환경 범위(environment_setting의 min~max) 조회 (Rule Engine Service의 Redis 캐시가
  없을 때만 호출되는 fallback)

### DatasourceGenerator

- 서비스 재시작 시 `GET /api/v1/sensors`로 전체 센서 목록을 조회합니다. DatasourceGenerator는
  sensor_cache를 메모리(In-Memory)에만 보관하므로, 재시작하면 캐시가 비게 되어 이 방식으로
  복구합니다. (평상시 센서 등록/삭제는 이벤트로만 전달되며, 이때는 호출되지 않습니다.)

> ℹ️ **변경 이력**: Rule Engine Service의 fallback 조회와 DatasourceGenerator의 재시작 시
> 조회는 원래 Cultivation Service를 호출했으나, `sensor`/`environment_setting`이 이관되면서
> Sensor Service를 호출하도록 바뀌었습니다. (자세한 내용은 [rule-Engine.md](./rule-Engine.md),
> [datasource-generator.md](./datasource-generator.md) 참고)

Rule Engine Service와 EnvironmentMeasuredEvent 관련해서는 REST/OpenFeign으로 직접 통신하지
않으며, RabbitMQ 이벤트 구독으로만 연결됩니다. (단, 위 목표 환경 범위 fallback 조회는 예외적으로
OpenFeign을 사용합니다.)

---

# RabbitMQ

## Publish Event

> ℹ️ **변경 이력**: 아래 3개 이벤트는 `sensor`/`environment_setting`이 Cultivation Service에서
> 이관되면서 발행 주체도 함께 옮겨왔습니다. 원래 이 문서에는 "발행하는 이벤트는 없습니다"라고
> 적혀 있었습니다.

### SensorRegisteredEvent

센서를 등록할 때 발행합니다(개별/배치 등록 모두, 배치는 디바이스별로 각각 발행).

```json
{
    "deviceEui": "24e124128c067999",
    "cultivationId": 3,
    "sensorType": "TEMPERATURE",
    "registeredAt": "2026-08-15T09:00:00"
}
```

구독 서비스: DatasourceGenerator (sensor_cache 반영, 시뮬레이션 데이터 생성 대상 목록 갱신용)

---

### SensorDeletedEvent

센서를 삭제할 때 발행합니다.

```json
{
    "deviceEui": "24e124128c067999",
    "cultivationId": 3,
    "deletedAt": "2026-08-15T09:00:00"
}
```

구독 서비스: DatasourceGenerator (sensor_cache에서 제거)

---

### EnvironmentRangeUpdatedEvent

environment_setting을 생성/수정(저장)할 때 발행합니다. 단일 목표값을 범위(min~max)로 변환한 값을 담습니다.

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

구독 서비스: Rule Engine Service (Redis 캐시 write-through 갱신용), Cultivation Service
(cultivation 상태가 `CREATED`이면 `RUNNING`으로 전환 — 환경 저장과 재배 시작이 더 이상 하나의
로컬 트랜잭션이 아니라 이 이벤트로 연결됩니다)

---

## Subscribe Event

### EnvironmentMeasuredEvent

Rule Engine Service가 검증을 마친 센서 측정값을 매초 발행하면 구독합니다.
Redis에는 매번 저장하고, InfluxDB에는 재배별 10초 스로틀링을 적용해 저장합니다.

```json
{
  "cultivationId": 3,
  "deviceEui": "24e124128c067999",
  "temperature": 22.4,
  "humidity": 88.1,
  "co2": 1050,
  "light": 420,
  "measuredAt": "2026-08-15T12:30:00"
}
```

---

### SensorErrorEvent

Rule Engine Service가 센서 오류/연결 해제를 감지하면 발행합니다.

구독 시 sensor 테이블의 status를 갱신합니다.

```
ONLINE → OFFLINE / ERROR
```

> ℹ️ **변경 이력**: 이 이벤트의 구독 주체는 원래 Cultivation Service였으나, `sensor` 테이블
> 소유권이 Sensor Service로 옮겨지며 함께 이전했습니다.

---

### CultivationDeletedEvent

Cultivation Service가 재배를 삭제할 때 발행합니다.

```json
{
    "cultivationId": 3,
    "deletedAt": "2026-08-15T09:00:00"
}
```

구독 시 해당 cultivation_id의 sensor/environment_setting 행을 정리합니다(보상 삭제). DB가
분리되어 있어 `ON DELETE CASCADE`를 쓸 수 없기 때문에 이벤트로 대체합니다.

---

# Scheduler

## Weekly Scheduler

매주 InfluxDB 데이터를 집계하여 AI Service로 전달합니다(OpenFeign). 사용자 요청을 기다리지
않고 Scheduler가 먼저 집계·전달하며, AI Service는 이를 받아 리포트를 생성하고
`WeeklyReportCompletedEvent`를 발행합니다. (자세한 내용은 [ai.md](./ai.md) 참고)

---

# Sequence

## 센서 데이터 저장 및 조회

Rule Engine Service

↓

RabbitMQ Publish (EnvironmentMeasuredEvent, 매초)

↓

Sensor Service

├── Redis 저장 (매초, 항상)
└── InfluxDB 저장 (10초 이상 경과했을 때만)

↓

Dashboard

↓

현재 상태 조회 (Redis, 매초 최신) / 차트 조회 (InfluxDB, 10초 간격 이력)

---

## 센서 일괄 등록 (재배 생성 시)

Cultivation Service

↓

Cultivation 생성

↓

Sensor Service에 OpenFeign 호출 (`POST /sensors/cultivations/{cultivationId}/batch`,
소유권 확인 생략)

↓

device_eui 중복/기존 등록 여부 검증

↓

성공 시 sensor 각 행 INSERT + RabbitMQ Publish (SensorRegisteredEvent, 디바이스별로)

↓

실패 시 에러 응답 → Cultivation Service가 방금 생성한 cultivation을 보상 삭제

---

## 센서 개별 등록/환경 설정 저장

Client

↓

Gateway

↓

Sensor Service

↓

Cultivation Service에 OpenFeign 호출 (`GET /api/v1/cultivations/{cultivationId}/owner`)

↓

소유자 불일치 시 에러 응답(S011) / 일치 시 다음 단계로

↓

sensor 또는 environment_setting에 저장 + RabbitMQ Publish
(SensorRegisteredEvent/SensorDeletedEvent 또는 EnvironmentRangeUpdatedEvent)

---

## 재배 삭제에 따른 정리

Cultivation Service

↓

Cultivation 삭제

↓

RabbitMQ Publish (CultivationDeletedEvent)

↓

Sensor Service 구독 → 해당 cultivation_id의 sensor/environment_setting 삭제 (보상 삭제)

---

# 예외 상황

- RabbitMQ 구독 실패
- Redis 저장 실패
- InfluxDB 저장 실패
- 서비스 재시작 직후 스로틀링 기준 시각 초기화로 인한 일시적 중복 기록 (최초 1회 정도, 영향 미미)
- 조회 데이터 없음
- 잘못된 period 값
- 데이터 집계 실패
- AI Service 호출 실패
- 이미 등록된 device_eui (센서 등록 시 중복)
- 존재하지 않는 센서
- 요청자가 해당 cultivation의 소유자가 아님 (Cultivation Service 소유권 확인 실패)
- 환경 설정 저장 시 필드를 하나도 보내지 않음
- environment_setting 이력이 없어 환경 평균을 계산할 수 없음
- Cultivation Service 소유권 확인 API 호출 실패/타임아웃 (개별 등록/수정/삭제가 실패로 처리됨)
- 센서 일괄 등록 실패로 인한 재배 생성 취소(보상 삭제) 시, 보상 삭제 자체가 실패하는 극단적인 경우 (별도 모니터링/재시도 대상)

---

# 추후 개발 예정

- InfluxDB 장기 보관 데이터에 대한 단계별 Downsampling (예: 최근 7일 원본 → 이후 1분/1시간 평균)
- 재배/센서 타입별 스로틀링 주기 차등 적용
- Grafana 연동
- 이상 데이터 탐지 고도화
