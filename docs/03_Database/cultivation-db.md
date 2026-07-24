# Cultivation Database

## 개요

Cultivation Database는 사용자의 버섯 재배 자체와 그 재배에 필요한 센서/환경 데이터를
함께 관리하는 Cultivation Service 소유의 PostgreSQL Database입니다(구 Sensor
Service 통합). 재배 생성/진행/종료, 수확 기록, 생육 사진 메타데이터, 센서
장치, 목표 환경(위험 한계값) 이력, 공공데이터 기반 버섯 참조 데이터, 문의(Inquiry)를
모두 이 DB에서 관리합니다.

이전에는 Cultivation DB와 Sensor DB가 분리되어 있어 `cultivation_id`/`mushroom_id`
같은 참조가 소프트 참조(FK 없음)였지만, 병합 이후에는 같은 DB 안이므로 실제 FK로
바꿀 수 있는 관계는 FK로 강제합니다(`cultivation_sensor.cultivation_id`,
`environment_setting.cultivation_id`, `cultivation.mushroom_id`).

외부 팀이 제공한 ERD(DDL)를 기준으로 테이블명/컬럼을 전면 정리했습니다(`mushroom_embedding`
벡터 검색 기능만 우리 프로젝트 범위에서 제외). 아래 표시대로 테이블명이 여럿 바뀌었습니다.

| 기존 | 변경 |
|------|------|
| `photo` | `cultivation_photo` |
| `sensor` | `cultivation_sensor` |
| `sensor_type` (브릿지 테이블) | `cultivation_sensor_type` |
| `measurement_type` | `sensor_type` (자연키 → 대리키) |
| `inquiry`(단일 테이블) | `inquiry` + `inquiry_category` + `inquiry_answer` (분리) |

---

# ERD

```
cultivation
──────────────────────────────────────────────
PK  id
    user_id
    name
    mushroom_id
    status
    mode
    started_at
    finished_at
    created_at
    updated_at

    UNIQUE(user_id, name)

          │ 1
          │
          ├──────────────┬────────────────┬──────────────────────┐
          ▼              ▼                ▼                      ▼
harvest              cultivation_photo  cultivation_sensor   environment_setting
──────────────       ─────────────────  ──────────────────   ──────────────────
PK  id                PK  id             PK  id                PK  id
FK  cultivation_id    FK  cultivation_id FK  cultivation_id     FK  cultivation_id
(UNIQUE)                  object_key     device_eui (UNIQUE)    FK  sensor_type_id
    harvest_weight        storage_type   location               threshold_min
    memo                  uploaded_at    location_detail        threshold_max
    harvested_at          created_at     device_model           created_at
    product_score                       status                 updated_at
    product_grade                       is_deleted
                                         created_at
                                              │ 1
                                              │
                                              ▼
                                        cultivation_sensor_type
                                        ───────────────────────
                                        PK  id
                                        FK  cultivation_sensor_id
                                        FK  sensor_type_id

                                        UNIQUE(cultivation_sensor_id, sensor_type_id)


cultivation
    │ 1
    ▼
cultivation_member
──────────────────────────────────────────────
PK  id
FK  cultivation_id
    user_id           (Auth Service, 소프트 참조)
    role
    joined_at

    UNIQUE(cultivation_id, user_id)


sensor_type (전역 참조, 4종 고정 시드 데이터)
──────────────────────────────────────────────
PK  id                (대리키 — 기존 measurement_type.code 자연키는 완전 제거)
    sensor_type        (예: '온도')
    value_unit         (예: '°C')


mushroom_reference (전역 참조 테이블)
──────────────────────────────────────────────
PK  id                (cultivation.mushroom_id가 FK로 참조)
    mushroom_name_ko
    mushroom_name_en
    mushroom_scientific_name
    characteristics
    health_benefits
    cultivation_guide
    additional_info
    created_at
    updated_at

          │ 1
          │
          ▼
mushroom_reference_threshold
──────────────────────────────────────────────
PK  id
FK  mushroom_reference_id
FK  sensor_type_id
    threshold_min
    threshold_max
    created_at
    updated_at

    UNIQUE(mushroom_reference_id, sensor_type_id)


inquiry_category (전역 참조, 시드 데이터)
──────────────────────────────────────────────
PK  id
    category_name      (id=1 '일반 문의', id=2 '경작 문의')


inquiry  (cultivation_id는 의도적으로 소프트 참조 — 재배 삭제 후에도 문의 기록 보존)
──────────────────────────────────────────────
PK  id
    user_id            (Auth Service, 소프트 참조)
FK  category_id
    cultivation_id     (nullable, 소프트 참조. category_id가 "경작 문의"일 때만 채워짐)
    title
    status
    created_at

          │ 1
          │
          ▼
inquiry_answer  (스레드형. pre_id는 셀프 FK — NULL이면 최초 문의 행)
──────────────────────────────────────────────
PK  id
FK  inquiry_id
    content            (문의 본문 또는 사용자의 추가 질문)
    answer_content     (관리자 답변)
FK  pre_id             (self-FK, nullable)
    created_at
```

---

# Table

## cultivation

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | X | 소유자 (Auth Service의 userId, 소프트 참조) |
| name | VARCHAR(100) | X | 재배 이름 (사용자 지정), `UNIQUE(user_id, name)` |
| mushroom_id | BIGINT | X | 버섯 참조 ID, FK `mushroom_reference.id` |
| status | VARCHAR(20) | X | 재배 상태 (CREATED/RUNNING/FINISHED), 기본값 CREATED |
| mode | VARCHAR(20) | X | 생육 단계 (GROWTH/HARVEST), 기본값 GROWTH |
| started_at | DATETIME | O | 첫 목표 환경 설정 저장 시점에 자동으로 채워짐 |
| finished_at | DATETIME | O | 재배 종료 시점 |
| created_at | DATETIME | X | 생성 일시 |
| updated_at | DATETIME | X | 수정 일시 |

재배 이름은 전역이 아니라 **같은 사용자 안에서만 유일**합니다. `status`는 재배 생성
시 `CREATED`로 시작해, 목표 환경을 처음 저장하는 시점에 같은 트랜잭션에서
`RUNNING`으로 자동 전환되고, 사용자가 명시적으로 종료하면 `FINISHED`가 됩니다.
`mushroom_id`는 이제 같은 DB 안의 `mushroom_reference.id`를 실제 FK로
참조합니다(병합 이전에는 소프트 참조였음). 자연키 코드 컬럼은 두지 않고 대리키
`id`만으로 버섯을 식별합니다(ERD 기준).

`mode`는 `status`와 별개 축입니다 — `status`가 재배의 생애주기(생성/진행/종료)라면
`mode`는 `RUNNING`인 동안의 환경 단계(생육기/수확기)입니다. `GROWTH`로 시작해,
생육 분석 결과 `growth_record.analysis_data->>'growthStage'`가 `수확적기`가 되면 같은 요청 처리
안에서 `HARVEST`로 자동 전환됩니다(사용자 조작 없음). 재배당 수확이 한 번뿐이라
`HARVEST`에서 다시 `GROWTH`로 되돌아가지 않습니다. 자세한 흐름은
[growth-analysis.md](../04_sequence/growth-analysis.md) 참고.

---

## cultivation_member

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id` |
| user_id | BIGINT | X | 멤버 (Auth Service의 userId, 소프트 참조) |
| role | VARCHAR(20) | X | 역할 (OWNER/MEMBER), 기본값 MEMBER |
| joined_at | DATETIME | X | 참여(등록) 일시 |

재배 하나를 여러 사용자가 함께 볼 수 있게 하는 공유 기능입니다. 재배 생성 시
같은 트랜잭션에서 생성자를 `role = OWNER`로 자동 등록하며, 이후 OWNER가 다른
사용자를 `role = MEMBER`로 초대할 수 있습니다. `UNIQUE(cultivation_id, user_id)`로
같은 사용자가 같은 재배에 중복 등록되는 것을 막습니다.

권한은 OWNER만 환경 설정/수확 기록/센서 관리/멤버 관리/재배 종료·삭제를 할 수 있고,
MEMBER는 조회만 가능하다고 우선 가정합니다 — 세부 권한 정책은 추후 조정될 수
있습니다. `cultivation.user_id`(생성자)는 이 테이블 도입 이후에도 그대로 유지합니다
— 기존에 이 컬럼을 참조하는 문서/로직(재배 이름 유일성, 재배 소유권 확인 API 등)을
그대로 두기 위함이며, `cultivation_member`의 OWNER 행과 값이 항상 일치합니다.

---

## harvest

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id`, UNIQUE |
| harvest_weight | NUMERIC(6,2) | O | 수확량(g) |
| memo | TEXT | O | 수확 메모 |
| harvested_at | DATETIME | X | 수확 일시 |
| product_score | NUMERIC(4,1) | O | 상품 등급 원점수(0~100). AI Service가 생육 점수 평균과 환경 유지 점수를 합산해 계산 후 전달 |
| product_grade | VARCHAR(10) | O | 상품 등급 (TOP/HIGH/MID/LOW). Cultivation Service가 `product_score`를 구간별로 매핑 |

재배 하나당 수확은 한 건만 기록됩니다(`UNIQUE(cultivation_id)`). 병 재배를 전제로
하나의 재배 단위(병)에서 자란 버섯은 한 번에 수확하며, 재배마다 여러 번 나눠
수확하지 않습니다. `harvest_weight`/`product_score`는 외부 ERD와 정밀도를 맞춰
각각 `NUMERIC(6,2)`(소수점 둘째 자리까지)/`NUMERIC(4,1)`(0.0~100.0)로 둡니다.

`product_score`/`product_grade`는 수확 기록 시점에는 비어 있다가, `HarvestCompletedEvent`를
구독한 AI Service가 생육 점수 평균(자체 DB의 `growth_record`)과 환경 유지 점수(Cultivation
Service에 새로 조회)를 합산해 원점수를 계산해 돌려주면 그때 채워집니다. Cultivation
Service는 전달받은 원점수를 아래 구간으로 매핑해 `product_grade`에 저장합니다.

| product_score | product_grade |
|----------------|----------------|
| 95점 이상 | TOP (최상) |
| 80점 이상 | HIGH (상) |
| 60점 이상 | MID (중) |
| 60점 미만 | LOW (하) |

자세한 흐름은 [product-grade.md](../04_sequence/product-grade.md) 참고.

---

## cultivation_photo

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id` |
| object_key | VARCHAR(500) | X | 저장소 내 상대 경로 |
| storage_type | VARCHAR(10) | X | 저장 위치 (MINIO/LOCAL), 기본값 MINIO |
| uploaded_at | DATETIME | X | 사용자가 촬영/업로드한 시각 |
| created_at | DATETIME | X | DB에 저장된 시각 |

외부 ERD의 `photo` 테이블명을 `cultivation_photo`로 바꿨습니다(다른 도메인의 사진과
구분되도록). 컬럼 구성/의미는 기존과 동일합니다. 자세한 내용은
[minio.md](./minio.md) 참고.

---

## sensor_type

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK (대리키) |
| sensor_type | VARCHAR(20) | X | 표시용 한글 이름 (예: 온도) |
| value_unit | VARCHAR(10) | X | 단위 (예: °C, %, ppm, lux) |

측정 항목 4종(온도/습도/CO2/조도)의 고정 시드 데이터입니다. 외부 ERD 기준으로
기존 `measurement_type`의 자연키(`code` VARCHAR PK)를 제거하고 대리키 `id`로
전환했습니다(테이블명도 `measurement_type` → `sensor_type`으로 변경). `environment_setting`/
`mushroom_reference_threshold`/`cultivation_sensor_type` 세 곳이 이 값을 자유
문자열로 반복하지 않고 `sensor_type_id`로 이 테이블을 FK 참조합니다.

---

## cultivation_sensor

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id` |
| device_eui | VARCHAR(32) | X | 장치 고유 식별자 (LoRaWAN DevEUI), UNIQUE |
| location | VARCHAR(10) | O | 설치 위치 (짧은 코드, 예: A동) |
| location_detail | VARCHAR(50) | O | 세부 위치 |
| device_model | VARCHAR(100) | O | 장치 모델명 |
| status | VARCHAR(10) | X | 상태 (ONLINE/OFFLINE/ERROR), 기본값 OFFLINE, 시스템이 관리 |
| is_deleted | BOOLEAN | X | 소프트 삭제 플래그, 기본값 FALSE |
| created_at | DATETIME | X | 등록 일시 |

외부 ERD의 `sensor` 테이블명을 `cultivation_sensor`로 바꿨습니다(컬럼 구성/의미는
동일). PK는 대리키 `id`이고 `device_eui`는 UNIQUE 제약을 가진 일반 컬럼입니다.
`device_eui` UNIQUE 제약은 외부 ERD에는 명시되어 있지 않지만, 같은 장치가 두 재배에
중복 등록되는 것을 막아야 하는 우리 프로젝트의 기존 기능(장치 배치 등록, MQTT 토픽
`sensor/{deviceEui}` 라우팅)상 반드시 필요해 그대로 유지합니다. 삭제는 `is_deleted`
플래그로 처리합니다. `cultivation_id`는 병합 이후 같은 DB 안의 실제 FK입니다(이전에는
소프트 참조).

---

## cultivation_sensor_type

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_sensor_id | BIGINT | X | FK, `cultivation_sensor.id` |
| sensor_type_id | BIGINT | X | FK, `sensor_type.id` |

장치 하나가 여러 항목을 동시에 측정할 수 있어 `cultivation_sensor`와 1:N 관계입니다.
외부 ERD 기준으로 기존 브릿지 테이블 `sensor_type`을 `cultivation_sensor_type`으로
리네이밍하고, FK 컬럼도 `sensor_id`/`measurement_type_code`에서
`cultivation_sensor_id`/`sensor_type_id`로 바꿨습니다. `UNIQUE(cultivation_sensor_id,
sensor_type_id)`로 중복 등록을 막습니다.

---

## environment_setting

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | FK, `cultivation.id` |
| sensor_type_id | BIGINT | X | FK, `sensor_type.id` |
| threshold_min | NUMERIC(10,4) | X | 하한 |
| threshold_max | NUMERIC(10,4) | X | 상한 |
| created_at | DATETIME | X | 이 값이 저장된 시각 |
| updated_at | DATETIME | O | 생성 시각과 동일하게 유지됨 (UPDATE 경로 없음) |

사용자가 최종 저장한 환경값을 **범위**로, 항목별 행으로 저장합니다. 수정할 때도
UPDATE가 아니라 새 행을 INSERT합니다 — 이 테이블 하나가 "현재값"과 "이력"을 동시에
표현합니다. `cultivation_id`는 병합 이후 같은 DB 안의 실제 FK입니다.

`measurement_type_code` 컬럼은 `sensor_type_id`로 대체되었고, `threshold_unit`
컬럼은 제거했습니다 — 단위는 각 행마다 중복 저장할 필요 없이 `sensor_type_id`로
`sensor_type.value_unit`을 조회하면 되기 때문입니다(정규화 개선). 이에 따라 환경
설정을 조회할 때는 `sensor_type`과 JOIN해서 단위를 함께 가져와야 합니다. 자세한
내용은 [cultivation.md](../01_Domain/cultivation.md),
[environment-control.md](../04_sequence/environment-control.md),
[sensor-data.md](../04_sequence/sensor-data.md) 참고.

---

## mushroom_reference

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK. `cultivation.mushroom_id`가 FK로 참조 |
| mushroom_name_ko | VARCHAR(50) | X | 버섯 한글 이름 |
| mushroom_name_en | VARCHAR(50) | X | 버섯 영문 이름 |
| mushroom_scientific_name | VARCHAR(50) | O | 학명 |
| characteristics | TEXT | O | 특성 설명 |
| health_benefits | TEXT | O | 효능 |
| cultivation_guide | TEXT | O | 재배 가이드 |
| additional_info | TEXT | O | 추가 정보 |
| created_at | DATETIME | X | 생성일 |
| updated_at | DATETIME | O | 수정일 |

공공데이터 기반 5종 버섯의 고정 시드 데이터입니다. 자연키 코드 컬럼(예: `OYSTER`)은
두지 않고 대리키 `id`로만 식별합니다(ERD 기준, 이미 마이그레이션 완료). `characteristics`/
`health_benefits`/`cultivation_guide`/`additional_info` 텍스트 컬럼은 외부 ERD에는
없지만, AI Service의 챗봇/버섯 가이드 기능이 `mushroomId`로 정확히 일치하는 한 건을
그대로 조회해 LLM 컨텍스트(RAG 원문)로 사용하는 우리 프로젝트의 필수 기능이라 ERD
이상의 의도적 추가로 그대로 유지합니다. 5종 고정이라 별도의 임베딩·검색 색인 없이
직접 조회로 충분합니다.

---

## mushroom_reference_threshold

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| mushroom_reference_id | BIGINT | X | FK, `mushroom_reference.id` |
| sensor_type_id | BIGINT | X | FK, `sensor_type.id` |
| threshold_min | NUMERIC(10,4) | X | 하한 |
| threshold_max | NUMERIC(10,4) | X | 상한 |
| created_at | DATETIME | X | 생성일 |
| updated_at | DATETIME | O | 수정일 |

버섯 종류별 최적 생육 환경 범위를 항목별 행으로 저장합니다. `measurement_type_code`는
`sensor_type_id`로 대체되었고, `environment_setting`과 같은 이유로 `threshold_unit`
컬럼은 제거했습니다(단위는 `sensor_type.value_unit` 조회로 대체).
`UNIQUE(mushroom_reference_id, sensor_type_id)`로 중복 등록을 막습니다.

---

## inquiry_category

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| category_name | VARCHAR(100) | X | 카테고리 이름 |

시드 데이터: `id=1` "일반 문의", `id=2` "경작 문의". 기존 `inquiry.type`
(GENERAL/CULTIVATION) enum을 참조 테이블로 뺀 것으로, 향후 카테고리를 추가하려면
enum 값을 늘리는 배포 없이 행만 추가하면 됩니다.

---

## inquiry

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | X | 작성자 (Auth Service, 소프트 참조) |
| category_id | BIGINT | X | FK, `inquiry_category.id` |
| cultivation_id | BIGINT | O | "경작 문의"(category_id=2)일 때만 채워짐. 의도적으로 소프트 참조(FK 없음) |
| title | VARCHAR(200) | X | 제목 |
| status | VARCHAR(20) | X | 처리 상태 (PENDING/RESOLVED), 기본값 PENDING |
| created_at | DATETIME | X | 작성 일시 |

기존 3단계 상태(OPEN/ANSWERED/CLOSED)를 `PENDING`/`RESOLVED` 2단계로 축소했습니다.
일반 문의의 답변 완료와 경작 문의의 재배 삭제 처리 모두 `RESOLVED`로 통일하며, 실제
처리 내용은 `inquiry_answer` 행 유무 또는 재배 삭제 여부로 구분할 수 있으므로 상태값
자체는 단순화했습니다.

`cultivation_id`는 외부 ERD에는 없지만, "경작 문의 → 어떤 재배인지 식별해서 삭제
처리" 기능을 위해 의도적으로 추가 유지합니다. `category_id`가 "경작 문의"(`id=2`)일
때만 값이 채워지는 게 애플리케이션 레벨 불변식이며, DB CHECK로는 강제하지 않습니다
— `category_id`가 FK라 특정 값(2)을 하드코딩한 CHECK 제약은 카테고리가 데이터로
관리되는 설계 의도와 맞지 않다고 판단했습니다.

`admin_id`/`answered_at` 컬럼은 더 이상 `inquiry`에 두지 않습니다. 관리자가 누구인지/
언제 답변했는지는 `inquiry_answer.created_at`으로 대체됩니다. 외부 ERD의
`inquiry_answer`에도 `admin_id`가 없으며, 향후 필요해지면 `inquiry_answer`에
`admin_id`를 추가하는 것도 검토할 수 있으나 현재 스키마에는 넣지 않습니다.

---

## inquiry_answer

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| inquiry_id | BIGINT | X | FK, `inquiry.id` |
| content | TEXT | O | 최초 문의 본문 또는 사용자의 추가 질문 |
| answer_content | TEXT | O | 관리자 답변 |
| pre_id | BIGINT | O | 부모 스레드 행 참조 (self-FK). NULL이면 최초 문의 행 |
| created_at | DATETIME | X | 작성 일시 |

기존 `inquiry.answer` 단일 컬럼을 대체하는 스레드형 답변 테이블입니다. 문의 하나가
여러 개의 `inquiry_answer` 행(문의 본문 → 답변 → 추가 질문 → 재답변 ...)을 가질 수
있도록 `pre_id` self-FK로 체인을 구성합니다.

현재 범위에서 실제로 쓰이는 것은 두 종류뿐입니다.

- 문의 등록: `content`=문의 내용, `answer_content`=NULL, `pre_id`=NULL (최초 행)
- 일반 문의 답변: `content`=NULL, `answer_content`=답변 내용, `pre_id`=최초 행의 id

사용자가 답변에 대해 추가 질문을 남기고 관리자가 다시 답하는 스레드 확장은 스키마
차원에서는 가능하지만(같은 방식으로 `pre_id` 체인을 계속 이어가면 됨), 현재 범위는
기존 기능("관리자 답변 1회")과 동일하게 단순하게 다룹니다 — 과도하게 앞서 설계하지
않습니다. 경작 문의는 텍스트 답변이 없으므로 `inquiry_answer`에 답변 행을 추가하지
않고, `inquiry.status`만 `RESOLVED`로 바뀝니다.

---

# DDL

```sql
CREATE TABLE cultivation (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT NOT NULL,

    name VARCHAR(100) NOT NULL,

    mushroom_id BIGINT NOT NULL,

    status VARCHAR(20) NOT NULL DEFAULT 'CREATED',

    mode VARCHAR(20) NOT NULL DEFAULT 'GROWTH',

    started_at TIMESTAMP,

    finished_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uk_cultivation_user_name UNIQUE (user_id, name),

    CONSTRAINT chk_cultivation_status CHECK (status IN ('CREATED', 'RUNNING', 'FINISHED')),

    CONSTRAINT chk_cultivation_mode CHECK (mode IN ('GROWTH', 'HARVEST'))
);
```

```sql
CREATE TABLE cultivation_member (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    user_id BIGINT NOT NULL,

    role VARCHAR(20) NOT NULL DEFAULT 'MEMBER',

    joined_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_cultivation_member_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id) ON DELETE CASCADE,

    CONSTRAINT uk_cultivation_member UNIQUE (cultivation_id, user_id),

    CONSTRAINT chk_cultivation_member_role CHECK (role IN ('OWNER', 'MEMBER'))
);
```

```sql
CREATE TABLE harvest (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    harvest_weight NUMERIC(6,2),

    memo TEXT,

    harvested_at TIMESTAMP NOT NULL,

    product_score NUMERIC(4,1),

    product_grade VARCHAR(10),

    CONSTRAINT fk_harvest_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id) ON DELETE CASCADE,

    CONSTRAINT uk_harvest_cultivation UNIQUE (cultivation_id),

    CONSTRAINT chk_harvest_product_grade CHECK (product_grade IN ('TOP', 'HIGH', 'MID', 'LOW'))
);
```

```sql
CREATE TABLE cultivation_photo (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    object_key VARCHAR(500) NOT NULL,

    storage_type VARCHAR(10) NOT NULL DEFAULT 'MINIO',

    uploaded_at TIMESTAMP NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_cultivation_photo_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id) ON DELETE CASCADE,

    CONSTRAINT chk_cultivation_photo_storage_type CHECK (storage_type IN ('MINIO', 'LOCAL'))
);
```

```sql
CREATE TABLE sensor_type (
    id BIGSERIAL PRIMARY KEY,

    sensor_type VARCHAR(20) NOT NULL,

    value_unit VARCHAR(10) NOT NULL
);
```

```sql
CREATE TABLE cultivation_sensor (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    device_eui VARCHAR(32) NOT NULL,

    location VARCHAR(10),

    location_detail VARCHAR(50),

    device_model VARCHAR(100),

    status VARCHAR(10) NOT NULL DEFAULT 'OFFLINE',

    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_cultivation_sensor_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id) ON DELETE CASCADE,

    CONSTRAINT uk_cultivation_sensor_device_eui UNIQUE (device_eui),

    CONSTRAINT chk_cultivation_sensor_status CHECK (status IN ('ONLINE', 'OFFLINE', 'ERROR'))
);
```

```sql
CREATE TABLE cultivation_sensor_type (
    id BIGSERIAL PRIMARY KEY,

    cultivation_sensor_id BIGINT NOT NULL,

    sensor_type_id BIGINT NOT NULL,

    CONSTRAINT fk_cultivation_sensor_type_sensor
        FOREIGN KEY (cultivation_sensor_id) REFERENCES cultivation_sensor(id) ON DELETE CASCADE,

    CONSTRAINT fk_cultivation_sensor_type_type
        FOREIGN KEY (sensor_type_id) REFERENCES sensor_type(id),

    CONSTRAINT uk_cultivation_sensor_type UNIQUE (cultivation_sensor_id, sensor_type_id)
);
```

```sql
CREATE TABLE environment_setting (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    sensor_type_id BIGINT NOT NULL,

    threshold_min NUMERIC(10,4) NOT NULL,

    threshold_max NUMERIC(10,4) NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP,

    CONSTRAINT fk_environment_setting_cultivation
        FOREIGN KEY (cultivation_id) REFERENCES cultivation(id) ON DELETE CASCADE,

    CONSTRAINT fk_environment_setting_sensor_type
        FOREIGN KEY (sensor_type_id) REFERENCES sensor_type(id)
);
```

```sql
CREATE TABLE mushroom_reference (
    id BIGSERIAL PRIMARY KEY,

    mushroom_name_ko VARCHAR(50) NOT NULL,

    mushroom_name_en VARCHAR(50) NOT NULL,

    mushroom_scientific_name VARCHAR(50),

    characteristics TEXT,

    health_benefits TEXT,

    cultivation_guide TEXT,

    additional_info TEXT,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP
);
```

```sql
CREATE TABLE mushroom_reference_threshold (
    id BIGSERIAL PRIMARY KEY,

    mushroom_reference_id BIGINT NOT NULL,

    sensor_type_id BIGINT NOT NULL,

    threshold_min NUMERIC(10,4) NOT NULL,

    threshold_max NUMERIC(10,4) NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP,

    CONSTRAINT fk_mushroom_reference_threshold_reference
        FOREIGN KEY (mushroom_reference_id) REFERENCES mushroom_reference(id),

    CONSTRAINT fk_mushroom_reference_threshold_sensor_type
        FOREIGN KEY (sensor_type_id) REFERENCES sensor_type(id),

    CONSTRAINT uk_mushroom_reference_threshold UNIQUE (mushroom_reference_id, sensor_type_id)
);
```

```sql
ALTER TABLE cultivation
    ADD CONSTRAINT fk_cultivation_mushroom_id
    FOREIGN KEY (mushroom_id) REFERENCES mushroom_reference(id);
```

```sql
CREATE TABLE inquiry_category (
    id BIGSERIAL PRIMARY KEY,

    category_name VARCHAR(100) NOT NULL
);
```

```sql
CREATE TABLE inquiry (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT NOT NULL,

    category_id BIGINT NOT NULL,

    cultivation_id BIGINT,

    title VARCHAR(200) NOT NULL,

    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_inquiry_category
        FOREIGN KEY (category_id) REFERENCES inquiry_category(id),

    CONSTRAINT chk_inquiry_status CHECK (status IN ('PENDING', 'RESOLVED'))
);
```

```sql
CREATE TABLE inquiry_answer (
    id BIGSERIAL PRIMARY KEY,

    inquiry_id BIGINT NOT NULL,

    content TEXT,

    answer_content TEXT,

    pre_id BIGINT,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_inquiry_answer_inquiry
        FOREIGN KEY (inquiry_id) REFERENCES inquiry(id) ON DELETE CASCADE,

    CONSTRAINT fk_inquiry_answer_pre
        FOREIGN KEY (pre_id) REFERENCES inquiry_answer(id)
);
```

`mushroom_reference`가 `mushroom_reference_threshold`보다 먼저 생성되어야 하고,
`cultivation`이 `mushroom_reference` 생성 이후에 FK를 걸 수 있어 `cultivation.
mushroom_id` FK는 `ALTER TABLE`로 별도 추가합니다(시드 데이터 적재 순서 때문).
`inquiry_category`도 `inquiry`보다 먼저 생성 및 시드 적재가 되어야 합니다.
`inquiry.cultivation_id`/`inquiry.user_id`는 의도적으로 FK를 걸지 않습니다.

---

# Index

```sql
CREATE INDEX idx_cultivation_user
ON cultivation(user_id, created_at DESC);
```

사용자별 재배 목록 조회에 사용합니다.

```sql
CREATE INDEX idx_cultivation_member_user
ON cultivation_member(user_id);
```

`GET /cultivations/mine`(내가 OWNER/MEMBER로 속한 재배 목록 조회)에 사용합니다.
`UNIQUE(cultivation_id, user_id)` 제약이 만드는 인덱스는 `cultivation_id`가 앞에
와서 이 조회 패턴(사용자 기준)에는 그대로 쓸 수 없어 별도로 둡니다.

재배당 수확이 한 건뿐이라 `harvest.cultivation_id`의 `UNIQUE` 제약이 만드는 인덱스로
조회가 충분해, 별도 인덱스는 두지 않습니다.

```sql
CREATE INDEX idx_cultivation_photo_cultivation
ON cultivation_photo(cultivation_id, uploaded_at DESC);
```

재배별 최신 사진 조회에 사용합니다.

```sql
CREATE INDEX idx_cultivation_sensor_cultivation
ON cultivation_sensor(cultivation_id)
WHERE is_deleted = FALSE;
```

재배별 활성 센서 목록 조회에 사용하는 부분 인덱스입니다.

```sql
CREATE INDEX idx_environment_setting_lookup
ON environment_setting(cultivation_id, sensor_type_id, created_at DESC);
```

"현재값" 조회(재배+항목 기준 최신 행)와 이력 조회 모두에 사용합니다.

```sql
CREATE INDEX idx_mushroom_reference_threshold_reference
ON mushroom_reference_threshold(mushroom_reference_id);
```

버섯 하나의 전체 임계값 조회에 사용합니다.

```sql
CREATE INDEX idx_inquiry_user
ON inquiry(user_id, created_at DESC);
```

사용자 본인의 문의 목록 조회에 사용합니다.

```sql
CREATE INDEX idx_inquiry_status
ON inquiry(status, created_at DESC);
```

관리자의 미처리 문의 큐 조회에 사용합니다.

```sql
CREATE INDEX idx_inquiry_cultivation
ON inquiry(cultivation_id)
WHERE cultivation_id IS NOT NULL;
```

특정 재배에 대한 경작 문의 조회에 사용합니다.

```sql
CREATE INDEX idx_inquiry_answer_inquiry
ON inquiry_answer(inquiry_id, created_at);
```

문의 하나의 스레드(문의 본문 → 답변 → ...)를 시간순으로 조회하는 데 사용합니다.

---

# 관계

```
harvest, cultivation_photo, cultivation_sensor, environment_setting, cultivation_member
    │ cultivation_id (같은 DB 내 실제 FK, ON DELETE CASCADE)
    ▼
cultivation
    │ mushroom_id (같은 DB 내 실제 FK)
    ▼
mushroom_reference

cultivation_sensor_type
    │ cultivation_sensor_id (실제 FK), sensor_type_id (실제 FK)
    ▼
cultivation_sensor, sensor_type

environment_setting
    │ sensor_type_id (실제 FK)
    ▼
sensor_type

mushroom_reference_threshold
    │ mushroom_reference_id, sensor_type_id (실제 FK)
    ▼
mushroom_reference, sensor_type

inquiry
    │ user_id (Auth Service, 소프트 참조)
    │ category_id (실제 FK)
    │ cultivation_id (같은 DB지만 의도적 소프트 참조)
    ▼
inquiry_category, cultivation (참조는 하되 FK 없음)

inquiry_answer
    │ inquiry_id (실제 FK), pre_id (self-FK)
    ▼
inquiry, inquiry_answer
```

Cultivation Service는 병합 이전 Sensor Service와 데이터베이스를 공유하게 되어,
`cultivation_id`/`mushroom_id` 참조가 소프트 참조에서 실제 FK로 바뀌었습니다.
`user_id`(Auth Service)는 여전히 다른 서비스 DB를 가리키는 소프트 참조입니다.
`cultivation_member.user_id`도 마찬가지로 Auth Service를 가리키는 소프트 참조입니다.

---

# 고려 사항

- 재배 이름 유일성은 전역이 아니라 사용자 단위입니다.
- 버섯 종류는 자연키 코드 컬럼 없이 대리키 `id`(`mushroom_id`)로만 식별합니다(외부
  ERD 기준). 프론트엔드에서 버섯 이름을 표시할 때는 `GET /mushroom-references`(5종
  고정) 목록을 조회해 id→이름 매핑을 직접 만들어 사용하며, 재배 생성/조회 등 다른
  API 응답에는 버섯 이름을 별도로 포함하지 않습니다.
- `mode`가 `HARVEST`로 바뀌어도 `environment_setting`(목표 환경 범위)은 자동으로
  바뀌지 않습니다 — 수확기 환경 임계값 데이터를 아직 확보하지 못해, 현재는 `mode`가
  상태 표시와 알림 트리거 역할만 합니다. 데이터가 준비되면 `mushroom_reference_threshold`에
  생육기/수확기 구분을 추가하고, `mode`가 `HARVEST`로 바뀔 때 `environment_setting`도
  같이 자동 전환하는 것을 추후 개발할 예정입니다.
- `cultivation_member`는 재배 생성과 같은 트랜잭션에서 생성자를 OWNER로 자동 등록합니다.
  `cultivation.user_id`는 이 테이블 도입 이후에도 그대로 두며, 항상 그 재배의 OWNER
  행과 같은 값을 가집니다 — 기존에 `user_id`를 참조하던 부분(재배 이름 유일성, 소유권
  확인 API 등)을 다시 손댈 필요가 없도록 하기 위한 의도적인 중복입니다.
- `cultivation_member`의 권한 모델(OWNER 전체 권한, MEMBER 조회만 가능)은 우선 가정한
  것으로, 세부 정책은 추후 조정될 수 있습니다.
- `CREATED → RUNNING` 전환은 목표 환경 저장과 같은 트랜잭션 안에서 처리됩니다(병합
  이전에는 `EnvironmentRangeUpdatedEvent`를 스스로 구독하는 방식이었으나, 같은 서비스
  안이 되면서 불필요해졌습니다). Rule Engine Service를 위한 이벤트 발행 자체는
  계속됩니다.
- 수확이 기록되면 `HarvestCompletedEvent`가 발행되어, AI Service가 이를 구독해
  "인사이트" 사례를 즉시 적재하고, 상품 등급 원점수(`product_score`)를 계산해
  Cultivation Service에 돌려줍니다. `product_grade`는 그 원점수를 받은 Cultivation
  Service가 매깁니다. 이 콜백 호출이 실패하면 `product_score`/`product_grade`는
  NULL로 남으며, 재시도 정책은 추후 정합니다.
- 사진 저장소(`storage_type`)를 MinIO에서 로컬로(또는 그 반대로) 전환해도 기존 행을
  다시 쓸 필요가 없습니다.
- `cultivation_sensor`/`environment_setting`은 병합 이전 소프트 참조였던
  `cultivation_id`가 실제 FK(`ON DELETE CASCADE`)로 바뀌어, 재배 삭제 시 별도의
  보상 이벤트 없이 DB 트랜잭션만으로 함께 정리됩니다.
- `cultivation_sensor.device_eui` UNIQUE 제약은 외부 ERD에는 명시되어 있지 않지만,
  같은 장치가 두 재배에 중복 등록되는 것을 막아야 하는 우리 프로젝트의 기존 기능상
  반드시 필요해 유지합니다.
- `sensor_type`을 별도 참조 테이블(대리키)로 뺀 이유는 온도/습도/CO₂/조도라는 같은
  값 집합이 여러 테이블에서 자유 문자열로 반복되지 않도록 하기 위함입니다. 기존
  자연키(`code`)를 대리키(`id`)로 바꾼 것은 외부 ERD와의 정합을 맞추기 위함입니다.
- `environment_setting`/`mushroom_reference_threshold`의 `threshold_unit` 컬럼은
  제거했습니다. 단위가 `sensor_type_id`를 통해 `sensor_type.value_unit`에서 항상
  조회 가능해 각 행마다 중복 저장할 필요가 없기 때문입니다(정규화 개선). 조회 시
  `sensor_type`과의 JOIN이 필요해졌다는 점에 유의해야 합니다.
- `inquiry.cultivation_id`만 예외적으로 FK를 걸지 않습니다. 재배가 삭제된 뒤에도
  "어떤 재배에 대한 문의였는지" 기록이 남아야 하기 때문입니다.
- `inquiry`를 `inquiry`/`inquiry_category`/`inquiry_answer` 세 테이블로 분리한 것은
  외부 ERD 기준입니다. 카테고리를 참조 테이블로 빼 향후 카테고리 추가를 데이터
  변경만으로 가능하게 했고, 답변을 스레드형 테이블로 분리해 "문의 → 답변 →
  추가 질문" 확장 여지를 열어뒀습니다(현재 범위에서는 관리자 답변 1회만 사용).
