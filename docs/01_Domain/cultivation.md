# Cultivation Service

## 역할

Cultivation Service는 사용자의 버섯 재배 자체와 그 재배에 필요한 센서/환경 데이터를
함께 관리하는 서비스입니다. 재배 생성/조회/종료, 수확 기록(재배당 한 건), 생육 사진
업로드에 더해, 센서 장치 메타데이터, 사용자가 설정한 목표 환경(위험 한계값), 공공데이터
기반 버섯 참조 데이터, 센서 측정값 저장/조회/통계까지 담당합니다
(구 Sensor Service 통합). 문의(Inquiry) 접수와 관리자 처리도 이 서비스가 담당합니다.

측정값 자체는 Rule Engine Service가 MQTT로 수신·검증한 뒤 RabbitMQ로 전달하면
Cultivation Service가 구독해 Redis(실시간)/InfluxDB(이력)에 저장합니다.

---

# 책임

- 재배 생성/조회/이력 조회/종료
- 생육/수확 모드 자동 전환
- 재배 멤버 관리 (초대/조회/제거, OWNER/MEMBER)
- 수확 기록 저장 (재배당 한 건)
- 생육 사진 업로드
- 센서 장치 등록/조회/삭제
- 목표 환경 범위 저장/조회/평균 계산
- 버섯 참조 데이터 관리
- 센서 측정값 저장(Redis/InfluxDB)·조회·통계 (일일 피드백용 일간 통계 포함)
- 문의(Inquiry) 등록/조회, 관리자 답변·처리
- 재배 소유권 확인 API 제공 (내부용, 다른 서비스가 호출)
- 환경 준수율 집계 제공 + 상품 등급 원점수 수신·등급 매핑 (내부용, AI Service가 호출)

---

# 주요 기능

## 재배 생성

사용자가 재배 이름, 버섯(`mushroomId`), 등록할 센서 장치 목록을 지정해 재배를
생성합니다. 재배 이름은 같은 사용자 안에서만 유일하면 됩니다. 같은 트랜잭션 안에서
`cultivation` 행과 `cultivation_sensor` 행(장치가 있는 경우), 그리고 생성자를 OWNER로 등록하는
`cultivation_member` 행을 함께 생성합니다 — 이전에는 Cultivation Service와 Sensor
Service가 별도 DB였기 때문에 OpenFeign 동기 호출 + 실패 시 보상 삭제 방식이었지만,
병합 이후에는 하나의 로컬 트랜잭션으로 처리되어 더 견고합니다. 재배 상태는
`CREATED`로 시작해, 목표 환경을 처음 저장하면 같은 트랜잭션에서 `RUNNING`으로 자동
전환됩니다.

---

## 수확 기록 저장

재배가 `RUNNING`인 동안 수확을 기록합니다. 병 재배를 전제로 재배 하나(병 하나)당
수확은 한 건만 기록하며(`UNIQUE(cultivation_id)`), 두 번째 기록 시도는 거부됩니다.
기록 자체는 재배 상태를 바꾸지 않으며, 종료는 별도의 API로 처리합니다.

---

## 재배 종료

재배 상태를 `FINISHED`로 바꿉니다. 수확 기록과는 별개의 API로, 더 이상 수확 정보를
받지 않습니다.

---

## 생육/수확 모드 자동 전환

`status`(생애주기: CREATED/RUNNING/FINISHED)와 별개로, `RUNNING`인 동안의 환경
단계를 나타내는 `mode`(GROWTH/HARVEST)를 둡니다. 재배는 `GROWTH`로 시작합니다.

생육 분석([growth-analysis.md](../04_sequence/growth-analysis.md)) 요청마다 AI
Service가 돌려주는 `growthStage`가 `수확적기`이면, Cultivation Service는 그 응답을
받은 같은 요청 처리 안에서 `cultivation.mode`를 `HARVEST`로 전환합니다 — 사용자가
직접 조작하지 않고 시스템이 자동으로 판단합니다. 재배당 수확이 한 번뿐이라 다시
`GROWTH`로 되돌아가는 경우는 없습니다.

`mode`가 `HARVEST`로 바뀌면 `CultivationModeChangedEvent`를 발행해 사용자에게
알립니다. 다만 환경 설정(`environment_setting`)을 수확기에 맞는 값으로 자동
전환하는 로직은 아직 넣지 않았습니다 — 수확기 환경 임계값 데이터를 아직 확보하지
못했기 때문이며, 데이터가 준비되면 추후 개발할 예정입니다. 그때까지는 `mode`가
상태 표시와 알림 트리거 역할만 합니다.

---

## 재배 멤버 관리

재배 하나를 여러 사용자가 함께 볼 수 있게 하는 공유 기능입니다. 재배 생성 시
생성자가 `cultivation_member`에 `role = OWNER`로 자동 등록되며, OWNER는 다른
사용자를 `role = MEMBER`로 초대할 수 있습니다. `UNIQUE(cultivation_id, user_id)`로
같은 사용자의 중복 등록을 막습니다.

OWNER만 환경 설정/수확 기록/센서 관리/멤버 관리/재배 종료·삭제를 할 수 있고,
MEMBER는 조회만 가능하다고 우선 가정합니다 — 세부 권한 정책은 추후 조정될 수
있습니다. 멤버 초대는 현재 `userId`를 직접 지정하는 방식만 지원하며, 닉네임/이메일
검색으로 초대 대상을 찾는 기능은 Auth Service 연동이 필요해 추후 개발 예정입니다.
`cultivation.user_id`(생성자)는 이 기능 도입 이후에도 그대로 유지합니다 — 재배 이름
유일성, 재배 소유권 확인 API 등 기존에 이 컬럼을 참조하던 부분을 그대로 두기 위함이며,
`cultivation_member`의 OWNER 행과 항상 같은 값을 가집니다.

---

## 상품 등급 매핑

수확이 기록되면 `HarvestCompletedEvent`를 구독한 AI Service가 생육 점수 평균과 환경
유지 점수를 합산한 원점수(`productScore`, 0~100)를 비동기로 계산해 돌려줍니다.
Cultivation Service는 이 값을 받아 아래 구간으로 매핑해 `harvest.product_grade`에
저장합니다.

| product_score | product_grade |
|----------------|----------------|
| 95점 이상 | TOP (최상) |
| 80점 이상 | HIGH (상) |
| 60점 이상 | MID (중) |
| 60점 미만 | LOW (하) |

환경 유지 점수 계산에 필요한 원본 데이터(InfluxDB 측정값)는 Cultivation Service만
접근할 수 있어, AI Service의 요청에 맞춰 재배 기간 동안 측정 항목별로
`mushroom_reference_threshold` 추천 범위 안에 있었던 시간 비율을 집계해 제공합니다.
자세한 흐름은 [product-grade.md](../04_sequence/product-grade.md) 참고.

---

## 생육 사진 업로드

사용자가 촬영한 사진을 업로드하면 저장소(MinIO 또는 로컬)에 저장하고 `cultivation_photo`
메타데이터를 남깁니다. AI Service가 이 사진으로 Vision 분석을 수행합니다.

---

## 센서 장치 등록/조회/삭제

재배에 연결된 센서를 관리합니다(`cultivation_sensor`). PK는 대리키 `id`이고
`device_eui`는 UNIQUE 제약을 가진 일반 컬럼입니다. 장치 하나가 여러 측정 항목(온도/
습도/CO₂/조도)을 동시에 가질 수 있어 `cultivation_sensor_type` 하위 테이블로
관리합니다(측정 항목 자체는 전역 참조 테이블 `sensor_type`을 대리키 `sensor_type_id`로
참조). 삭제는 하드 삭제가 아니라 `is_deleted` 플래그로 처리해, 삭제된 센서라도 과거
`environment_setting` 이력이 계속 의미를 가질 수 있게 합니다.

---

## 목표 환경 저장/조회

사용자가 설정한 목표 환경을 범위(threshold_min~threshold_max)로, 측정 항목별 행으로
저장합니다. 항목은 `sensor_type_id`로 전역 참조 테이블 `sensor_type`을 가리키며,
단위(`value_unit`)는 `environment_setting`에 중복 저장하지 않고 `sensor_type`과
JOIN해 조회합니다(`threshold_unit` 컬럼 제거, 정규화 개선). 부분 수정(예: 온도만)을
지원하며, 수정 시 UPDATE가 아니라 새 행을 INSERT해 이력을 함께 표현합니다. 저장할
때마다 `EnvironmentRangeUpdatedEvent`를 발행합니다 — Rule Engine Service는 이
이벤트로 Redis 캐시를 갱신합니다. 해당 재배가 아직 `CREATED` 상태라면 같은
트랜잭션 안에서 `RUNNING`으로 전환합니다(별도 이벤트 구독 없이 내부 로직으로 처리,
멱등).

---

## 버섯 참조 데이터 관리

공공데이터 기준 5종 버섯의 참조 데이터입니다. 자연키 코드 컬럼은 두지 않고, 대리키
`id`를 `cultivation.mushroom_id`가 실제 FK로 참조합니다(같은 DB이므로 실제 FK로
연결, ERD 기준). 이름/특성/효능/재배 가이드 같은 텍스트 컬럼은 AI Service의 챗봇/버섯
가이드 기능이 `mushroomId`로 정확히 일치하는 한 건을 조회해 LLM 컨텍스트(RAG 원문)로
그대로 사용합니다. 버섯 종류가 5종 고정이라 별도의 검색 색인 없이 직접 조회로
충분합니다. 온도/습도/CO₂/조도 추천 범위는 `mushroom_reference_threshold`에
`sensor_type_id`별로 저장됩니다(단위는 마찬가지로 `sensor_type`과 JOIN해 조회).
프론트엔드는 `GET /mushroom-references` 목록을 조회해 id→이름 매핑을 직접 만들어
화면에 표시합니다.

---

## 센서 데이터 저장

MQTT로 발행된 값을 Rule Engine Service가 수신·검증한 뒤 RabbitMQ
(`EnvironmentMeasuredEvent`, 매초)로 전달하면, Cultivation Service가 구독해
저장합니다. Redis는 매초 그대로 갱신하고(현재값), InfluxDB는 재배별로 10초 간격으로
스로틀링해 저장합니다(이력/통계용, 일일 피드백의 환경 통계 계산에도 사용).

---

## 문의(Inquiry)

시스템 관리자(`users.role = ADMIN`)를 제외한 모든 사용자가 문의를 남길 수 있습니다.
문의는 `inquiry_category`로 관리되는 카테고리로 나뉩니다(시드 데이터: `id=1` "일반
문의", `id=2` "경작 문의" — 기존 GENERAL/CULTIVATION enum을 참조 테이블로 대체해
향후 카테고리 추가가 데이터 변경만으로 가능해졌습니다).

- **일반 문의(category_id=1)**: 재배와 무관한 일반적인 문의입니다. 관리자가 답변을
  작성하면 상태가 `PENDING → RESOLVED`로 바뀝니다. 답변 시 사용자에게 별도 알림은
  가지 않으며, 사용자는 문의 목록에서 직접 답변을 확인합니다.
- **경작 문의(category_id=2)**: 특정 재배(`cultivationId`)에 대한 문의입니다. 예를
  들어 재배에 오류가 발생해 삭제 후 다시 구성해야 하는 경우 등에 사용합니다. 관리자는
  이 문의에 대해 텍스트로 답변하지 않고, **읽기**와 해당 `cultivationId`가 가리키는
  **재배 삭제** 두 가지만 수행할 수 있습니다. 삭제를 실행하면 그 재배의 `harvest`/
  `cultivation_photo`/`cultivation_sensor`/`environment_setting`이 같은 DB 안에서
  함께 삭제(CASCADE)되고, 문의 상태는 `PENDING → RESOLVED`로 바뀝니다. 문의 자체는
  삭제되지 않고 처리 이력으로 남습니다(재배가 사라져도 `cultivation_id` 값은 그대로
  보존).

문의 등록/답변은 기존처럼 `inquiry` 한 테이블에 직접 쓰지 않고, `inquiry`(메타데이터
+ 상태)와 `inquiry_answer`(문의 본문/답변을 시간순으로 쌓는 스레드) 두 테이블로
나뉩니다. 문의를 등록하면 `inquiry` 행(status=PENDING)과 함께, 문의 내용을 담은
`inquiry_answer` 행(`content`=문의 내용, `answer_content`=NULL, `pre_id`=NULL, 즉
스레드의 최초 행)이 함께 생성됩니다. 일반 문의에 관리자가 답변하면 새
`inquiry_answer` 행(`content`=NULL, `answer_content`=답변 내용, `pre_id`=최초 행의
id)이 추가되고 `inquiry.status`가 `RESOLVED`로 바뀝니다. 사용자가 답변에 대해 추가
질문을 남기는 스레드 확장은 스키마상 가능하지만(`pre_id` 체인을 계속 이어감), 현재
범위는 기존과 동일하게 "관리자 답변 1회"로 단순하게 다룹니다. 경작 문의 처리는
`inquiry_answer` 행을 만들지 않고 `inquiry.status`만 `RESOLVED`로 바꿉니다(텍스트
답변이 없으므로 기존 동작과 동일).

`admin_id`/`answered_at` 컬럼은 더 이상 `inquiry`에 없습니다(스레드 테이블
`inquiry_answer.created_at`으로 "언제"는 확인 가능하며, "누가"는 현재 스키마에 없고
추후 `inquiry_answer.admin_id` 추가를 검토할 수 있습니다).

작성자가 관리자인지 여부는 API 계층에서 JWT의 `role` 클레임으로 판단하며, DB 제약으로
강제하지 않습니다(role 정보는 Auth Service 소유라 다른 서비스 DB에서 직접 검증할 수
없음). `inquiry.category_id`가 "경작 문의"일 때만 `cultivation_id`가 채워지는
불변식도 애플리케이션 레벨에서만 보장하며 DB CHECK로 강제하지 않습니다(카테고리가
FK라 특정 값 하드코딩 CHECK가 부적절하다고 판단).

---

# API

## 재배 생성/조회/이력/종료

POST /cultivations

GET /cultivations

GET /cultivations/{id}

GET /cultivations/history

PATCH /cultivations/{id}/finish

---

## 재배 멤버

POST /cultivations/{id}/members (OWNER만, userId로 멤버 초대)

GET /cultivations/{id}/members (멤버 목록 조회)

DELETE /cultivations/{id}/members/{userId} (OWNER만, 멤버 제거)

GET /cultivations/mine (내가 OWNER/MEMBER로 속한 cultivation_id 목록 — 인사이트
후보 제외용으로 AI Service도 호출)

---

## 수확 기록 저장/조회

POST /cultivations/{id}/harvest

GET /cultivations/{id}/harvest

---

## 상품 등급 (내부용, AI Service가 호출)

GET /cultivations/{id}/environment-compliance (측정 항목별 추천 범위 내 비율 집계)

PATCH /cultivations/{id}/harvest/product-score (AI Service가 계산한 원점수 전달 → 등급 매핑 후 저장)

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

GET /mushroom-references/{mushroomId}

GET /mushroom-references

---

## 재배 소유권 확인 (내부용)

GET /cultivations/{id}/owner

---

## 문의

GET /inquiry-categories (카테고리 목록 조회 — 일반 문의/경작 문의)

POST /inquiries (일반 사용자, categoryId로 카테고리 선택)

GET /inquiries (본인 문의 목록 조회)

GET /inquiries/{id} (스레드형 답변 목록 포함)

GET /admin/inquiries (관리자, 전체 문의 목록 조회)

PATCH /admin/inquiries/{id}/answer (관리자, 일반 문의 답변 등록 — inquiry_answer 행 추가)

DELETE /admin/cultivations/{id} (관리자, 경작 문의 처리 — 재배 삭제)

---

# Database

Cultivation Service는 하나의 PostgreSQL Database와 Redis/InfluxDB를 사용합니다.

## Table

- cultivation (UNIQUE(user_id, name), status CREATED/RUNNING/FINISHED)
- cultivation_member (재배 공유, UNIQUE(cultivation_id, user_id), role OWNER/MEMBER)
- harvest (재배당 한 건, UNIQUE(cultivation_id))
- cultivation_photo (object_key + storage_type)
- sensor_type (온도/습도/CO2/조도 4종 참조 테이블, 대리키)
- cultivation_sensor / cultivation_sensor_type
- environment_setting (sensor_type_id FK, threshold_unit 없음 — sensor_type과 JOIN)
- mushroom_reference / mushroom_reference_threshold
- inquiry_category / inquiry / inquiry_answer (카테고리 + 스레드형 답변)

자세한 내용은 [cultivation-db.md](../03_Database/cultivation-db.md) 참고.

---

# Redis

- 최신 센서 데이터 (`cultivation:{cultivationId}:current`, TTL 없음)

---

# 다른 서비스와의 통신

## 호출받는 서비스

### AI Service

- 생육 사진 조회(Vision 분석용, MinIO/로컬 경유), 환경 변경 이력/평균/일간 통계 조회,
  버섯 참조 데이터(RAG 컨텍스트) 조회, 재배 정보 조회, 환경 준수율 집계 조회(상품
  등급용), 상품 등급 원점수 전달

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
구독해 "인사이트" 사례를 적재하고 상품 등급 원점수를 계산합니다(계산 결과는 이벤트가
아니라 별도의 OpenFeign 콜백으로 전달됩니다).

### CultivationFinishedEvent

재배 종료 시 발행. Notification Service가 구독해 알림을 보냅니다.

### CultivationModeChangedEvent

생육 분석 결과 `growthStage`가 `수확적기`가 되어 `cultivation.mode`가 `GROWTH →
HARVEST`로 자동 전환될 때 발행. Notification Service가 구독해 "수확 시기가
가까워졌습니다" 알림을 보냅니다.

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
[product-grade.md](../04_sequence/product-grade.md),
[inquiry.md](../04_sequence/inquiry.md) 참고.

---

# 예외 상황

- 재배 이름 중복 (같은 사용자 안에서)
- 존재하지 않는 재배
- 다른 사용자의 재배에 대한 접근 시도
- 이미 멤버인 사용자를 다시 초대하는 시도
- OWNER를 멤버 목록에서 제거하려는 시도 (허용하지 않음 — 재배는 항상 OWNER가 있어야 함)
- 존재하지 않는 사용자를 멤버로 초대하는 시도
- MEMBER 권한으로 OWNER 전용 동작(환경 설정/수확 기록/센서 관리/멤버 관리/종료·삭제) 시도
- `FINISHED` 상태 재배에 대한 수확 기록/사진 업로드 시도
- 이미 수확 기록이 있는 재배에 대한 중복 수확 기록 시도
- 존재하지 않는 재배/수확에 대한 상품 등급 원점수 전달 시도, 0~100 범위를 벗어난
  원점수 전달 시도
- 사진 저장소 업로드 실패
- 중복된 device_eui 등록 시도
- 존재하지 않는 센서
- MQTT/RabbitMQ 수신 실패
- 관리자가 아닌 사용자가 문의 답변/재배 삭제 시도 (권한 없음)
- 존재하지 않는 문의/재배에 대한 답변·삭제 시도
- 일반 문의에 재배 삭제를 시도하거나 경작 문의에 텍스트 답변을 시도하는 등 카테고리에
  맞지 않는 처리 시도

---

# 추후 개발 예정

- 닉네임/이메일로 멤버 초대 (현재는 `userId` 직접 지정만 지원, Auth Service 연동 필요)
- `mode`가 `HARVEST`로 바뀔 때 `environment_setting`을 수확기 값으로 자동 전환하는
  로직 (수확기 환경 임계값 데이터 확보 후 진행)
- 버섯 참조 데이터 관리 UI/관리자 API
- 경작 문의를 재배 삭제 없이 종료 처리하는 기능 (현재는 삭제만 가능)
