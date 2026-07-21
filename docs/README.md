# 🍄 EcoSphere / 버섯 재배 자동화 플랫폼 — 문서 목차

IoT 센서와 AI(LLM + Vision)를 활용한 개인 맞춤형 버섯 재배 자동화 플랫폼의 요구사항/설계 문서입니다.

팀원 배포용 문서이며, 새로 합류하는 팀원은 아래 "읽는 순서"를 따라가면 전체 구조를 빠르게 파악할 수 있습니다.

---

# 📁 폴더 구조

| 폴더 | 내용 |
|------|------|
| [00_Project](./00_Project) | 프로젝트 개요, 기능/기술 요구사항, 아키텍처, 로드맵 |
| [01_Domain](./01_Domain) | 서비스별 도메인 설계 (책임, 기능, 이벤트, 서비스 간 통신) |
| [02_API](./02_API) | 서비스별 REST API 명세 (Request/Response/Error Code) |
| [03_Database](./03_Database) | 서비스별 DB 설계 (ERD/DDL/Index) 및 저장소별(Redis/InfluxDB/Elasticsearch/MinIO) 문서 |
| [04_sequence](./04_sequence) | 주요 시나리오별 서비스 간 시퀀스 다이어그램 |

---

# 📖 읽는 순서 (신규 팀원 추천)

1. [00_Project/overview.md](./00_Project/overview.md) — 프로젝트가 무엇인지
2. [00_Project/architecture.md](./00_Project/architecture.md) — 서비스 구성과 전체 그림
3. [00_Project/tech-stack.md](./00_Project/tech-stack.md) — 사용 기술
4. 담당 서비스의 `01_Domain` 문서 — 내가 만들 서비스의 책임과 경계
5. 담당 서비스의 `02_API` 문서 — 다른 서비스와 합의할 계약(Request/Response)
6. 담당 서비스의 `03_Database` 문서 — 테이블/저장소 설계
7. 관련 `04_sequence` 문서 — 실제 요청이 여러 서비스를 어떻게 거치는지

---

# 🧩 서비스별 문서 매트릭스

API Gateway를 포함해 9개 서비스로 구성됩니다. (기존 9개 서비스에서 Auth+User는 통합, Collector+RuleEngine+Sensor/Storage는 한때 통합했다가 다시 분리)

| 서비스 | 역할 요약 | Domain | API | Database | 관련 Sequence |
|--------|-----------|--------|-----|----------|----------------|
| API Gateway | 라우팅, 인증 토큰 검증 | - | - | - | 전체 시퀀스 최초 진입점 |
| Auth | 인증/인가(이메일+구글 소셜 로그인), JWT, 이메일 인증, 회원 프로필/탈퇴/재배 통계 (기존 Auth+User 통합) | [auth.md](./01_Domain/auth.md) | [auth-api.md](./02_API/auth-api.md) | [auth-db.md](./03_Database/auth-db.md) | [signup](./04_sequence/signup.md), [login](./04_sequence/login.md), [withdraw](./04_sequence/withdraw.md) |
| Cultivation | 재배 생성/관리(센서 등록은 Sensor Service에 OpenFeign 위임)/수확/사진 업로드, 공공데이터 기반 환경 추천(`mushroom_reference` 조회), 수확 임베딩 여부 관리(`harvest.is_embedded`, 인사이트 기능용), 재배 소유권 확인 API 제공(내부용, Sensor Service가 호출) | [cultivation.md](./01_Domain/cultivation.md) | [cultivation-api.md](./02_API/cultivation-api.md) | [cultivation-db.md](./03_Database/cultivation-db.md) | [create-cultivation](./04_sequence/create-cultivation.md), [harvest](./04_sequence/harvest.md), [growth-analysis](./04_sequence/growth-analysis.md), [sensor-error](./04_sequence/sensor-error.md), [insight](./04_sequence/insight.md) |
| AI | 생육 분석(Vision), 챗봇(대화 이력 저장/조회), 주간 리포트, 일일 피드백, 인사이트(타인의 유사 재배 사례 기반 피드백), 버섯 가이드(효능/주의사항) | [ai.md](./01_Domain/ai.md) | [ai-api.md](./02_API/ai-api.md) | [ai-db.md](./03_Database/ai-db.md), Redis(캐시), MinIO(읽기 전용) | [create-cultivation](./04_sequence/create-cultivation.md), [growth-analysis](./04_sequence/growth-analysis.md), [harvest](./04_sequence/harvest.md), [ai-chat](./04_sequence/ai-chat.md), [ai-report](./04_sequence/ai-report.md), [daily-feedback](./04_sequence/daily-feedback.md), [insight](./04_sequence/insight.md) |
| Embedding | 재배 참조 데이터 임베딩·벡터 검색 (AI 챗봇 유사 사례 검색용), 인사이트 사례 배치 임베딩·검색 (`cultivation_insight` 인덱스) | [embedding.md](./01_Domain/embedding.md) | [embedding-api.md](./02_API/embedding-api.md) | [elasticSearch.md](./03_Database/elasticSearch.md) | [ai-chat](./04_sequence/ai-chat.md), [insight](./04_sequence/insight.md) |
| Rule Engine | MQTT 수신(Collector), 검증, 규칙 평가/자동 제어, 센서 오류 감지 | [rule-Engine.md](./01_Domain/rule-Engine.md) | API 없음 (MQTT/RabbitMQ 기반, [rule-engine-api.md](./02_API/rule-engine-api.md) 참고) | Redis(목표 환경 범위 캐시만, 영구 저장소 없음) | [sensor-data](./04_sequence/sensor-data.md), [environment-control](./04_sequence/environment-control.md), [sensor-error](./04_sequence/sensor-error.md) |
| Sensor | 측정값 저장(Redis/InfluxDB)·조회·통계·주간 리포트 집계(Weekly Scheduler), 센서 장치 등록/조회/삭제(`sensor`, Cultivation DB에서 이관), 목표 환경 범위 저장/조회/평균 계산(`environment_setting`, Cultivation DB에서 이관 — Sensor Service 최초의 PostgreSQL) | [sensor.md](./01_Domain/sensor.md) | [sensor-api.md](./02_API/sensor-api.md) | [sensor-db.md](./03_Database/sensor-db.md), [influxdb.md](./03_Database/influxdb.md), [redis.md](./03_Database/redis.md) | [sensor-data](./04_sequence/sensor-data.md), [environment-control](./04_sequence/environment-control.md), [create-cultivation](./04_sequence/create-cultivation.md), [sensor-error](./04_sequence/sensor-error.md), [daily-feedback](./04_sequence/daily-feedback.md), [insight](./04_sequence/insight.md) |
| Notification | WebSocket/Telegram/Discord 알림, 알림 이력 조회/읽음 처리 | [notification.md](./01_Domain/notification.md) | [notification-api.md](./02_API/notification-api.md) (알림 발송 자체는 RabbitMQ 기반) | [notification-db.md](./03_Database/notification-db.md) | [environment-control](./04_sequence/environment-control.md), [harvest](./04_sequence/harvest.md), [sensor-error](./04_sequence/sensor-error.md) |
| DatasourceGenerator | 센서 데이터 생성/발행 (MQTT Publish 전용, REST API 없음, 기존 Datasource Service 리네임) | [datasource-generator.md](./01_Domain/datasource-generator.md) | API 없음 ([datasource-generator-api.md](./02_API/datasource-generator-api.md) 참고) | DB 없음, 메모리 캐시만 사용 ([datasource-generator-db.md](./03_Database/datasource-generator-db.md) 참고) | [sensor-data](./04_sequence/sensor-data.md) |

DB 전체 그림은 [database-overview.md](./03_Database/database-overview.md)에서 한 번에 볼 수 있습니다.

---

# 🕓 최근 주요 결정 사항 (변경 이력 요약)

문서가 여러 차례 업데이트되며 초기 설계와 달라진 부분입니다. 오래된 자료나 기억에 의존하지 말고 아래 최신 결정을 기준으로 작업해주세요.

### 1. 서비스 통합/분리를 여러 차례 재검토했다
강사 피드백("MSA를 흉내내지 않아도 된다, 합칠 수 있는 건 합쳐라")에 따라 서비스 구조를 재검토했고,
그 과정에서 통합했다가 다시 분리한 부분도 있습니다.

- **Auth + User → Auth Service (통합 유지)** — 인증과 회원 프로필은 항상 같이 조회/변경되는 경우가 많아 하나의 서비스, 하나의 `users` 테이블로 통합했습니다.
- **Collector + Rule Engine + Sensor(Storage) → 한때 Rule Engine Service로 통합했다가 다시 Rule Engine Service / Sensor Service로 분리** — MQTT 수신·규칙평가·자동제어와 측정값 저장·조회는 책임 크기와 변경 주기가 달라 분리를 유지하기로 했습니다. 대신 Collector는 Rule Engine Service에 남기고(수신·검증), 저장·조회 API는 Sensor Service로 분리했습니다. 두 서비스는 RabbitMQ(EnvironmentMeasuredEvent)로만 연결되며 서로 직접 호출하지 않습니다. **(이후 결정 사항 24번에서 `environment_setting`이 Sensor Service로 이관되며, Redis 캐시 미스 시에 한해 Rule Engine Service가 Sensor Service를 OpenFeign으로 예외적으로 호출하는 fallback이 생겼습니다. "서로 직접 호출하지 않는다"는 이제 평상시 기준으로만 유효합니다.)**
- **Datasource Service → DatasourceGenerator (리네임 유지)** — 역할은 동일하지만(장치/센서 메타데이터 관리, 시뮬레이션 데이터 발행), 실제 물리 센서가 아닌 시뮬레이션 데이터 생성기라는 점을 이름에 명시했습니다.

최종 구성: API Gateway, Auth, Cultivation, AI, Embedding, Rule Engine, Sensor, Notification, DatasourceGenerator (총 9개)

### 2. DB는 엔진별로 통합하고 스키마/네임스페이스로만 분리
서비스마다 별도 DB 인스턴스를 새로 만들지 않고, PostgreSQL(관계형) / InfluxDB(시계열) / Redis(캐시) / Elasticsearch(벡터) / MinIO(이미지) 5개 저장소를 서비스별로 나눠 쓰는 구조입니다.

### 3. AI 생육 분석은 "환경 계산"이 아니라 "사진 Vision 분석" 방식이다
처음엔 논문 기준값 대비 환경 유지율로 생육 상태를 계산하는 방식(방법 3)을 검토했지만,
최종적으로는 **사용자가 직접 촬영한 사진을 AI Vision 모델로 분석**하는 방식(방법 2)으로 확정했습니다.

- 카메라 센서가 자동 촬영하는 방식이 **아닙니다.** 사용자가 앱/웹에서 사진을 찍어 업로드합니다.
- 사진 업로드는 **Cultivation Service**가 담당합니다. (DatasourceGenerator 아님)
- Vision 모델은 AI Service 내부에 포함되며, 사전에 학습된 성장 단계별 이미지와 비교해 균사 성장률/갓 크기/색상/병충해를 판정합니다.
- 판정된 수치는 결정론적 결과이며, **LLM은 그 결과를 해석하는 설명·개선 방안만 생성**합니다. 수치 자체를 LLM이 새로 추정하지 않습니다.

### 4. 사진 저장은 MinIO, 메타데이터는 Cultivation DB의 `photo` 테이블
이미지 원본은 MinIO(`mushroom-photos` 버킷)에, URL과 업로드 시각 같은 메타데이터는 Cultivation Service의 PostgreSQL에 저장합니다.

### 5. 회원 탈퇴는 Soft Delete + 이벤트 기반 후속 처리
Auth Service가 Soft Delete와 Refresh Token 삭제를 같은 트랜잭션에서 내부 처리한 뒤 `UserDeletedEvent`를 발행하면, Cultivation Service가 이를 구독해 재배 데이터를 비활성화합니다. (기존에는 Auth Service도 이 이벤트를 구독했으나, Refresh Token 삭제가 내부 로직이 되며 더 이상 구독하지 않습니다.)

### 6. environment_setting은 단일 목표값이 아닌 범위(min~max)로 저장
Rule Engine Service의 자동 제어가 값이 범위를 벗어날 때만 장치를 동작시키도록 하여, 범위 안에서 불필요하게 켜고 끄는 것(채터링)을 막기 위함입니다.

- 환경 설정 저장 API(`PATCH /cultivations/{id}/environment`)는 그대로 단일 목표값을 주고받습니다. (재배 생성 시 보여주는 추천값은 이후 결정 사항 9번에서 범위 형태로 바뀌었습니다. **API 자체의 소유 서비스는 결정 사항 24번에서 Cultivation Service → Sensor Service로 바뀌었습니다.**)
- Cultivation Service가 저장 시점에 단일 목표값에 허용 오차를 적용해 범위로 변환합니다. (예: 온도 22℃ ± 1.5℃ → temp_min 20.5 / temp_max 23.5) **(이 변환 로직의 주체도 결정 사항 24번에서 Sensor Service로 이관되었습니다.)**
- 컬럼명은 `temp_min`/`temp_max`, `humidity_min`/`humidity_max`, `co2_min`/`co2_max`, `light_min`/`light_max`였습니다. **(결정 사항 22번에서 항목별 행 구조로 재설계되며 `(type, min, max, unit)`으로 바뀌었습니다. 아래 22번 참고. 테이블 자체는 결정 사항 24번에서 Sensor Service DB로 이관되었습니다.)**
- Rule Engine Service는 현재 센서값이 이 범위를 벗어날 때만 장치를 제어합니다. (자세한 내용은 [cultivation-db.md](./03_Database/cultivation-db.md), [rule-Engine.md](./01_Domain/rule-Engine.md) 참고)

### 7. Rule Engine Service가 목표 환경 범위를 Redis에 캐싱한다
처음에는 규칙 평가 때마다(센서 데이터 수신마다) Cultivation Service를 OpenFeign으로 매번 동기 호출했지만,
호출 빈도가 너무 높아 Cultivation Service에 부하가 몰리고 장애 시 자동 제어까지 막히는 문제가 있어
Redis 캐시를 도입했습니다.

- 캐시 키: `cultivation:{cultivationId}:range`, TTL 24시간
- Cultivation Service가 environment_setting을 생성/수정할 때 `EnvironmentRangeUpdatedEvent`를 발행하고, Rule Engine Service가 이를 구독해 캐시를 미리 채워둡니다(write-through). 규칙 평가 시에는 이 캐시를 먼저 조회합니다.
- 캐시가 없을 때(TTL 만료, 서비스 재시작 직후 등)만 Cultivation Service를 OpenFeign으로 호출하는 fallback으로 동작합니다.
- 이 Redis는 Sensor Service의 "최신 센서값" 캐시와는 별개이며, Rule Engine Service는 여전히 PostgreSQL/InfluxDB 같은 영구 저장소는 갖지 않습니다. (자세한 내용은 [rule-Engine.md](./01_Domain/rule-Engine.md), [redis.md](./03_Database/redis.md) 참고)

> ℹ️ **변경 이력**: 결정 사항 24번에서 `environment_setting`이 Cultivation Service에서 Sensor Service로 이관되며, 위 write-through 이벤트 발행 주체와 fallback 호출 대상이 모두 **Sensor Service**로 바뀌었습니다.

### 8. 센서 데이터 저장 시 Redis는 실시간(매초), InfluxDB는 10초로 스로틀링
센서가 1초 주기로 값을 보내는 걸 그대로 InfluxDB에 다 기록하면 재배 1건 기준 한 달에 수백 MB,
장기 누적 시 용량이 부담스러워집니다. 반면 재배실 환경(온도/습도/CO₂/조도)은 물리적으로 초 단위로
급변하지 않으므로, 저장 주기와 제어 반응 주기를 분리했습니다.

- **Rule Engine Service의 규칙 평가/자동 제어는 매초 원본 데이터로 그대로 수행**합니다. 반응 속도에는 영향이 없습니다.
- **Sensor Service의 Redis(현재값 캐시)는 EnvironmentMeasuredEvent를 받을 때마다(매초) 항상 덮어씁니다.** 값 1건만 유지하는 구조라 매초 갱신해도 용량에 영향이 없고, 대시보드 체감 실시간성도 그대로 유지됩니다.
- **Sensor Service의 InfluxDB(이력 저장)만 재배별로 10초 이상 지났을 때만 기록**합니다. 저장 용량을 약 1/10로 줄일 수 있습니다.
- 마지막 InfluxDB 기록 시각은 Sensor Service가 재배별로 애플리케이션 메모리에서 관리합니다. (자세한 내용은 [sensor.md](./01_Domain/sensor.md), [influxdb.md](./03_Database/influxdb.md) 참고)

### 9. 환경 추천은 AI Service가 아닌 Cultivation Service가 담당한다 (참조 테이블 조회)
공공데이터로 확보한 버섯 재배 참조 데이터가 5종류뿐이라, AI Service가 Embedding/Vector
Search/LLM을 거쳐도 항상 같은 값이 나오는 문제가 있었습니다. 이 조회에는 AI가 필요 없다고
판단해 담당을 옮겼습니다.

- Cultivation Service DB에 `mushroom_reference`라는 전역 참조 테이블을 신설했습니다. (`mushroom_type`별 최적 환경 범위 + 설명, 공공데이터 기반 시드 데이터, cultivation과 FK 관계 없음)
- 재배 생성 시 Cultivation Service가 이 테이블을 **내부에서 직접 조회**하며, AI Service·Embedding Service·Elasticsearch·LLM을 전혀 호출하지 않습니다.
- **`mushroom_reference`(최적 범위)와 `environment_setting`(위험 한계값)은 서로 다른 개념입니다.** 전자는 "이 조건이면 잘 자란다"는 참고용 추천 데이터이고, 후자는 사용자가 직접 정하는 자동 제어 기준값입니다. Rule Engine Service는 지금까지와 동일하게 `environment_setting`(및 그 Redis 캐시)만 사용하며, `mushroom_reference`를 직접 참조하지 않습니다.
- AI Service는 더 이상 "환경 추천" 책임을 갖지 않습니다. (생육 분석/챗봇/리포트는 그대로 유지) Embedding Service/Elasticsearch는 AI 챗봇의 선택적 유사 사례 검색 용도로만 남아있습니다. (자세한 내용은 [cultivation-db.md](./03_Database/cultivation-db.md), [cultivation.md](./01_Domain/cultivation.md), [ai.md](./01_Domain/ai.md) 참고)

### 10. 센서 "장치" CRUD는 DatasourceGenerator가 아닌 Cultivation Service가 담당한다
센서 장치의 등록/조회/삭제를 원래 DatasourceGenerator가 담당했지만, 센서가 항상 특정 재배에
종속되는 정보이고 DatasourceGenerator는 데이터 생성/발행 역할에 집중하는 것이 책임 경계가
명확하다고 판단해 Cultivation Service로 옮겼습니다. (측정값 저장/조회는 여전히 Sensor Service,
값 검증/자동제어는 여전히 Rule Engine Service입니다 — 이번에 옮긴 것은 오직 센서 "장치" 메타데이터입니다.)

- `sensor` 테이블이 DatasourceGenerator DB → Cultivation DB로 이전되었습니다. `cultivation_id`는 이제 같은 DB 내 실제 FK입니다.
- API 경로도 `POST/GET/DELETE /api/v1/sensors` → `POST/GET/DELETE /api/v1/cultivations/{id}/sensors`로 이동했습니다.
- DatasourceGenerator는 센서를 직접 소유하지 않는 대신, Cultivation Service가 발행하는 `SensorRegisteredEvent`/`SensorDeletedEvent`를 구독해 "어떤 센서에 대해 데이터를 생성/발행할지"만 판단하는 읽기 전용 캐시(`sensor_cache`)를 둡니다. (이벤트 기반 동기화 — OpenFeign 동기 호출 방식은 채택하지 않았습니다. 단, 이 캐시를 어디에 저장할지는 이후 결정 사항 12번에서 DB에서 메모리로 바뀌었습니다.)
- 센서 상태(ONLINE/OFFLINE/ERROR) 갱신용 `SensorErrorEvent`(Rule Engine Service 발행)의 구독 주체도 DatasourceGenerator에서 Cultivation Service로 함께 이전했습니다.
- 이 변경으로 아래 "열려있는 이슈"에 있던 `/api/v1/sensors` 경로 충돌이 자연스럽게 해소되었습니다. (자세한 내용은 [cultivation-db.md](./03_Database/cultivation-db.md), [cultivation.md](./01_Domain/cultivation.md), [datasource-generator-db.md](./03_Database/datasource-generator-db.md) 참고)

> ℹ️ **변경 이력**: 결정 사항 24번에서 `sensor` 테이블과 그 CRUD API가 Cultivation Service에서 **Sensor Service**로 다시 이관되었습니다. 이번 결정의 "DatasourceGenerator가 아닌 Cultivation Service" 부분은 더 이상 최신 상태가 아니며, 자세한 내용은 아래 24번을 참고해주세요.

### 11. 센서 등록 정보를 `device_eui` 기반으로 전면 재설계하고, "데이터 소스" 개념을 제거했다

실제 하드웨어 식별자인 `device_eui`를 기준으로 센서를 관리하는 것이 자연스럽다고 판단해, 결정 사항 10번에서 옮긴 `sensor` 테이블의 필드 구조 자체를 다시 정리했습니다.

- `sensor` 테이블의 PK가 자동 생성 `id`(BIGSERIAL)에서 사용자가 등록 시 직접 입력하는 `device_eui`(INT)로 바뀌었습니다. (이후 결정 사항 18번에서 `VARCHAR(32)`로 다시 수정되었습니다.)
- 센서 등록 시 입력 필드가 `device_eui`, `place`, `location`, `device_model`, `sensor_type`으로 정리되었습니다. 기존의 `sensor_uuid`, `name`, `installed_at`, `datasource_id` 컬럼은 제거되었습니다.
- 위치 정보(`place`/`location`)가 센서 레코드에 직접 저장되면서, 별도로 위치를 그룹핑하던 **"데이터 소스"(datasource) 엔티티 자체가 필요 없어져 완전히 제거**되었습니다. DatasourceGenerator DB의 `datasource` 테이블과 그 REST API(`/api/v1/datasources`)가 모두 삭제되었고, 그 결과 DatasourceGenerator는 REST API를 전혀 제공하지 않는 서비스가 되었습니다(MQTT Publish + RabbitMQ Subscribe만 수행).
- 센서의 유일한 실질 식별자가 `device_eui`가 되면서, 등록 API뿐 아니라 **측정 파이프라인 전체**(MQTT Payload, RabbitMQ의 `EnvironmentMeasuredEvent`/`SensorErrorEvent`, InfluxDB Tag)에서 기존에 쓰던 범용 필드명 `sensorId`를 `deviceEui`로 통일했습니다. (컬럼명은 DB에서는 `device_eui`(snake_case), JSON payload에서는 `deviceEui`(camelCase)를 사용하는 기존 컨벤션을 그대로 따릅니다.)
- API 경로 파라미터도 `{sensorId}` → `{deviceEui}`로 바뀌었습니다. (예: `DELETE /api/v1/cultivations/{id}/sensors/{deviceEui}`)
- (자세한 내용은 [cultivation-db.md](./03_Database/cultivation-db.md), [cultivation.md](./01_Domain/cultivation.md), [cultivation-api.md](./02_API/cultivation-api.md), [datasource-generator.md](./01_Domain/datasource-generator.md), [datasource-generator-api.md](./02_API/datasource-generator-api.md), [datasource-generator-db.md](./03_Database/datasource-generator-db.md) 참고)

> ℹ️ **변경 이력**: `device_eui` 기반 필드 구조 자체는 그대로 유효하지만, `sensor` 테이블의 소유 서비스는 결정 사항 24번에서 Cultivation DB → Sensor DB로 다시 바뀌었습니다.

### 12. DatasourceGenerator의 `sensor_cache`를 PostgreSQL 대신 메모리(In-Memory)로 전환했다

결정 사항 10~11번을 거치며 DatasourceGenerator DB에는 `sensor_cache` 테이블 하나만 남았는데,
device_eui/cultivationId/sensorType 세 컬럼만 갖고 조인·트랜잭션이 전혀 없는 단순 조회용
캐시였습니다. 이런 용도로 별도 PostgreSQL 인스턴스(스키마)를 유지하는 것이 과하다고 판단해
DB 자체를 없앴습니다.

- `sensor_cache`는 이제 PostgreSQL 테이블이 아니라 서비스 메모리(In-Memory, 예: ConcurrentHashMap)에서만 관리됩니다. 이로써 DatasourceGenerator는 어떤 영구 저장소도 갖지 않는 서비스가 되었습니다(REST API도 결정 사항 11번에서 이미 없어졌으므로, MQTT Publish + RabbitMQ Subscribe만 남습니다).
- 평상시 갱신 방식(`SensorRegisteredEvent`/`SensorDeletedEvent` 구독)은 그대로지만, 메모리 캐시는 서비스 재시작 시 비게 되는 문제가 새로 생겼습니다. 이를 해결하기 위해 **서비스 시작 시점에 한해** Cultivation Service의 신규 엔드포인트 `GET /api/v1/sensors`(전체 재배의 센서 목록, 내부용)를 OpenFeign으로 호출해 캐시를 일괄 재구성합니다.
- 이벤트 유실(RabbitMQ 장애 등)에 대한 주기적 재동기화(reconciliation) 배치는 아직 없으며, 현재는 서비스 재시작 시의 전체 재조회만으로 복구합니다. (추후 개발 예정)
- (자세한 내용은 [datasource-generator-db.md](./03_Database/datasource-generator-db.md), [datasource-generator.md](./01_Domain/datasource-generator.md), [datasource-generator-api.md](./02_API/datasource-generator-api.md), [cultivation-api.md](./02_API/cultivation-api.md)의 "센서 전체 목록 조회 (내부용)" 참고)

### 13. 재배 생성 시 센서 장치를 함께 등록할 수 있게 됐고, AI Service에 "버섯 가이드" 기능이 추가됐다

실제 사용자 흐름을 구체화하면서(재배 생성 → 등록할 디바이스 선택 → 버섯 효능/주의사항
확인 → 환경 설정 → 위험값 초과 시 자동 제어) 두 가지를 반영했습니다.

- `POST /cultivations` 요청에 `devices`(선택)를 추가해, 재배 생성과 초기 센서 등록을 한 트랜잭션으로 처리할 수 있게 했습니다. 이후 개별적으로 센서를 추가/삭제하는 기존 `POST/DELETE /cultivations/{id}/sensors` API도 그대로 유지됩니다. `devices`에 이미 등록된 device_eui가 섞여 있으면 재배 생성 자체가 롤백됩니다.
- AI Service에 "버섯 가이드"(효능/재배 시 주의사항을 LLM으로 생성) 기능이 추가되었습니다. `POST /ai/mushroom-guide`를 Client가 재배 생성 직후 AI Service에 직접 호출합니다(Cultivation Service를 거치지 않음).
- 환경 추천(mushroom_reference 조회)을 AI에서 뺐던 이유(5종 고정이라 항상 같은 값)와 동일한 문제가 있지만, 이번엔 "수치"가 아닌 "설명 문서"를 만드는 것이라 LLM 활용이 여전히 유효하다고 판단했습니다. 대신 반복 호출을 피하기 위해 `cultivationId`가 아닌 `mushroomType` 기준으로 캐싱합니다(`ai:mushroom:{mushroomType}:guide`, TTL 7일).
- `mushroom_reference.description`(재배 생성 응답에 포함되는 짧은 한 줄 문구)과 버섯 가이드(효능/주의사항, 별도 API로 조회하는 긴 설명)는 서로 다른 콘텐츠입니다.
- (자세한 내용은 [cultivation-api.md](./02_API/cultivation-api.md), [cultivation.md](./01_Domain/cultivation.md), [ai.md](./01_Domain/ai.md), [ai-api.md](./02_API/ai-api.md), [create-cultivation.md](./04_sequence/create-cultivation.md), [redis.md](./03_Database/redis.md) 참고)

> ℹ️ **변경 이력**: `POST /cultivations`에 `devices`를 함께 보내는 요청 형태는 그대로지만, 내부 처리 방식은 결정 사항 24번에서 "하나의 트랜잭션"에서 "Cultivation 생성 후 Sensor Service를 OpenFeign으로 배치 호출(실패 시 보상 삭제)"로 바뀌었습니다.

### 14. Rule Engine Service의 자동 제어 정지 기준을 "범위 경계"가 아닌 "중앙값"으로 명확히 했다

기존에도 목표 환경을 범위(min~max)로 저장해 채터링을 줄이고 있었지만, 제어를 멈추는 시점이
"범위 안으로 복귀하는 즉시"였기 때문에 경계 부근에서 값이 미세하게 오르내리면 여전히 반복
On/Off가 발생할 수 있었습니다. 이를 보완했습니다.

- 제어를 **시작**하는 기준은 그대로 범위 경계(min/max)입니다.
- 제어를 **멈추는** 기준은 범위 경계가 아니라 범위의 중앙값(mid = (min+max)/2)입니다. 즉, 장치가 한 번 켜지면 값이 범위 안으로 돌아온 뒤에도 중앙값에 도달할 때까지 계속 동작합니다.
- 이 중앙값은 사용자가 환경 설정 저장 시 입력했던 단일 목표값과 정확히 같습니다(허용 오차가 대칭이므로).
- 이 제어를 구현하려면 Rule Engine Service가 재배별/장치별 현재 ON/OFF 상태를 추적해야 하는데, 구체적인 저장 방식(Redis 키 설계 등)은 아직 확정하지 않았고 추후 개발 예정으로 남겨뒀습니다.
- (자세한 내용은 [rule-Engine.md](./01_Domain/rule-Engine.md), [environment-control.md](./04_sequence/environment-control.md) 참고)

### 15. `mushroom_reference`에 이름/특성/효능/재배 가이드 등 텍스트 데이터를 병합했다

공공데이터로 확보한 버섯별 상세 정보(이름 한글/영문/학명, 특성, 효능, 재배 가이드, 추가 정보,
임베딩)를 별도 `mushroom`이라는 테이블(PK `mushroom_id`)로 논의했지만, 버섯 종류당 한 행만
존재하는 정적 참조 데이터라는 점에서 기존 `mushroom_reference`와 본질적으로 같은 데이터이므로
병합하기로 했습니다.

- `mushroom_reference`에 `mushroom_name_ko`/`mushroom_name_en`/`mushroom_scientific_name`/`characteristics`/`health_benefits`/`cultivation_guide`/`additional_info` 컬럼이 추가되었습니다. PK는 기존과 동일하게 `mushroom_type`을 유지하고, 별도 대리키 `mushroom_id`는 도입하지 않았습니다(`mushroom_type`이 이미 재배 생성 API, AI 버섯 가이드 API 등 시스템 전반의 식별자이기 때문입니다).
- 원안의 `embedding VECTOR(1024)` 컬럼은 Cultivation DB(PostgreSQL)에 두지 않았습니다. 이미 [elasticSearch.md](./03_Database/elasticSearch.md)에 "PostgreSQL과 데이터를 중복 저장하지 않는다"는 원칙이 있었고, 임베딩 계산/보관은 Embedding Service의 책임이기 때문입니다. 대신 Cultivation Service가 `MushroomReferenceUpdatedEvent`를 발행하면 Embedding Service가 구독해 characteristics/health_benefits/cultivation_guide/additional_info를 임베딩하고 Elasticsearch의 `mushroom_environment` 인덱스를 갱신합니다.
- AI Service의 "버섯 가이드"(결정 사항 13번)는 이제 이 텍스트를 무(無)에서 생성하지 않고, Cultivation Service를 OpenFeign으로 호출해(`GET /api/v1/mushroom-references/{mushroomType}`, 내부용) 가져온 원문을 LLM 프롬프트의 RAG 컨텍스트로 사용합니다. LLM은 원문을 그대로 반환하지 않고 자연스러운 문장의 `benefits`/`precautions`로 재구성합니다.
- 전체 재생성(임베딩 모델 교체 등 예외 상황)을 위한 `GET /api/v1/mushroom-references`(전체 목록, 내부용) 엔드포인트도 함께 추가했습니다. 평상시 동기화는 이벤트 기반이라 이 호출은 드뭅니다.
- Elasticsearch의 `mushroom_environment` 인덱스 스키마도 이 필드들을 반영하도록 확장했습니다.
- (자세한 내용은 [cultivation-db.md](./03_Database/cultivation-db.md), [cultivation.md](./01_Domain/cultivation.md), [cultivation-api.md](./02_API/cultivation-api.md), [ai.md](./01_Domain/ai.md), [ai-api.md](./02_API/ai-api.md), [embedding.md](./01_Domain/embedding.md), [embedding-api.md](./02_API/embedding-api.md), [elasticSearch.md](./03_Database/elasticSearch.md) 참고)

### 16. 구글 소셜 로그인을 추가하고, 회원 탈퇴에 `status` 컬럼을 도입했다

`users` 테이블 DDL을 다시 검토하던 중, "추후 개발 예정"에 있던 구글 OAuth2 로그인을 실제로
추가하기로 하면서 스키마를 함께 정리했습니다.

- `users`에 `provider`(LOCAL/GOOGLE) 컬럼이 추가되었습니다. `email` 단독 `UNIQUE` 대신 `(email, provider)` 조합 `UNIQUE`로 바뀌어, 같은 이메일이라도 LOCAL 계정과 GOOGLE 계정을 별도 행으로 허용합니다(두 계정을 하나로 합치는 계정 연동은 추후 개발 예정). `password`는 GOOGLE 계정에는 없으므로 NULL을 허용하며, `provider가 LOCAL이면 password가 NOT NULL`이라는 CHECK 제약을 추가했습니다.
- `POST /auth/google`이 신설되었습니다. 프론트엔드가 구글 로그인으로 받은 ID Token을 전달하면 Auth Service가 구글 공개키로 검증하고, 최초 로그인이면 자동으로 회원가입까지 처리합니다(별도의 "구글 회원가입" API는 없음).
- `role`(USER/ADMIN)은 그대로 유지했습니다. mushroom_reference 갱신, Embedding 관리 API 같은 관리자 전용 기능이 이미 이 값에 의존하고 있어서, provider/status 추가와는 별개로 남겨뒀습니다.
- 회원 탈퇴 여부를 명시적으로 표현하기 위해 `status`(ACTIVE/DELETED) 컬럼을 도입했습니다. 처음에는 `status` 도입과 함께 기존 `deleted_at` 컬럼을 제거하려 했으나, 아래 결정 사항 17번에서 다시 검토되어 `deleted_at`을 복원했습니다.
- `nickname`도 실제 `UNIQUE` 제약을 추가했습니다. 기존에는 인덱스만 있고 제약이 없었는데, 에러 코드 A009(중복 닉네임)가 이미 문서화되어 있던 것과 맞춰 정합성을 맞췄습니다.
- (참고: 같은 시점에 검토한 `mushroom`/`environment_setting`/`cultivation`/`sensor`/`harvest` DDL 초안은 오래된 버전이었고, `mushroom_reference` 병합(결정 사항 15번)과 device_eui 기반 sensor 설계(결정 사항 10~11번) 등 기존 결정을 그대로 유지하기로 확인했습니다. 새 DDL 자체가 문서를 대체하지는 않았습니다.)
- (자세한 내용은 [auth-db.md](./03_Database/auth-db.md), [auth.md](./01_Domain/auth.md), [auth-api.md](./02_API/auth-api.md), [login.md](./04_sequence/login.md), [withdraw.md](./04_sequence/withdraw.md) 참고)

### 17. `deleted_at`을 복원하고, 휴면 계정(`DORMANT`) 상태를 추가했다

결정 사항 16번에서 `status` 도입과 함께 `deleted_at`을 제거했는데, "탈퇴 취소(계정 복구) 기능을
만들려면 정확한 탈퇴 시점이 필요한데 `status`만으로는 알 수 없지 않냐"는 지적이 있었습니다.
`updated_at`으로 대체하는 방안도 검토했지만, `updated_at`은 탈퇴 이외의 이유로도 바뀌는 값이라
신뢰할 수 없어 기각했습니다.

- `users`에 `deleted_at`(nullable TIMESTAMP)을 복원했습니다. 탈퇴 여부 조회/필터링은 `status = 'DELETED'`로, 정확한 탈퇴 시각은 `deleted_at`으로 확인합니다. `CHECK ((status = 'DELETED') = (deleted_at IS NOT NULL))` 제약으로 두 컬럼이 항상 일치하도록 강제합니다. "탈퇴 취소" 기능 자체는 이번에 구현하지 않고 추후 개발 예정으로 남겨뒀습니다(스키마만 우선 반영).
- 같은 논의 중에 `status`가 ACTIVE/DELETED 두 값만 가진다면 `deleted_at` 하나만으로도 표현이 가능해 두 컬럼이 중복 아니냐는 질문이 나왔는데, 이메일 인증은 회원가입 시 이미 필수라 "인증 대기" 상태는 없지만 장기 미로그인 계정을 위한 휴면(`DORMANT`) 상태를 추가하기로 하면서 `status`가 3개 값을 가지게 되어 `status`(상태 구분)와 `deleted_at`(탈퇴 시각)을 함께 두는 것으로 정리되었습니다.
- 휴면 전환은 별도 배치 없이 **로그인 시도 시점**에 판단합니다. `users`에 `last_login_at` 컬럼을 추가했고, 로그인 시 비밀번호 검증에 성공한 뒤 `last_login_at` 기준으로 휴면 기준일(문서상 예시 90일)을 넘었으면 `status`를 `DORMANT`로 전환하고 이메일 인증번호를 발송합니다. JWT는 이 시점에 발급하지 않습니다.
- 재활성화는 새 엔드포인트 `POST /auth/login/reactivate`로 처리합니다. 인증번호를 검증하면 `status`를 `ACTIVE`로 되돌리고, 최초 로그인 시도에서 이미 끝난 비밀번호 검증을 다시 요구하지 않고 그대로 JWT를 발급합니다.
- `POST /auth/login`은 `status = 'DELETED'`인 계정의 로그인도 거부하도록 명확히 했습니다(기존에는 이 분기가 문서에 없었습니다).
- (자세한 내용은 [auth-db.md](./03_Database/auth-db.md), [auth.md](./01_Domain/auth.md), [auth-api.md](./02_API/auth-api.md), [login.md](./04_sequence/login.md), [withdraw.md](./04_sequence/withdraw.md) 참고)

### 18. `device_eui` 타입을 `INT`에서 `VARCHAR(32)`로 바꿨다

결정 사항 11번에서 `device_eui`를 `INT`로 정했었는데, 실제 실습실 장비(Milesight AM107)가
MQTT로 보내는 페이로드를 확인해보니 `device_eui`가 `"24e124128c067999"`처럼 LoRaWAN 표준
64비트 DevEUI를 16자리 hex 문자열로 표현한 값이었습니다. 앞자리 `0`이 의미를 가질 수 있고
산술 연산이 필요 없는 순수 식별자라, 정수 타입이 아니라 문자열이 맞다고 판단했습니다.

- `cultivation-db.md`의 `sensor.device_eui` DDL을 `INT NOT NULL PRIMARY KEY`에서
  `VARCHAR(32) NOT NULL PRIMARY KEY`로 바꿨습니다.
- `sensor_cache`(DatasourceGenerator, 메모리)의 키 타입도 `Map<Integer, ...>`에서
  `Map<String, ...>`로 바꿨습니다.
- 이 변경은 문서 전반의 표기(DDL, API 요청/응답 예시, RabbitMQ 이벤트 예시, InfluxDB Tag 예시)에만
  우선 반영했습니다. 실제 엔티티 코드(`Sensor.java`의 `id` 필드)는 아직 `Long`이며, 이번에는
  건드리지 않기로 했습니다(문서 우선 반영, 코드 반영은 추후).
- (자세한 내용은 [cultivation-db.md](./03_Database/cultivation-db.md), [cultivation-api.md](./02_API/cultivation-api.md), [cultivation.md](./01_Domain/cultivation.md), [datasource-generator-db.md](./03_Database/datasource-generator-db.md) 참고)

### 19. Notification Service와 AI Service에 각각 PostgreSQL DB를 신설했다 (`notification`, `chat_message`)

전체 DB/테이블 구성을 다시 점검하면서, "테이블 개수가 너무 적은 것 아니냐"는 의견이 있었습니다.
검토해보니 실제로 두 곳이 비어 있었습니다. Notification Service는 알림을 발송만 하고 기록을
남기지 않아 사용자가 지난 알림을 다시 볼 방법이 없었고, AI 챗봇도 `ai:{hash}` 응답 캐시만
있을 뿐 대화 이력을 조회할 방법이 없었습니다.

- Notification Service에 `notification` 테이블(`id`, `user_id`, `type`, `message`, `is_read`, `created_at`)을 추가했습니다. `type`은 기존에 구독하던 6개 RabbitMQ 이벤트와 1:1로 대응합니다. 알림 목록 조회(`GET /notifications`)/읽음 처리(`PATCH /notifications/{id}/read`, `PATCH /notifications/read-all`) API가 함께 추가되면서, Notification Service가 처음으로 REST API와 PostgreSQL DB를 갖게 되었습니다.
- AI Service에 `chat_message` 테이블(`id`, `user_id`, `cultivation_id`, `message`, `answer`, `created_at`)을 추가했습니다. `POST /ai/chat`이 매 질의응답을 이 테이블에 저장하며, 새로 추가된 `GET /ai/chat/history`로 재배별 대화 이력을 조회할 수 있습니다. 기존 `ai:{hash}` Redis 캐시(동일 질문 재요청 시 LLM 재호출 방지용)는 역할이 겹치지 않아 그대로 유지됩니다. AI Service가 처음으로 PostgreSQL DB를 갖게 되었습니다.
- 두 서비스 모두 기존 Auth DB/Cultivation DB에 얹지 않고 완전히 독립된 새 DB로 만들었습니다("서비스마다 자기 DB만 소유한다"는 기존 원칙을 그대로 유지).
- 테이블/컬럼 설계 과정에서 `harvest.cultivation_id UNIQUE`(재배당 수확 1회만 허용) 제약이 실제 버섯 재배(여러 번 수확하는 "플러시")와 맞지 않을 수 있다는 점도 별도로 논의되었으나, 이번 변경 범위에는 포함하지 않았습니다(추후 검토 필요).
- (자세한 내용은 [notification-db.md](./03_Database/notification-db.md), [notification-api.md](./02_API/notification-api.md), [notification.md](./01_Domain/notification.md), [ai-db.md](./03_Database/ai-db.md), [ai-api.md](./02_API/ai-api.md), [ai.md](./01_Domain/ai.md), [database-overview.md](./03_Database/database-overview.md) 참고)

### 20. 월간 리포트를 폐기하고, 리포트 생성 방식을 사용자 요청(pull)에서 Scheduler 기반(push)으로 통일했다

버섯 재배 기간이 한 달을 넘지 않는다는 도메인 특성을 다시 짚으면서, "월간" 단위 리포트 자체가
성립하지 않는다는 점을 확인했습니다. 동시에 리포트 생성 방식이 문서마다 서로 다르게(Scheduler가
미리 생성 vs 사용자 요청 시 동기 생성) 적혀 있던 모순도 함께 정리했습니다.

- `sensor.md`의 Monthly Scheduler, `GET /sensors/report/monthly`, `sensor-api.md`의 월간 데이터 조회, `ai.md`/`ai-api.md`/`notification.md`의 `MonthlyReportCompletedEvent`/월간 알림을 모두 제거했습니다.
- 리포트 생성은 이제 Sensor Service의 Weekly Scheduler가 매주 먼저 InfluxDB 집계 데이터를 AI Service에 전달(push)하고, AI Service가 그 자리에서 리포트를 생성해 Redis(`report:{cultivationId}:weekly`)에 저장한 뒤 `WeeklyReportCompletedEvent`를 발행합니다.
- 기존 `POST /ai/report`(사용자 요청 기반 생성)는 제거되었고, 이미 생성된 리포트를 읽기만 하는 `GET /ai/report`로 대체되었습니다.
- (자세한 내용은 [ai-report.md](./04_sequence/ai-report.md), [sensor.md](./01_Domain/sensor.md), [ai.md](./01_Domain/ai.md), [ai-api.md](./02_API/ai-api.md) 참고)

### 21. "일일 피드백" 기능을 추가했다 (사용자가 환경을 수정하면 생육 변화를 매일 비교해 알려줌)

사용자가 재배 환경(예: 온도)을 mushroom_reference 추천값과 다르게 임의로 수정했을 때, 그
수정이 실제로 생육에 도움이 됐는지를 매일 확인해 알려주는 기능입니다. 주간 리포트와 마찬가지로
Scheduler 기반(push)으로 만들기로 했습니다.

- AI Service에 Daily Scheduler를 추가했습니다. 매일 재배별로 생육 분석 이력(`growth_record`)과 환경 변경 이력(Cultivation Service의 `environment_setting`, **결정 사항 24번에서 Sensor Service로 이관**)을 비교해 LLM으로 피드백을 생성하고, `daily_feedback`에 저장한 뒤 `DailyFeedbackCompletedEvent`를 발행합니다.
- `growth_record` 테이블을 신설했습니다. 기존에는 생육 분석(Vision) 결과를 `ai:{cultivationId}:analysis` Redis 캐시(TTL 6시간)에만 저장해 하루만 지나도 이력이 사라졌는데, 일일 피드백이 여러 날짜의 추이를 비교하려면 영구 저장이 필요했기 때문입니다. Redis 캐시는 "빠른 재조회"용으로 그대로 유지됩니다.
- 사용자가 전날 생육 사진을 찍지 않았다면 비교할 `growth_record`가 없으므로, 이 경우 LLM을 호출하지 않고 "전날 사진이 없어 피드백을 남길 수 없습니다"라는 고정 문구로 `daily_feedback` 행을 생성합니다(건너뛰지 않음 — "피드백 없음"과 "비교 데이터 없음"을 구분하기 위함).
- 조회는 `GET /ai/feedback/daily`로 제공하며, 생성 자체는 API로 트리거하지 않습니다.
- (자세한 내용은 [daily-feedback.md](./04_sequence/daily-feedback.md), [ai-db.md](./03_Database/ai-db.md), [ai.md](./01_Domain/ai.md), [ai-api.md](./02_API/ai-api.md) 참고)

### 22. `environment_setting`을 "재배당 1행" 구조에서 "항목별 여러 행이 이력으로 쌓이는" 구조로 재설계했다

결정 사항 21번(일일 피드백)을 구현하려면 "환경값을 언제 얼마나 바꿨는지"에 대한 이력이
필요한데, 기존 `environment_setting`은 `cultivation_id UNIQUE`라 수정할 때마다 이전 값을
덮어써 이력이 전혀 남지 않았습니다. 별도 `environment_setting_history` 테이블을 추가하는
방안도 검토했지만, 테이블 자체를 항목별 행으로 바꾸고 UPDATE 대신 INSERT만 하도록 하면 테이블
하나로 "현재값"과 "이력"을 동시에 표현할 수 있어 이 방식으로 확정했습니다.

- 컬럼 구조가 `(temp_min, temp_max, humidity_min, humidity_max, co2_min, co2_max, light_min, light_max)` 8개에서 `(type, min, max, unit)`으로 바뀌었습니다. `type`은 `TEMPERATURE`/`HUMIDITY`/`CO2`/`LIGHT` 중 하나이며, 재배 하나당 항목별로 여러 행이 쌓입니다.
- `min`/`max`는 `DECIMAL(4,1)`로 통일했습니다(기존에는 CO₂/조도가 `INT`). 하나의 컬럼을 모든 항목이 공유하는 구조라 온도의 소수점 정밀도를 살리는 쪽으로 맞췄고, CO₂/조도는 정수여도 그대로 저장됩니다.
- `cultivation_id`의 `UNIQUE` 제약을 제거했습니다. `cultivation`과의 관계가 1:1에서 1:N으로 바뀌었습니다. "현재값"은 `(cultivation_id, type)` 기준 최신 행(`created_at DESC`)으로 조회합니다.
- `PATCH /cultivations/{id}/environment`가 4개 필드를 모두 요구하지 않고 **부분 수정**(예: 온도만)을 지원하도록 바뀌었습니다. 수정한 항목만 새 행이 INSERT되고, 나머지 항목의 최신 행은 그대로 유지됩니다. `EnvironmentRangeUpdatedEvent`(Rule Engine Service의 Redis 캐시 갱신용)는 기존과 동일하게 4개 항목 전체를 담아 발행합니다(Rule Engine이 항상 4개 항목 전체를 알아야 하므로).
- (자세한 내용은 [cultivation-db.md](./03_Database/cultivation-db.md), [cultivation-api.md](./02_API/cultivation-api.md), [cultivation.md](./01_Domain/cultivation.md), [daily-feedback.md](./04_sequence/daily-feedback.md) 참고)

> ℹ️ **변경 이력**: 여기서 확정한 `(type, min, max, unit)` 행 구조는 그대로 유효하지만, 테이블의 소유 서비스는 결정 사항 24번에서 Cultivation DB → Sensor DB로 이관되었습니다. `PATCH /cultivations/{id}/environment`도 Sensor Service의 엔드포인트로 옮겨졌습니다. 자세한 내용은 아래 24번, [sensor-db.md](./03_Database/sensor-db.md) 참고.

### 23. "인사이트" 기능을 추가했다 (같은 버섯 종류 + 유사한 온도로 재배한 타인의 사례 기반 피드백)

일일 피드백(결정 사항 21번)이 **자기 자신**의 환경 변경 이력과 생육 추이를 비교하는 기능이라면,
인사이트는 같은 버섯 종류이면서 유사한(오차 범위 내) 온도로 재배했던 **다른 사용자들**의
완료된 재배 사례를 바탕으로 현재 재배 상태를 피드백해주는 기능입니다. 두 기능은 서로 다른
기능이며, "일일 피드백" 문서/코드를 확장하는 대신 완전히 새로운 흐름과 인덱스로 분리했습니다.

- **데이터 적재(배치)와 조회(on-demand)를 명확히 분리했습니다.** AI Service의 Insight Batch Scheduler가 매일 00시에 실행되어, 전체 사용자를 합산해 아직 임베딩되지 않은 수확(harvest)이 20건 이상이면 그 배치를 Embedding Service로 보내 임베딩 후 Elasticsearch(`cultivation_insight` 인덱스, 신규)에 저장합니다. 반면 조회는 이 배치와 완전히 독립적으로, 사용자가 `GET /ai/insight`를 요청한 시점에만 검색/요약이 이루어집니다(일일 피드백/AI 리포트처럼 미리 생성해두지 않음).
- **"매일 다시 훑는" 방식이 아니라 "임계치(20건)에 도달한 만큼만 처리하는" 배치 방식입니다.** 처음에는 별도 워터마크(마지막 처리 시각/ID) 관리 방식도 검토했지만, Cultivation Service DB에 `harvest.is_embedded`(boolean) 컬럼을 두어 AI Service가 단순 조회만 하도록 단순화했습니다 — 이 논의 초반에는 "AI Service가 직접 오케스트레이션하는 게 맞는지"도 다시 짚었는데, 사용자의 원래 설계 의도("스케줄링이 돌면서... 즉시 임베딩 서비스로 가서...")대로 AI Service가 오케스트레이터 역할을 하는 것으로 확정했습니다.
- Cultivation Service에 내부용 엔드포인트 3종(`GET /api/v1/harvests/unembedded-count`, `GET /api/v1/harvests/unembedded`, `PATCH /api/v1/harvests/embedded`)과, 진행 중인 재배도 조회 가능한 `GET /api/v1/cultivations/{cultivationId}/environment-average`를 추가했습니다. 환경값은 기간 가중 평균(각 설정값이 적용된 기간의 길이로 가중)으로 계산하며, 별도 컬럼에 저장하지 않고 조회 시점마다 계산합니다. **(결정 사항 24번에서 `environment-average` 조회는 Sensor Service로 이관되었고, AI Service의 배치 적재용으로 여러 건을 한 번에 조회하는 `POST /api/v1/sensors/environment-averages` 배치 엔드포인트도 새로 추가되었습니다. `unembedded-count`/`unembedded`/`embedded`는 그대로 Cultivation Service에 남아 있습니다.)**
- 임베딩되는 "사례"는 버섯 종류 + 기간 가중 평균 환경값(온도/습도/CO₂/조도) + `growth_record`의 생육 점수 + 수확량을 조합합니다.
- Embedding Service에 내부용 엔드포인트 2종(`POST /api/v1/embeddings/insights` 배치 임베딩, `POST /api/v1/embeddings/insights/search` 유사 사례 검색)을 추가했습니다. 검색은 기존 챗봇 유사 사례 검색(Cosine Similarity Vector Search)과 달리, 버섯 종류(정확히 일치) + 온도(오차 범위)의 term/range 필터 기반입니다.
- Elasticsearch에 `cultivation_insight` 인덱스를 신설했습니다. `mushroom_environment`(5종 고정 참조 데이터, 이벤트 기반 갱신)와 달리 완료된 재배가 계속 쌓이는 이력성 인덱스이며, 스케줄러 기반 배치로 적재됩니다.
- 조회 결과는 Redis(`ai:{cultivationId}:insight`, TTL 24시간)에 캐시합니다. 다른 AI 캐시(챗봇/리포트/가이드)는 모두 push로 미리 채워지지만, 인사이트만 유일하게 사용자 요청 시점(pull)에 채워지는 캐시입니다. 유사 사례가 없어 고정 문구로 응답한 경우는 캐시하지 않습니다(다음 배치 이후 매칭될 수 있으므로).
- RabbitMQ 이벤트를 발행하지 않습니다. 배치 임베딩은 스케줄러가 OpenFeign으로 직접 오케스트레이션하고, 조회는 사용자 요청에 대한 동기 응답이라 별도 알림이 필요하지 않기 때문입니다(일일 피드백의 `DailyFeedbackCompletedEvent`와 다른 점).
- (자세한 내용은 [insight.md](./04_sequence/insight.md), [cultivation-db.md](./03_Database/cultivation-db.md), [cultivation-api.md](./02_API/cultivation-api.md), [cultivation.md](./01_Domain/cultivation.md), [ai.md](./01_Domain/ai.md), [ai-api.md](./02_API/ai-api.md), [embedding.md](./01_Domain/embedding.md), [embedding-api.md](./02_API/embedding-api.md), [elasticSearch.md](./03_Database/elasticSearch.md), [redis.md](./03_Database/redis.md), [database-overview.md](./03_Database/database-overview.md) 참고)

### 24. `sensor`/`environment_setting` 테이블과 그 API를 Cultivation Service에서 Sensor Service로 완전히 이관했다

팀원들과의 회의 결과, 결정 사항 10번에서 Cultivation Service로 옮겼던 센서 "장치" 메타데이터(`sensor`)와,
처음부터 Cultivation DB에 있던 목표 환경/위험 한계값 이력(`environment_setting`)을 **Sensor Service로
다시 이관**하기로 했습니다. 측정값(Redis/InfluxDB)을 이미 소유한 서비스가 장치 메타데이터와 그 장치가
쓰는 목표 범위까지 함께 갖는 것이 "센서 관련 데이터는 Sensor Service"라는 경계에 더 맞는다고 판단했습니다.
DB뿐 아니라 **API 소유권도 함께 완전히 이관**했습니다(Cultivation Service가 내부적으로 프록시하는 방식은
채택하지 않았습니다) — Sensor Service가 처음으로 PostgreSQL을 갖게 되었습니다.

- **DB 이관**: `sensor`, `environment_setting` 테이블이 Cultivation DB에서 새로 만든 Sensor DB로 옮겨졌습니다. 기존과 동일하게 `cultivation_id`는 실제 FK가 아닌 일반 BIGINT 컬럼입니다(Cultivation Service의 `cultivation.user_id`가 Auth Service의 `users.id`를 FK 없이 참조하던 것과 같은 패턴 — 서로 다른 DB이므로 DB 레벨 FK 자체가 불가능합니다). 자세한 DDL은 [sensor-db.md](./03_Database/sensor-db.md) 참고.
- **재배 생성 시 센서 등록은 그대로 `POST /cultivations`의 `devices`로 처리하되, 이제 하나의 로컬 트랜잭션이 아닙니다.** Cultivation Service가 `cultivation` 행을 먼저 만든 뒤, Sensor Service의 내부용 배치 등록 엔드포인트(`POST /api/v1/sensors/cultivations/{cultivationId}/batch`)를 OpenFeign으로 동기 호출합니다. 이 호출이 실패하면(예: `device_eui` 중복) Cultivation Service가 방금 만든 `cultivation` 행을 **보상 삭제(compensating delete)**합니다. 진짜 분산 트랜잭션(2PC 등)은 아니며, 실패 시 되돌리는 saga-lite 패턴임을 문서에 명시했습니다.
- **`ON DELETE CASCADE`를 이벤트로 대체**했습니다. 같은 DB 안에 있을 때는 재배 삭제 시 `sensor`/`environment_setting`이 CASCADE로 자동 삭제됐지만, 이제 다른 DB라 불가능합니다. 대신 Cultivation Service가 재배 삭제 시 새 `CultivationDeletedEvent`를 발행하고, Sensor Service가 이를 구독해 해당 `cultivation_id`의 `sensor`/`environment_setting` 행을 보상 삭제합니다.
- **소유권 검증도 API로 대체**했습니다. 같은 DB에서는 `cultivation_id`로 그냥 조인해서 소유자를 확인할 수 있었지만, 이제 Sensor Service가 쓰기 작업(개별 센서 등록/삭제, 환경 설정 저장) 전에 Cultivation Service의 새 내부용 엔드포인트 `GET /api/v1/cultivations/{cultivationId}/owner`(`{userId, status}` 반환)를 호출해 소유자를 확인합니다. 단, 배치 등록 경로는 예외입니다 — 호출 주체 자체가 Cultivation Service이므로 소유권 확인이 이미 보장되어 있어 생략합니다.
- **이벤트 발행 주체가 바뀌었습니다.** `SensorRegisteredEvent`/`SensorDeletedEvent`/`EnvironmentRangeUpdatedEvent`는 이제 Sensor Service가 발행합니다(구독자는 각각 DatasourceGenerator, DatasourceGenerator, Rule Engine Service로 기존과 동일). Rule Engine Service가 발행하는 `SensorErrorEvent`(센서 상태 갱신용)의 구독 주체는 Cultivation Service에서 Sensor Service로 바뀌었습니다.
- **환경 저장이 더 이상 재배 상태 전환과 한 트랜잭션이 아니게 되면서, CREATED → RUNNING 전환을 이벤트 기반으로 바꿨습니다.** 기존에는 "환경 저장 + 재배 상태를 CREATED에서 RUNNING으로" 자체가 Cultivation Service 내부의 한 트랜잭션이었지만, `environment_setting`이 Sensor Service로 이관되며 더 이상 불가능해졌습니다. 대신 Cultivation Service가 `EnvironmentRangeUpdatedEvent`를 구독해, 해당 재배에 대해 이 이벤트를 처음 받는 시점에 상태를 CREATED → RUNNING으로 전환합니다(이미 RUNNING/FINISHED면 아무 동작 없음 — 멱등).
- **Rule Engine Service의 Redis 캐시 미스 fallback 대상이 Cultivation Service에서 Sensor Service로 바뀌었습니다.** (결정 사항 7번 참고) 목표 환경 범위 캐시가 없을 때만 예외적으로 호출하는 대상이 이제 Sensor Service입니다.
- **AI Service의 환경 이력/평균 조회도 모두 Sensor Service로 옮겨졌습니다.** Daily Scheduler의 환경 변경 이력 조회, Insight Batch Scheduler의 환경 평균 조회가 모두 Sensor Service를 호출합니다. Insight Batch Scheduler는 재배 여러 건을 한 번에 조회해야 해서, N+1 호출을 피하기 위한 새 배치 엔드포인트 `POST /api/v1/sensors/environment-averages`(`cultivationIds` 배열 요청 → 평균 배열 응답, `environment_setting` 이력이 없는 재배는 결과에서 조용히 제외)를 Sensor Service에 추가했습니다.
- **인사이트 조회 시 `mushroomType`을 얻는 경로가 바뀌면서, 기존에 없던 자잘한 틈을 하나 발견해 함께 메웠습니다.** 기존에는 Cultivation Service의 환경 평균 조회 API가 `mushroomType`과 환경 평균을 함께 응답해줬는데, 환경 평균이 Sensor Service로 넘어가면서(Sensor Service는 `cultivation.mushroom_type`에 접근할 수 없어 이 값을 모릅니다) 두 값을 각각 다른 서비스에서 받아와야 하게 됐습니다. 그런데 확인해보니 Cultivation Service의 재배 상세 조회(`GET /{cultivationId}`) 응답에는 애초에 `mushroomType`이 없었습니다(이번 이관과 무관한 기존 누락). 이 참에 `mushroomType` 필드를 이 응답에 추가했고, AI Service는 이제 mushroomType은 Cultivation Service, 환경 평균은 Sensor Service, 이렇게 두 번 나눠 호출합니다.
- **에러 코드가 재정리되었습니다.** cultivation-api.md의 에러 코드가 C001~C009에서 **C001~C005**로 줄었고(센서/환경 관련 코드 제거 + 배치 등록 실패용 신규 C005 추가), 그만큼 sensor-api.md에 **S007~S011**이 새로 추가되었습니다.
- 이 결정은 결정 사항 10번(센서 장치 CRUD를 DatasourceGenerator → Cultivation Service로), 결정 사항 12번(DatasourceGenerator 캐시 재구성용 전체 목록 조회를 Cultivation Service에 추가) 등 과거 결정의 "소유 서비스" 부분을 뒤집습니다. 필드 구조(device_eui 등, 결정 사항 11번)나 범위 저장 방식(결정 사항 6번), 항목별 이력 구조(결정 사항 22번) 등 "설계 자체"는 그대로 유지되며, 옮겨간 곳이 바뀌었을 뿐입니다.
- (자세한 내용은 [sensor.md](./01_Domain/sensor.md), [sensor-api.md](./02_API/sensor-api.md), [sensor-db.md](./03_Database/sensor-db.md), [cultivation.md](./01_Domain/cultivation.md), [cultivation-api.md](./02_API/cultivation-api.md), [cultivation-db.md](./03_Database/cultivation-db.md), [create-cultivation.md](./04_sequence/create-cultivation.md), [daily-feedback.md](./04_sequence/daily-feedback.md), [insight.md](./04_sequence/insight.md), [rule-Engine.md](./01_Domain/rule-Engine.md), [rule-engine-api.md](./02_API/rule-engine-api.md), [ai.md](./01_Domain/ai.md), [ai-api.md](./02_API/ai-api.md), [datasource-generator.md](./01_Domain/datasource-generator.md), [datasource-generator-api.md](./02_API/datasource-generator-api.md), [datasource-generator-db.md](./03_Database/datasource-generator-db.md), [redis.md](./03_Database/redis.md), [database-overview.md](./03_Database/database-overview.md), [architecture.md](./00_Project/architecture.md) 참고)

---

# ✅ 해결된 이슈

- ~~`/api/v1/sensors` 경로 충돌~~ — DatasourceGenerator(장치 등록)와 Sensor Service(측정값 조회)가 같은 경로 프리픽스를 썼던 문제였습니다. 센서 장치 CRUD가 Cultivation Service(`/api/v1/cultivations/{id}/sensors`)로 이전되며 해소되었습니다. (결정 사항 10번 참고. **단, 결정 사항 24번에서 센서 장치 CRUD가 다시 Sensor Service로 이관되며 경로도 `/api/v1/sensors/cultivations/{id}`로 바뀌었습니다 — 원래의 경로 충돌 자체는 여전히 발생하지 않습니다, 새 경로 체계는 [sensor-api.md](./02_API/sensor-api.md) 개요 참고.**)

---

# ✍️ 문서 작성 규칙

새 문서를 추가할 때는 기존 문서와 동일한 형식을 유지해주세요.

- `01_Domain`: 역할 → 책임 → 주요 기능 → API → Database → Redis → 다른 서비스와의 통신 → Event → Sequence → 예외 상황 → 추후 개발 예정
- `02_API`: 개요(Base URL/인증방식) → 기능별 Request/Process/Response → Error Code → OpenFeign → Event
- `03_Database`: 개요 → ERD → Table → DDL → Index → 관계 → 고려 사항
- `04_sequence`: 개요 → Sequence(다이어그램) → 상세 과정 → 사용 Database → OpenFeign/RabbitMQ/MQTT → 예외 상황 → 고려 사항
