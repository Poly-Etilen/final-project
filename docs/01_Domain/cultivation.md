# Cultivation Service

## 역할

Cultivation Service는 사용자의 버섯 재배 정보를 관리하는 핵심 서비스입니다.

사용자는 새로운 재배를 생성하고, 재배 진행 상황과 수확 결과를 관리할 수 있습니다. 재배에
연결되는 센서 "장치"의 등록/조회/삭제와 목표 환경(위험 한계값) 저장/조회는 Sensor Service의
책임입니다.

> ℹ️ **변경 이력**: 센서 장치 CRUD는 원래 DatasourceGenerator → Cultivation Service 순으로
> 옮겨왔지만, 팀 회의 결과 `sensor`/`environment_setting` 테이블과 관련 API를 Sensor
> Service로 완전히 이관했습니다. Sensor Service가 이미 센서 측정값(InfluxDB/Redis)과
> 통계/차트/리포트를 전담하고 있어, 센서 장치 메타데이터와 목표 환경까지 함께 소유하는 것이
> "database per service" 원칙에 더 맞는다고 판단했습니다. Cultivation Service는 재배 생성
> 시 `devices`를 함께 등록하는 사용자 흐름을 유지하기 위해 Sensor Service를 OpenFeign으로
> 호출합니다. (자세한 내용은 [sensor.md](./sensor.md),
> [cultivation-db.md](../03_Database/cultivation-db.md)의 변경 이력,
> [README.md](../README.md)의 결정 사항 #24 참고)

---

# 책임

- 재배 생성 (Sensor Service 호출을 통한 센서 장치 함께 등록 포함)
- 재배 조회
- 재배 수정
- 재배 삭제
- 재배 종료
- 수확 정보 저장
- 재배 이력 관리
- 생육 사진 업로드
- 생육 사진 이력 관리
- 재배 소유권 확인 API 제공 (Sensor Service가 쓰기 요청 시 호출)
- 수확 임베딩 여부 관리 (인사이트 기능용)

---

# 주요 기능

## 재배 생성

새로운 버섯 재배를 생성합니다.

사용자는

- 재배 이름
- 버섯 종류
- 등록할 센서 장치 목록 (선택, device_eui/place/location/device_model/sensor_type)

를 입력합니다.

> ℹ️ **변경 이력**: 원래 센서 장치 등록은 재배 생성과 완전히 분리된 별도 API였지만, 실제
> 사용자는 재배를 만드는 화면에서 연결할 디바이스도 함께 선택하는 경우가 많아 재배 생성
> 요청에 디바이스 목록을 포함할 수 있도록 확장했습니다. 당시에는 재배 생성과 디바이스 등록이
> 하나의 로컬 트랜잭션으로 처리되었습니다.

> ℹ️ **변경 이력**: `sensor` 테이블이 Sensor Service로 이관되면서 더 이상 하나의 로컬
> 트랜잭션으로 처리할 수 없습니다. `devices`로 센서를 함께 등록하는 사용자 흐름 자체는
> 그대로 유지하기로 했지만(팀 회의 결정), Cultivation Service는 cultivation 행을 생성한 뒤
> Sensor Service의 배치 등록 엔드포인트를 OpenFeign으로 동기 호출하고, 실패하면 방금
> 생성한 cultivation을 보상 삭제(compensating delete)합니다. 디바이스는 여전히 선택
> 항목이며, 생략하면 이후 Sensor Service의 개별 등록 API로 추가할 수 있습니다. (자세한
> 내용은 [sensor.md](./sensor.md) 참고)

이후 자체 보유한 `mushroom_reference` 참조 테이블(공공데이터 기반 5종 버섯의 최적 환경 범위)을
버섯 종류로 조회하여 추천값을 반환합니다. AI Service를 호출하지 않습니다.

> ℹ️ **변경 이력**: `mushroom_reference`에 이름(한글/영문/학명), 특성, 효능, 재배 가이드, 추가
> 정보 컬럼이 추가되었습니다. 이 원문 텍스트는 재배 생성 응답에는 포함되지 않고, AI Service가
> "버섯 가이드" 기능에서 `GET /api/v1/mushroom-references/{mushroomType}`(내부용)로 조회해
> LLM 프롬프트의 RAG 컨텍스트로 사용합니다. (자세한 내용은 [ai.md](./ai.md) 참고)

> ℹ️ **변경 이력**: 원래는 AI Service(Embedding/Vector Search/LLM)를 호출해 추천값을 생성했지만,
> 버섯 종류가 공공데이터 기준 5가지로 고정되어 있어 항상 동일한 값이 나오는 조회에는 AI가
> 불필요하다고 판단, Cultivation Service가 직접 보유한 참조 테이블 조회로 단순화했습니다.

사용자가 추천 환경을 수정한 후 저장하는 과정(환경 설정 저장)은 더 이상 Cultivation Service의
책임이 아닙니다. Sensor Service가 목표 환경(위험 한계값)을 소유하며, 저장 방식(단일값 →
범위 변환, INSERT-only 이력 등)도 그대로 이전되었습니다. 재배 상태를 `CREATED`에서
`RUNNING`으로 전환하는 시점("환경 저장 = 재배 시작") 역시 이제 Sensor Service가 발행하는
`EnvironmentRangeUpdatedEvent`를 Cultivation Service가 구독해 처리하는 이벤트 기반 흐름으로
바뀌었습니다(더 이상 하나의 로컬 트랜잭션이 아님). (자세한 내용은 [sensor.md](./sensor.md),
[sensor-db.md](../03_Database/sensor-db.md), 아래 "구독 이벤트" 참고)

---

## 재배 목록 조회

사용자가 생성한 모든 재배 목록을 조회합니다.

조회 정보

- 재배 이름
- 버섯 종류
- 현재 상태
- 생성일

---

## 재배 상세 조회

재배의 상세 정보를 조회합니다.

조회 정보

- 재배 이름
- 재배 상태
- 생성일
- 수정일

> ℹ️ **변경 이력**: 응답에 포함되던 환경 설정(목표값)과 센서 상태는 `environment_setting`/
> `sensor`가 Sensor Service로 이관되면서 제거했습니다. Cultivation Service가 매 조회마다
> Sensor Service를 호출해 조합하지 않고, 클라이언트가 필요 시 Sensor Service의 조회 API를
> 직접 호출하도록 책임을 분리했습니다. (자세한 내용은 [sensor.md](./sensor.md) 참고)

---

## 재배 종료

재배를 종료합니다.

종료 시

- 종료일 저장
- 상태 변경
- 수확 정보 입력

---

## 수확 기록 저장

재배 종료 후

- 수확량
- 메모

를 저장합니다.

---

## 재배 이력 조회

완료된 재배 목록을 조회합니다.

---

## 생육 사진 업로드

사용자가 직접 촬영한 사진을 업로드하여 생육 상태를 분석합니다.

카메라 센서가 아닌, 사용자가 앱/웹에서 직접 촬영한 사진을 업로드하는 방식입니다.

사진은 MinIO에 저장되고, Cultivation Service는 AI Service를 호출하여
학습된 Vision 모델로 사진을 분석한 결과(균사 성장률, 갓 크기, 색상, 병충해 여부)를 함께 반환합니다.

분석 결과는 즉시 반환되며, 최근 분석 결과는 캐시되어 재조회할 수 있습니다.

---

## 생육 사진 이력 조회

재배별로 업로드된 사진 목록을 조회합니다.

---

## 재배 소유권 확인 (내부용)

센서/환경 관련 쓰기 요청을 처리하기 전, Sensor Service가 요청자가 해당 cultivation의 소유자가
맞는지 확인할 수 있도록 `userId`와 `status`를 반환하는 내부용 API를 제공합니다.

> ℹ️ **변경 이력**: `sensor`/`environment_setting`이 Sensor Service로 이관되면서 Sensor
> Service는 더 이상 `cultivation.user_id`에 직접 접근할 수 없게 되었습니다. 이 API를 신설해
> 소유권 확인 책임을 대신합니다. 단, Cultivation Service가 재배 생성 시 호출하는 배치
> 등록은 호출자가 Cultivation Service 자신이므로 이 확인을 거치지 않습니다. (자세한 내용은
> [sensor.md](./sensor.md), [sensor-db.md](../03_Database/sensor-db.md) 참고)

재배 삭제 시에는 `CultivationDeletedEvent`를 발행해 Sensor Service가 해당 cultivation_id의
sensor/environment_setting을 정리(보상 삭제)하도록 합니다. 서로 다른 DB이므로 DB 레벨
`ON DELETE CASCADE`를 쓸 수 없기 때문입니다.

---

## 수확 임베딩 여부 관리 (인사이트 기능용)

"인사이트" 기능(같은 버섯 종류 + 유사한 온도로 재배했던 타인의 사례를 바탕으로 피드백을 주는
기능)을 위해, 어떤 수확 건이 Elasticsearch에 임베딩되었는지를 `harvest.is_embedded` 플래그로
관리합니다.

Cultivation Service는 임베딩 자체(텍스트 요약 생성, 벡터화, Elasticsearch 저장)를 수행하지
않으며, AI Service가 배치 처리를 위해 조회/갱신할 수 있는 내부용 API 3종만 제공합니다.

- 미임베딩 건수 조회 (`GET /api/v1/harvests/unembedded-count`)
- 미임베딩 목록 조회 (`GET /api/v1/harvests/unembedded`)
- 임베딩 완료 처리 (`PATCH /api/v1/harvests/embedded`)

> ℹ️ **변경 이력**: 미임베딩 목록 응답에 포함되던 환경 평균(기간 가중 평균)과, "현재" 재배의
> 환경 평균을 조회하는 별도 API(`GET /api/v1/cultivations/{cultivationId}/environment-average`)는
> `environment_setting`이 Sensor Service로 이관되면서 더 이상 Cultivation Service가 계산할
> 수 없어 제거했습니다. AI Service의 인사이트 배치 스케줄러는 이제 Cultivation Service(미임베딩
> 목록)와 Sensor Service(환경 평균 일괄 조회)를 각각 호출해 조합합니다. (자세한 내용은
> [ai.md](./ai.md), [sensor.md](./sensor.md), [insight.md](../04_sequence/insight.md) 참고)

> ℹ️ **변경 이력**: "인사이트" 기능 추가를 위해 신설했습니다. AI Service가 별도의 워터마크를
> 관리하는 대신 Cultivation Service가 "임베딩 여부"를 단순 boolean 컬럼(`harvest.is_embedded`)으로
> 소유하고, AI Service는 조회/갱신만 하는 방식으로 설계했습니다.

---

# API

## 재배 생성

POST /cultivations

---

## 재배 목록 조회

GET /cultivations

---

## 재배 상세 조회

GET /cultivations/{cultivationId}

---

## 재배 수정

PATCH /cultivations/{cultivationId}

---

## 재배 삭제

DELETE /cultivations/{cultivationId}

---

## 재배 종료

PATCH /cultivations/{cultivationId}/finish

---

## 수확 정보 저장

PATCH /cultivations/{cultivationId}/harvest

---

## 재배 이력 조회

GET /cultivations/history

---

## 생육 사진 업로드 (+분석)

POST /cultivations/{cultivationId}/photos

사진을 업로드하면 동시에 AI Vision 분석을 수행하여 결과를 함께 반환합니다.

---

## 생육 사진 이력 조회

GET /cultivations/{cultivationId}/photos

---

## 최근 생육 분석 결과 조회

GET /cultivations/{cultivationId}/analysis

새로 분석을 수행하지 않고, 가장 최근 분석 결과(캐시)를 반환합니다.

---

> ℹ️ **변경 이력**: 센서 등록/목록조회/상세조회/삭제(`POST`/`GET`/`GET`/`DELETE
> /cultivations/{cultivationId}/sensors...`) API는 `sensor` 테이블과 함께 Sensor Service로
> 완전히 이관되어 이 문서에서 제거되었습니다. [sensor.md](./sensor.md)를 참고하세요.

---

## 재배 소유권 확인 (내부용)

GET /api/v1/cultivations/{cultivationId}/owner

Sensor Service가 센서/환경 쓰기 요청 처리 전 소유권을 확인하기 위해서만 호출합니다.

---

## 버섯 참조 데이터 조회 (내부용)

GET /api/v1/mushroom-references/{mushroomType}

AI Service가 "버섯 가이드" 생성 시에만 호출합니다. 자세한 내용은
[cultivation-api.md](../02_API/cultivation-api.md) 참고.

---

## 버섯 참조 데이터 전체 목록 조회 (내부용)

GET /api/v1/mushroom-references

Embedding Service가 Elasticsearch 전체 재생성 시에만 호출합니다.

---

## 미임베딩 수확 건수 조회 (내부용)

GET /api/v1/harvests/unembedded-count

AI Service가 00시 배치 스케줄러에서 임계치(20건) 도달 여부를 판단하기 위해 호출합니다.

---

## 미임베딩 수확 목록 조회 (내부용)

GET /api/v1/harvests/unembedded

AI Service가 배치 임베딩 대상 데이터를 가져오기 위해 호출합니다. 환경 평균은 더 이상 이
응답에 포함되지 않으며, AI Service가 Sensor Service를 별도로 호출해 채웁니다.

---

## 수확 임베딩 완료 처리 (내부용)

PATCH /api/v1/harvests/embedded

AI Service가 Embedding Service 호출에 성공한 직후 호출합니다.

---

> ℹ️ **변경 이력**: 재배 환경 평균 조회(`GET /api/v1/cultivations/{cultivationId}/environment-average`)는
> `environment_setting`과 함께 Sensor Service로 이관되었습니다. [sensor.md](./sensor.md)를
> 참고하세요.

---

# Database

Cultivation Service는 별도의 PostgreSQL Database를 사용합니다.

### Table

- mushroom_reference (공공데이터 기반 5종 최적 환경 범위, 전역 참조 데이터)
- cultivation
- harvest
- photo

> ℹ️ **변경 이력**: `environment_setting`과 `sensor`는 Sensor Service 소유의 새 DB로
> 이관되어 더 이상 이 Database에 없습니다. (자세한 내용은
> [sensor-db.md](../03_Database/sensor-db.md) 참고)

---

# Redis

현재 사용하지 않습니다.

---

# MinIO

사용자가 업로드한 생육 사진 원본을 저장합니다.

Bucket

```
mushroom-photos
```

Key 구조

```
{cultivationId}/{uploadedAt}.jpg
```

---

# 다른 서비스와의 통신

## 호출하는 서비스

### AI Service

생육 사진 Vision 분석 요청

AI 리포트 조회

환경 추천은 더 이상 AI Service를 호출하지 않고, Cultivation Service가 mushroom_reference를 직접 조회합니다.

---

### Sensor Service

재배 생성 시 `devices`가 있으면 배치 등록을 위해 호출합니다
(`POST /api/v1/sensors/cultivations/{cultivationId}/batch`). 실패하면 방금 생성한
cultivation을 보상 삭제합니다.

> ℹ️ **변경 이력**: 이전에는 재배 상세 조회 시 표시할 "현재 센서 상태/환경 통계/실시간 센서
> 데이터"를 조회하기 위해 Sensor Service를 호출했지만, 재배 상세 조회 응답에서 해당 필드들을
> 제거하면서 이 호출도 없앴습니다. 클라이언트가 필요 시 Sensor Service를 직접 호출합니다.
> (자세한 내용은 위 "재배 상세 조회" 참고)

---

## 호출받는 서비스

API Gateway

### Sensor Service

센서/환경 쓰기 요청 처리 전 소유권 확인을 위해 `GET /api/v1/cultivations/{cultivationId}/owner`를
호출합니다. (배치 등록 호출은 예외 — 호출자가 Cultivation Service 자신이므로 확인하지 않습니다.)

> ℹ️ **변경 이력**: Rule Engine Service의 목표 환경 범위 fallback 조회와 DatasourceGenerator의
> 재시작 시 전체 센서 목록 조회는 `environment_setting`/`sensor`가 Sensor Service로 이관되면서
> 더 이상 Cultivation Service를 호출하지 않습니다. 이제 둘 다 Sensor Service를 직접 호출합니다.
> (자세한 내용은 [rule-Engine.md](./rule-Engine.md), [datasource-generator.md](./datasource-generator.md) 참고)

### AI Service

"버섯 가이드" 생성 시 `GET /api/v1/mushroom-references/{mushroomType}`로 mushroom_reference의
특성/효능/재배 가이드/추가 정보를 조회합니다. LLM 프롬프트의 RAG 컨텍스트로만 사용되며,
Cultivation Service는 이 값을 가공하지 않고 그대로 반환합니다.

"인사이트" 기능을 위해 `GET /api/v1/harvests/unembedded-count`, `GET /api/v1/harvests/unembedded`,
`PATCH /api/v1/harvests/embedded`도 호출합니다(환경 평균 조회는 Sensor Service로 이관되어
더 이상 Cultivation Service를 호출하지 않습니다). (자세한 내용은 [ai.md](./ai.md),
[insight.md](../04_sequence/insight.md) 참고)

### Embedding Service

Elasticsearch 전체 재생성이 필요할 때만 `GET /api/v1/mushroom-references`(전체 목록)를
호출합니다. 평상시에는 `MushroomReferenceUpdatedEvent`로만 동기화하므로 이 호출은 드뭅니다.

---

# Event

## 발행 이벤트

### CultivationCreatedEvent

재배 생성 완료

---

### CultivationFinishedEvent

재배 종료

---

### HarvestCompletedEvent

수확 완료

---

### CultivationDeletedEvent

재배를 삭제할 때 발행합니다.

```json
{
    "cultivationId": 3,
    "deletedAt": "2026-08-15T09:00:00"
}
```

구독 서비스: Sensor Service (해당 cultivation_id의 sensor/environment_setting을 정리하는
보상 삭제용. DB가 분리되어 `ON DELETE CASCADE`를 쓸 수 없기 때문에 이벤트로 대체합니다.)

> ℹ️ **변경 이력**: `EnvironmentRangeUpdatedEvent`/`SensorRegisteredEvent`/`SensorDeletedEvent`는
> 발행 주체가 Sensor Service로 바뀌면서 이 문서에서 제거되었습니다. 대신 위
> `CultivationDeletedEvent`를 새로 추가했습니다. (자세한 내용은 [sensor.md](./sensor.md),
> [sensor-db.md](../03_Database/sensor-db.md) 참고)

---

### MushroomReferenceUpdatedEvent

관리자가 `mushroom_reference`를 등록/수정할 때 발행합니다. 버섯 종류가 5종으로 고정된 정적
데이터라 매우 드물게 발생합니다.

```json
{
    "mushroomType": "OYSTER",
    "mushroomNameKo": "느타리버섯",
    "mushroomNameEn": "Oyster Mushroom",
    "mushroomScientificName": "Pleurotus ostreatus",
    "characteristics": "군생하며 갓은 회갈색~담회색을 띠고, 균사 성장 속도가 빠른 편입니다.",
    "healthBenefits": "식이섬유와 베타글루칸이 풍부해 면역력 강화와 콜레스테롤 감소에 도움을 줍니다.",
    "cultivationGuide": "다습한 환경을 선호하지만 환기가 부족하면 곰팡이가 발생하기 쉬우니 CO₂ 농도 관리에 유의해야 합니다.",
    "additionalInfo": null,
    "updatedAt": "2026-08-15T09:00:00"
}
```

구독 서비스: Embedding Service (텍스트를 임베딩해 Elasticsearch의 mushroom_environment 인덱스 갱신)

---

## 구독 이벤트

### EnvironmentRangeUpdatedEvent

Sensor Service가 environment_setting을 생성/수정할 때 발행합니다. cultivation 상태가 아직
`CREATED`라면 이를 계기로 `RUNNING`으로 전환합니다. 이미 `RUNNING`/`FINISHED`인 재배가
환경을 재수정하는 경우(진행 중 목표값 조정)에는 상태를 변경하지 않습니다.

> ℹ️ **변경 이력**: 원래 "환경 저장 + 재배 상태 전환(CREATED → RUNNING)"은 Cultivation
> Service가 하나의 로컬 트랜잭션으로 처리했습니다. `environment_setting`이 Sensor Service로
> 이관되면서 더 이상 같은 DB 트랜잭션으로 묶을 수 없어, Cultivation Service가 Sensor
> Service의 `EnvironmentRangeUpdatedEvent`를 구독해 상태를 전환하는 방식으로 바꿨습니다.
> (자세한 내용은 [sensor.md](./sensor.md), [create-cultivation.md](../04_sequence/create-cultivation.md) 참고)

> ℹ️ **변경 이력**: `SensorErrorEvent`(Rule Engine Service 발행, sensor.status 갱신용) 구독은
> `sensor` 테이블이 Sensor Service로 이관되면서 함께 옮겨갔습니다. 이제 Rule Engine Service의
> `SensorErrorEvent`는 Sensor Service가 구독합니다.

---

# Sequence

## 재배 생성

Client

↓

Gateway

↓

Cultivation Service

↓

Cultivation 생성

↓

devices가 있다면 Sensor Service에 OpenFeign 호출
(배치 등록, `POST /api/v1/sensors/cultivations/{cultivationId}/batch`)

↓

실패 시 Cultivation 보상 삭제 + 에러 응답 / 성공 시 다음 단계로

↓

mushroom_reference 조회 (버섯 종류 기준)

↓

환경 추천 반환

↓

Client

↓

(선택) 버섯 가이드 조회 — Client가 AI Service를 직접 호출 (자세한 내용은 [ai.md](./ai.md) 참고)

↓

사용자 수정

↓

Cultivation Service

↓

PostgreSQL 저장

---

## 재배 종료

Client

↓

Gateway

↓

Cultivation Service

↓

Harvest 저장

↓

재배 종료

↓

Event 발행

> ℹ️ **변경 이력**: "센서 등록" 시퀀스(Cultivation Service가 sensor 레코드를 생성하고
> SensorRegisteredEvent를 발행하던 흐름)는 `sensor` 테이블이 Sensor Service로 이관되면서
> 이 문서에서 제거되었습니다. [sensor.md](./sensor.md)를 참고하세요.

---

# 예외 상황

- 존재하지 않는 재배
- 이미 종료된 재배
- 권한 없는 재배 접근
- 지원하지 않는 버섯 종류 (mushroom_reference에 없음)
- 재배 생성 시 Sensor Service 배치 등록 실패 (device_eui 중복 등 포함) — 재배 생성 자체가 취소되며, 방금 생성한 cultivation을 보상 삭제
- 사진 업로드 실패
- 지원하지 않는 파일 형식
- Vision 분석 실패

---

# 추후 개발 예정

- 재배 템플릿 저장
- 즐겨찾는 환경 저장
- 자동 재배 스케줄 설정
- 재배 목표 설정