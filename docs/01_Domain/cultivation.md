# Cultivation Service

## 역할

Cultivation Service는 사용자의 버섯 재배 자체와 그 재배에 필요한 센서/환경 데이터를
함께 관리하는 서비스입니다. 재배 생성/조회/종료, 재배당 여러 번 발생하는 수확(flush)
기록, 생육 사진 업로드에 더해, 센서 장치 메타데이터, 사용자가 설정한 목표 환경(위험
한계값), 공공데이터 기반 버섯 참조 데이터, 센서 측정값 저장/조회/통계까지 담당합니다
(구 Sensor Service 통합). 문의(Inquiry) 접수와 관리자 처리도 이 서비스가 담당합니다.

측정값 자체는 Rule Engine Service가 MQTT로 수신·검증한 뒤 RabbitMQ로 전달하면
Cultivation Service가 구독해 Redis(실시간)/InfluxDB(이력)에 저장합니다.

---

# 책임

- 재배 생성/조회/이력 조회/종료
- 수확(flush) 기록 저장 (재배당 여러 번 가능)
- 생육 사진 업로드
- 센서 장치 등록/조회/삭제
- 목표 환경 범위 저장/조회/평균 계산
- 버섯 참조 데이터 관리
- 센서 측정값 저장(Redis/InfluxDB)·조회·통계 (일일 피드백용 일간 통계 포함)
- 문의(Inquiry) 등록/조회, 관리자 답변·처리
- 재배 소유권 확인 API 제공 (내부용, 다른 서비스가 호출)

---

# 주요 기능

## 재배 생성

사용자가 재배 이름, 버섯 종류(`mushroomType`), 등록할 센서 장치 목록을 지정해 재배를
생성합니다. 재배 이름은 같은 사용자 안에서만 유일하면 됩니다. 같은 트랜잭션 안에서
`cultivation` 행과 `sensor` 행(장치가 있는 경우)을 함께 생성합니다 — 이전에는
Cultivation Service와 Sensor Service가 별도 DB였기 때문에 OpenFeign 동기 호출 +
실패 시 보상 삭제 방식이었지만, 병합 이후에는 하나의 로컬 트랜잭션으로 처리되어 더
견고합니다. 재배 상태는 `CREATED`로 시작해, 목표 환경을 처음 저장하면 같은
트랜잭션에서 `RUNNING`으로 자동 전환됩니다.

---

## 수확(flush) 기록 저장

재배가 `RUNNING`인 동안 수확이 있을 때마다 기록합니다. 여러 번 반복 호출할 수 있으며,
재배 상태는 바뀌지 않습니다. 서버가 자동으로 순번(`flush_no`)을 채번합니다.

---

## 재배 종료

재배 상태를 `FINISHED`로 바꿉니다. 수확 기록과는 별개의 API로, 더 이상 수확 정보를
받지 않습니다.

---

## 생육 사진 업로드

사용자가 촬영한 사진을 업로드하면 저장소(MinIO 또는 로컬)에 저장하고 `photo`
메타데이터를 남깁니다. AI Service가 이 사진으로 Vision 분석을 수행합니다.

---

## 센서 장치 등록/조회/삭제

재배에 연결된 센서를 관리합니다. PK는 대리키 `id`이고 `device_eui`는 UNIQUE 제약을
가진 일반 컬럼입니다. 장치 하나가 여러 측정 항목(온도/습도/CO₂/조도)을 동시에 가질 수
있어 `sensor_type` 하위 테이블로 관리합니다. 삭제는 하드 삭제가 아니라 `is_deleted`
플래그로 처리해, 삭제된 센서라도 과거 `environment_setting` 이력이 계속 의미를 가질
수 있게 합니다.

---

## 목표 환경 저장/조회

사용자가 설정한 목표 환경을 범위(threshold_min~threshold_max)로, 측정 항목별 행으로
저장합니다. 부분 수정(예: 온도만)을 지원하며, 수정 시 UPDATE가 아니라 새 행을
INSERT해 이력을 함께 표현합니다. 저장할 때마다 `EnvironmentRangeUpdatedEvent`를
발행합니다 — Rule Engine Service는 이 이벤트로 Redis 캐시를 갱신합니다. 해당 재배가
아직 `CREATED` 상태라면 같은 트랜잭션 안에서 `RUNNING`으로 전환합니다(별도 이벤트
구독 없이 내부 로직으로 처리, 멱등).

---

## 버섯 참조 데이터 관리

공공데이터 기준 5종 버섯의 참조 데이터입니다. `mushroom_type` 코드(예: `OYSTER`)가
`cultivation.mushroom_type`과 연결되는 자연키 역할을 합니다(같은 DB이므로 실제 FK로
연결). 이름/특성/효능/재배 가이드 같은 텍스트 컬럼은 AI Service의 챗봇/버섯 가이드
기능이 `mushroomType`으로 정확히 일치하는 한 건을 조회해 LLM 컨텍스트(RAG 원문)로
그대로 사용합니다. 버섯 종류가 5종 고정이라 별도의 검색 색인 없이 직접 조회로
충분합니다. 온도/습도/CO₂/조도 추천 범위는 `mushroom_reference_threshold`에 항목별로
저장됩니다.

---

## 센서 데이터 저장

MQTT로 발행된 값을 Rule Engine Service가 수신·검증한 뒤 RabbitMQ
(`EnvironmentMeasuredEvent`, 매초)로 전달하면, Cultivation Service가 구독해
저장합니다. Redis는 매초 그대로 갱신하고(현재값), InfluxDB는 재배별로 10초 간격으로
스로틀링해 저장합니다(이력/통계용, 일일 피드백의 환경 통계 계산에도 사용).

---

## 문의(Inquiry)

시스템 관리자(`users.role = ADMIN`)를 제외한 모든 사용자가 문의를 남길 수 있습니다.
문의는 두 유형으로 나뉩니다.

- **일반 문의(GENERAL)**: 재배와 무관한 일반적인 문의입니다. 관리자가 `answer`
  텍스트를 작성해 답변하면 상태가 `OPEN → ANSWERED`로 바뀝니다. 답변 시 사용자에게
  별도 알림은 가지 않으며, 사용자는 문의 목록에서 직접 답변을 확인합니다.
- **경작 문의(CULTIVATION)**: 특정 재배(`cultivationId`)에 대한 문의입니다. 예를 들어
  재배에 오류가 발생해 삭제 후 다시 구성해야 하는 경우 등에 사용합니다. 관리자는 이
  문의에 대해 텍스트로 답변하지 않고, **읽기**와 해당 `cultivationId`가 가리키는
  **재배 삭제** 두 가지만 수행할 수 있습니다. 삭제를 실행하면 그 재배의 `harvest`/
  `photo`/`sensor`/`environment_setting`이 같은 DB 안에서 함께 삭제(CASCADE)되고,
  문의 상태는 `OPEN → CLOSED`로 바뀝니다. 문의 자체는 삭제되지 않고 처리 이력으로
  남습니다(재배가 사라져도 `cultivation_id` 값은 그대로 보존).

작성자가 관리자인지 여부는 API 계층에서 JWT의 `role` 클레임으로 판단하며, DB 제약으로
강제하지 않습니다(role 정보는 Auth Service 소유라 다른 서비스 DB에서 직접 검증할 수
없음).

---

# API

## 재배 생성/조회/이력/종료

POST /cultivations

GET /cultivations

GET /cultivations/{id}

GET /cultivations/history

PATCH /cultivations/{id}/finish

---

## 수확 기록 저장/조회

POST /cultivations/{id}/harvests

GET /cultivations/{id}/harvests

---

## 사진 업로드

POST /cultivations/{id}/photos

---

## 센서 등록/조회/삭제 (배치 포함)

POST /cultivations/{id}/sensors/batch (내부용)

POST/GET/DELETE /cultivations/{id}/sensors

---

## 목표 환경 저장/조회

PATCH /cultivations/{id}/environment

GET /cultivations/{id}/environment-history

GET /cultivations/{id}/environment-average

POST /environment-averages (내부용, 배치 조회)

---

## 센서 통계

GET /cultivations/{id}/current

GET /cultivations/{id}/stats (기간별 집계, AI Service의 일일 피드백은 최근 24시간
기준으로 호출)

---

## 버섯 참조 데이터 조회 (내부용)

GET /mushroom-references/{mushroomType}

GET /mushroom-references

---

## 재배 소유권 확인 (내부용)

GET /cultivations/{id}/owner

---

## 문의

POST /inquiries (일반 사용자, GENERAL 또는 CULTIVATION 유형 선택)

GET /inquiries (본인 문의 목록 조회)

GET /inquiries/{id}

GET /admin/inquiries (관리자, 전체 문의 목록 조회)

PATCH /admin/inquiries/{id}/answer (관리자, GENERAL 유형 답변 등록)

DELETE /admin/cultivations/{id} (관리자, CULTIVATION 유형 문의 처리 — 재배 삭제)

---

# Database

Cultivation Service는 하나의 PostgreSQL Database와 Redis/InfluxDB를 사용합니다.

## Table

- cultivation (UNIQUE(user_id, name), status CREATED/RUNNING/FINISHED)
- harvest (재배당 여러 건, UNIQUE(cultivation_id, flush_no))
- photo (object_key + storage_type)
- measurement_type (온도/습도/CO2/조도 4종 참조 테이블)
- sensor / sensor_type
- environment_setting
- mushroom_reference / mushroom_reference_threshold
- inquiry (일반 문의 / 경작 문의)

자세한 내용은 [cultivation-db.md](../03_Database/cultivation-db.md) 참고.

---

# Redis

- 최신 센서 데이터 (`cultivation:{cultivationId}:current`, TTL 없음)

---

# 다른 서비스와의 통신

## 호출받는 서비스

### AI Service

- 생육 사진 조회(Vision 분석용, MinIO/로컬 경유), 환경 변경 이력/평균/일간 통계 조회,
  버섯 참조 데이터(RAG 컨텍스트) 조회, 재배 정보 조회

### Rule Engine Service

- 목표 환경 범위 캐시 미스 시 fallback 조회

### DatasourceGenerator

- 서비스 시작 시 전체 센서 목록 조회 (내부용)

### Notification Service

- Telegram/Discord 챗봇 웹훅 발신자 → cultivationId 조회 시 재배 정보 확인 (AI
  Service 경유)

### API Gateway

- 재배/수확/사진/센서/환경/통계/문의 관련 REST API 요청

---

# Event

## Publish

### HarvestCompletedEvent

수확 기록 저장 시 발행. Notification Service가 구독해 알림을 보내고, AI Service가
구독해 "인사이트" 사례를 적재합니다.

### CultivationFinishedEvent

재배 종료 시 발행. Notification Service가 구독해 알림을 보냅니다.

### EnvironmentRangeUpdatedEvent

목표 환경 저장 시 발행. Rule Engine Service가 구독해 Redis 캐시를 갱신합니다. 재배의
`CREATED → RUNNING` 전환은 같은 서비스 내부 로직으로 처리되므로 더 이상 이 이벤트를
구독할 필요가 없습니다.

### SensorRegisteredEvent / SensorDeletedEvent

센서 등록/삭제 시 발행. DatasourceGenerator가 구독해 `sensor_cache`(메모리)를
갱신합니다.

### CultivationDeletedEvent

경작 문의 처리로 재배가 삭제될 때 발행합니다. 현재 이 이벤트를 구독하는 서비스는
없지만, 추후 Notification Service의 채널 정리 등 다른 서비스의 후속 정리 작업을 위해
남겨둡니다.

---

## Subscribe

### EnvironmentMeasuredEvent

Rule Engine Service가 발행. Redis/InfluxDB에 저장합니다.

### SensorErrorEvent

Rule Engine Service가 발행. 센서 `status`를 갱신합니다.

### UserDeletedEvent

Auth Service가 회원 탈퇴 시 발행. 해당 사용자의 재배 데이터를 비활성화합니다.

---

# Sequence

관련 시퀀스는 [create-cultivation.md](../04_sequence/create-cultivation.md),
[harvest.md](../04_sequence/harvest.md),
[growth-analysis.md](../04_sequence/growth-analysis.md),
[sensor-data.md](../04_sequence/sensor-data.md),
[environment-control.md](../04_sequence/environment-control.md),
[sensor-error.md](../04_sequence/sensor-error.md),
[daily-feedback.md](../04_sequence/daily-feedback.md),
[insight.md](../04_sequence/insight.md),
[inquiry.md](../04_sequence/inquiry.md) 참고.

---

# 예외 상황

- 재배 이름 중복 (같은 사용자 안에서)
- 존재하지 않는 재배
- 다른 사용자의 재배에 대한 접근 시도
- `FINISHED` 상태 재배에 대한 수확 기록/사진 업로드 시도
- 사진 저장소 업로드 실패
- 중복된 device_eui 등록 시도
- 존재하지 않는 센서
- MQTT/RabbitMQ 수신 실패
- 관리자가 아닌 사용자가 문의 답변/재배 삭제 시도 (권한 없음)
- 존재하지 않는 문의/재배에 대한 답변·삭제 시도
- GENERAL 문의에 재배 삭제를 시도하거나 CULTIVATION 문의에 텍스트 답변을 시도하는 등
  유형에 맞지 않는 처리 시도

---

# 추후 개발 예정

- 재배 공유(여러 사용자가 같은 재배에 접근)
- 버섯 참조 데이터 관리 UI/관리자 API
- 경작 문의를 재배 삭제 없이 종료 처리하는 기능 (현재는 삭제만 가능)
