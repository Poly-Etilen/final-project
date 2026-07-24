# 🍄 EcoSphere / 버섯 재배 자동화 플랫폼 — 문서 목차

IoT 센서와 AI(LLM + Vision)를 활용한 개인 맞춤형 버섯 재배 자동화 플랫폼의
요구사항/설계 문서입니다.

팀원 배포용 문서이며, 새로 합류하는 팀원은 아래 "읽는 순서"를 따라가면 전체 구조를
빠르게 파악할 수 있습니다.

---

# 📁 폴더 구조

| 폴더 | 내용 |
|------|------|
| [00_Project](./00_Project) | 프로젝트 개요, 기능/기술 요구사항, 아키텍처, 로드맵 |
| [01_Domain](./01_Domain) | 서비스별 도메인 설계 (책임, 기능, 이벤트, 서비스 간 통신) |
| [03_Database](./03_Database) | 서비스별 DB 설계 (ERD/DDL/Index) 및 저장소별(Redis/InfluxDB/Photo Storage) 문서 |
| [04_sequence](./04_sequence) | 주요 시나리오별 서비스 간 시퀀스 다이어그램 |

REST API 명세(Request/Response/Error Code)는 이 문서 세트에 포함하지 않습니다.
각자 담당 서비스를 구현하며 `01_Domain`의 책임/기능과 `04_sequence`의 흐름을
기준으로 직접 작성합니다.

---

# 📖 읽는 순서 (신규 팀원 추천)

1. [00_Project/overview.md](./00_Project/overview.md) — 프로젝트가 무엇인지
2. [00_Project/architecture.md](./00_Project/architecture.md) — 서비스 구성과 전체 그림
3. [00_Project/tech-stack.md](./00_Project/tech-stack.md) — 사용 기술
4. 담당 서비스의 `01_Domain` 문서 — 내가 만들 서비스의 책임과 경계
5. 담당 서비스의 `03_Database` 문서 — 테이블/저장소 설계
6. 관련 `04_sequence` 문서 — 실제 요청이 여러 서비스를 어떻게 거치는지

---

# 🧱 설계 원칙

### MSA — Database per Service

서비스마다 별도 PostgreSQL 스키마를 소유합니다. 같은 DB 안의 관계는 실제 FK/UNIQUE
제약으로 강제하고, 서비스 경계를 넘는 참조(예: `cultivation_id`, `user_id`)는 DB
레벨 FK 없이 순수 값(소프트 참조)으로만 둡니다. 존재/소유권 검증이 필요하면
OpenFeign으로 확인합니다.

### 정규화 — 공통 도메인은 참조 테이블로

온도/습도/CO₂/조도라는 같은 측정 항목 집합이 여러 테이블에서 자유 문자열로
반복되지 않도록 `measurement_type` 참조 테이블(Cultivation DB)로 한 번만 정의하고,
`sensor_type`/`environment_setting`/`mushroom_reference_threshold`가 이를 FK로
참조합니다.

### 저장소 추상화 — 사진은 object_key + storage_type

생육 사진은 전체 URL을 저장하지 않고, 저장소 내 상대 경로(`object_key`)와 저장
위치(`storage_type`: MINIO/LOCAL)만 저장합니다. 실제 접근 경로 계산은 조회 시점에
서비스 설정 기준으로 이루어져, 저장소를 MinIO에서 로컬로(또는 그 반대로) 바꾸더라도
기존 데이터를 다시 쓸 필요가 없습니다.

### Soft Delete

회원 탈퇴는 `users` 행을 삭제하지 않고 `status = 'DELETED'` + `deleted_at`으로
표현합니다. 개인정보를 즉시 파기하지 않으며, 별도의 탈퇴 이력 테이블을 두지
않습니다.

### 임베딩·벡터 검색 없이 직접 조회

`mushroom_reference`(Cultivation DB, 5종 고정)와 `insight`(AI DB)는 모두 정확한 값
매칭/범위 비교로 충분히 검색되는 데이터입니다. 별도의 Embedding Service나
Elasticsearch 없이, 각 데이터를 소유한 서비스가 인덱스가 걸린 SQL 조회로 직접
제공합니다.

### 이벤트 기반 서비스 간 정합성

서비스 경계를 넘는 상태 전파(재배 비활성화, 상태 전환, 캐시 갱신 등)는 RabbitMQ
이벤트로 처리하며, 동기 호출(OpenFeign)은 즉시 응답이 필요하거나 소유권/존재
확인이 필요한 경우로 제한합니다.

---

# 🧩 서비스별 문서 매트릭스

API Gateway를 포함해 7개 서비스로 구성됩니다. Cultivation Service와 Sensor
Service는 트랜잭션 경계가 자주 겹쳐 하나의 서비스로 통합했습니다.

| 서비스 | 역할 요약 | Domain | Database | 관련 Sequence |
|--------|-----------|--------|----------|----------------|
| API Gateway | 라우팅, 인증 토큰 검증 | - | - | 전체 시퀀스 최초 진입점 |
| Auth | 인증(이메일+구글 소셜 로그인, `oauth_user`로 provider 정규화), JWT(role 클레임 포함), 이메일 인증, 휴면 계정 전환/재활성화, 회원 탈퇴(Soft Delete) | [auth.md](./01_Domain/auth.md) | [auth-db.md](./03_Database/auth-db.md) | [signup](./04_sequence/signup.md), [login](./04_sequence/login.md), [withdraw](./04_sequence/withdraw.md) |
| Cultivation | 재배 생성/조회/이력/종료, 수확(재배당 한 번) 기록, 상품 등급 매핑, 생육 사진 업로드, 센서 장치 CRUD, 목표 환경 범위 저장/조회/평균 계산, 버섯 참조 데이터 관리, 측정값 저장(Redis/InfluxDB)·조회·통계(일일 피드백용 일간 집계 포함), 문의(Inquiry) 등록/조회/관리자 답변·처리 | [cultivation.md](./01_Domain/cultivation.md) | [cultivation-db.md](./03_Database/cultivation-db.md), [influxdb.md](./03_Database/influxdb.md), [redis.md](./03_Database/redis.md) | [create-cultivation](./04_sequence/create-cultivation.md), [harvest](./04_sequence/harvest.md), [product-grade](./04_sequence/product-grade.md), [growth-analysis](./04_sequence/growth-analysis.md), [sensor-data](./04_sequence/sensor-data.md), [environment-control](./04_sequence/environment-control.md), [sensor-error](./04_sequence/sensor-error.md), [inquiry](./04_sequence/inquiry.md) |
| AI | 생육 분석(Vision), 챗봇(웹 WebSocket 채팅방+'/' 명령어 / Telegram·Discord 자연어 응답), 일일 피드백(생육 추이 비교 + 환경 통계), 인사이트(타인의 유사 재배 사례 후보 리스트+상세 조회), 상품 등급 원점수 계산, 버섯 가이드(효능/주의사항) | [ai.md](./01_Domain/ai.md) | [ai-db.md](./03_Database/ai-db.md), Redis(캐시), Photo Storage(읽기 전용) | [growth-analysis](./04_sequence/growth-analysis.md), [harvest](./04_sequence/harvest.md), [ai-chat](./04_sequence/ai-chat.md), [daily-feedback](./04_sequence/daily-feedback.md), [insight](./04_sequence/insight.md), [product-grade](./04_sequence/product-grade.md) |
| Rule Engine | MQTT 수신(Collector), 검증, 규칙 평가/자동 제어(중앙값 목표), 센서 오류 감지 | [rule-Engine.md](./01_Domain/rule-Engine.md) | Redis(목표 환경 범위 캐시만) | [sensor-data](./04_sequence/sensor-data.md), [environment-control](./04_sequence/environment-control.md), [sensor-error](./04_sequence/sensor-error.md) |
| Notification | Telegram/Discord 알림 발송, 재배 단위 알림 채널 등록/조회/삭제, 발송 이력 관리 | [notification.md](./01_Domain/notification.md) | [notification-db.md](./03_Database/notification-db.md) | [environment-control](./04_sequence/environment-control.md), [harvest](./04_sequence/harvest.md), [sensor-error](./04_sequence/sensor-error.md) |
| DatasourceGenerator | 센서 데이터 시뮬레이션 및 MQTT 발행 (REST API 없음) | [datasource-generator.md](./01_Domain/datasource-generator.md) | DB 없음, 메모리 캐시(`sensor_cache`)만 사용 | [sensor-data](./04_sequence/sensor-data.md) |

DB 전체 그림은 [database-overview.md](./03_Database/database-overview.md)에서 한
번에 볼 수 있습니다.

---

# 🗺️ 시퀀스 문서 목록

| 영역 | 문서 |
|------|------|
| 회원 | [signup](./04_sequence/signup.md), [login](./04_sequence/login.md), [withdraw](./04_sequence/withdraw.md) |
| 재배 | [create-cultivation](./04_sequence/create-cultivation.md), [harvest](./04_sequence/harvest.md), [product-grade](./04_sequence/product-grade.md) |
| 센서/환경 | [sensor-data](./04_sequence/sensor-data.md), [environment-control](./04_sequence/environment-control.md), [sensor-error](./04_sequence/sensor-error.md) |
| AI | [growth-analysis](./04_sequence/growth-analysis.md), [daily-feedback](./04_sequence/daily-feedback.md), [insight](./04_sequence/insight.md), [ai-chat](./04_sequence/ai-chat.md) |
| 문의 | [inquiry](./04_sequence/inquiry.md) |
