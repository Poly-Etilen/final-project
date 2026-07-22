# Cultivation Service

## 역할

Cultivation Service는 사용자의 버섯 재배 자체를 관리하는 서비스입니다. 재배 생성/조회/
종료, 재배당 여러 번 발생하는 수확(flush) 기록, 생육 사진 업로드를 담당합니다. 센서
장치·목표 환경값·버섯 참조 데이터는 Sensor Service가 소유하며, Cultivation Service는
재배 생성 시 Sensor Service를 OpenFeign으로 호출해 센서를 등록합니다.

---

# 책임

- 재배 생성/조회/이력 조회/종료
- 수확(flush) 기록 저장 (재배당 여러 번 가능)
- 생육 사진 업로드
- 재배 소유권 확인 API 제공 (내부용, 다른 서비스가 호출)

---

# 주요 기능

## 재배 생성

사용자가 재배 이름, 버섯 종류(`mushroomType`), 등록할 센서 장치 목록을 지정해 재배를
생성합니다. 재배 이름은 같은 사용자 안에서만 유일하면 됩니다. `cultivation` 행을 먼저
만든 뒤 Sensor Service의 배치 등록 엔드포인트를 OpenFeign으로 호출해 센서를 등록하고,
이 호출이 실패하면 방금 만든 `cultivation` 행을 보상 삭제합니다. 재배 상태는 `CREATED`로
시작해, Sensor Service가 목표 환경을 처음 저장하면 `RUNNING`으로 자동 전환됩니다.

---

## 수확(flush) 기록 저장

재배가 `RUNNING`인 동안 수확이 있을 때마다 기록합니다. 여러 번 반복 호출할 수 있으며,
재배 상태는 바뀌지 않습니다. 서버가 자동으로 순번(`flush_no`)을 채번합니다.

---

## 재배 종료

재배 상태를 `FINISHED`로 바꿉니다. 수확 기록과는 별개의 API로, 더 이상 수확 정보를 받지
않습니다.

---

## 생육 사진 업로드

사용자가 촬영한 사진을 업로드하면 저장소(MinIO 또는 로컬)에 저장하고 `photo` 메타데이터를
남깁니다. AI Service가 이 사진으로 Vision 분석을 수행합니다.

---

## 재배 소유권 확인 (내부용)

Sensor Service 등 다른 서비스가 쓰기 작업 전에 요청자가 특정 재배의 소유자인지 확인할 수
있도록 내부용 API를 제공합니다.

---

# API

## 재배 생성

POST /cultivations

---

## 재배 목록/상세 조회

GET /cultivations

GET /cultivations/{id}

---

## 재배 이력 조회

GET /cultivations/history

---

## 재배 종료

PATCH /cultivations/{id}/finish

---

## 수확 기록 저장/조회

POST /cultivations/{id}/harvests

GET /cultivations/{id}/harvests

---

## 사진 업로드

POST /cultivations/{id}/photos

---

## 재배 소유권 확인 (내부용)

GET /cultivations/{id}/owner

---

# Database

Cultivation Service는 하나의 PostgreSQL Database를 사용합니다.

## Table

- cultivation (UNIQUE(user_id, name), status CREATED/RUNNING/FINISHED)
- harvest (재배당 여러 건, UNIQUE(cultivation_id, flush_no))
- photo (object_key + storage_type)

자세한 내용은 [cultivation-db.md](../03_Database/cultivation-db.md) 참고.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Sensor Service

- 재배 생성 시 센서 배치 등록 (`POST /api/v1/sensors/cultivations/{cultivationId}/batch`)
- 재배 생성 시 버섯 참조 데이터 조회 (`GET /api/v1/mushroom-references/{mushroomType}`, 환경 추천용)

---

## 호출받는 서비스

### Sensor Service

- 재배 소유권 확인 (`GET /cultivations/{cultivationId}/owner`)

### AI Service

- 생육 사진 조회 (Vision 분석용, MinIO/로컬 경유)

### API Gateway

- 재배/수확/사진 관련 REST API 요청

---

# Event

## Publish

### CultivationDeletedEvent

재배 삭제 시 발행. Sensor Service가 구독해 해당 재배의 `sensor`/`environment_setting`을
정리합니다.

### HarvestCompletedEvent

수확 기록 저장 시 발행. Notification Service가 구독해 알림을 보내고, AI Service가
구독해 "인사이트" 사례를 적재합니다.

### CultivationFinishedEvent

재배 종료 시 발행. Notification Service가 구독해 알림을 보냅니다.

---

## Subscribe

### EnvironmentRangeUpdatedEvent

Sensor Service가 목표 환경을 처음 저장하면 발행. 해당 재배가 `CREATED`이면 `RUNNING`으로
전환합니다(멱등).

### UserDeletedEvent

Auth Service가 회원 탈퇴 시 발행. 해당 사용자의 재배 데이터를 비활성화합니다.

---

# Sequence

관련 시퀀스는 [create-cultivation.md](../04_sequence/create-cultivation.md),
[harvest.md](../04_sequence/harvest.md),
[growth-analysis.md](../04_sequence/growth-analysis.md) 참고.

---

# 예외 상황

- 재배 이름 중복 (같은 사용자 안에서)
- 존재하지 않는 재배
- 다른 사용자의 재배에 대한 접근 시도
- Sensor Service 배치 등록 실패 (재배 생성 롤백)
- `FINISHED` 상태 재배에 대한 수확 기록/사진 업로드 시도
- 사진 저장소 업로드 실패

---

# 추후 개발 예정

- 재배 공유(여러 사용자가 같은 재배에 접근)
