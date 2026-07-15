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

| 서비스 | 역할 요약 | Domain | API | Database | 관련 Sequence |
|--------|-----------|--------|-----|----------|----------------|
| Auth | 인증/인가, JWT, 이메일 인증 | [auth.md](./01_Domain/auth.md) | [auth-api.md](./02_API/auth-api.md) | [auth-db.md](./03_Database/auth-db.md) | [signup](./04_sequence/signup.md), [login](./04_sequence/login.md) |
| User | 회원 프로필, 탈퇴, 재배 통계 | [user.md](./01_Domain/user.md) | [user-api.md](./02_API/user-api.md) | [user-db.md](./03_Database/user-db.md) | [signup](./04_sequence/signup.md), [withdraw](./04_sequence/withdraw.md) |
| Cultivation | 재배 생성/관리/수확/사진 업로드 | [cultivation.md](./01_Domain/cultivation.md) | [cultivation-api.md](./02_API/cultivation-api.md) | [cultivation-db.md](./03_Database/cultivation-db.md) | [create-cultivation](./04_sequence/create-cultivation.md), [harvest](./04_sequence/harvest.md), [growth-analysis](./04_sequence/growth-analysis.md) |
| AI | 환경 추천, 생육 분석(Vision), 챗봇, 리포트 | [ai.md](./01_Domain/ai.md) | [ai-api.md](./02_API/ai-api.md) | Redis(캐시), MinIO(읽기 전용) | [growth-analysis](./04_sequence/growth-analysis.md), [harvest](./04_sequence/harvest.md), [ai-chat](./04_sequence/ai-chat.md), [ai-report](./04_sequence/ai-report.md) |
| Embedding | 재배 참조 데이터 임베딩·벡터 검색 | [embedding.md](./01_Domain/embedding.md) | [embedding-api.md](./02_API/embedding-api.md) | [elasticSearch.md](./03_Database/elasticSearch.md) | [create-cultivation](./04_sequence/create-cultivation.md) |
| Sensor | 환경 측정값 저장/조회/통계 | [sensor.md](./01_Domain/sensor.md) | [sensor-api.md](./02_API/sensor-api.md) | [influxdb.md](./03_Database/influxdb.md), [redis.md](./03_Database/redis.md) | [sensor-data](./04_sequence/sensor-data.md), [environment-control](./04_sequence/environment-control.md) |
| Rule Engine | 임계치 기반 자동 제어 판단 | [rule-Engine.md](./01_Domain/rule-Engine.md) | API 없음 (MQTT/RabbitMQ 기반) | 없음 | [environment-control](./04_sequence/environment-control.md), [sensor-error](./04_sequence/sensor-error.md) |
| Notification | WebSocket/Telegram/Discord 알림 | [notification.md](./01_Domain/notification.md) | API 없음 (RabbitMQ 기반) | 없음 | [environment-control](./04_sequence/environment-control.md), [harvest](./04_sequence/harvest.md), [sensor-error](./04_sequence/sensor-error.md) |
| Datasource | IoT 장치/센서 메타데이터 관리 | [datasource.md](./01_Domain/datasource.md) | [datasource-api.md](./02_API/datasource-api.md) | [datasource-db.md](./03_Database/datasource-db.md) | [sensor-data](./04_sequence/sensor-data.md) |

DB 전체 그림은 [database-overview.md](./03_Database/database-overview.md)에서 한 번에 볼 수 있습니다.

---

# 🕓 최근 주요 결정 사항 (변경 이력 요약)

문서가 여러 차례 업데이트되며 초기 설계와 달라진 부분입니다. 오래된 자료나 기억에 의존하지 말고 아래 최신 결정을 기준으로 작업해주세요.

### 1. 서비스는 필요 이상으로 쪼개지 않는다
Auth+User, Collector+RuleEngine+Storage 같은 과도한 세분화 대신, 지금의 9개 서비스(Auth/User/Cultivation/AI/Embedding/Sensor/Rule Engine/Notification/Datasource) + API Gateway 구조로 확정했습니다.

### 2. DB는 엔진별로 통합하고 스키마/네임스페이스로만 분리
서비스마다 별도 DB 인스턴스를 새로 만들지 않고, PostgreSQL(관계형) / InfluxDB(시계열) / Redis(캐시) / Elasticsearch(벡터) / MinIO(이미지) 5개 저장소를 서비스별로 나눠 쓰는 구조입니다.

### 3. AI 생육 분석은 "환경 계산"이 아니라 "사진 Vision 분석" 방식이다
처음엔 논문 기준값 대비 환경 유지율로 생육 상태를 계산하는 방식(방법 3)을 검토했지만,
최종적으로는 **사용자가 직접 촬영한 사진을 AI Vision 모델로 분석**하는 방식(방법 2)으로 확정했습니다.

- 카메라 센서가 자동 촬영하는 방식이 **아닙니다.** 사용자가 앱/웹에서 사진을 찍어 업로드합니다.
- 사진 업로드는 **Cultivation Service**가 담당합니다. (Datasource Service 아님)
- Vision 모델은 AI Service 내부에 포함되며, 사전에 학습된 성장 단계별 이미지와 비교해 균사 성장률/갓 크기/색상/병충해를 판정합니다.
- 판정된 수치는 결정론적 결과이며, **LLM은 그 결과를 해석하는 설명·개선 방안만 생성**합니다. 수치 자체를 LLM이 새로 추정하지 않습니다.

### 4. 사진 저장은 MinIO, 메타데이터는 Cultivation DB의 `photo` 테이블
이미지 원본은 MinIO(`mushroom-photos` 버킷)에, URL과 업로드 시각 같은 메타데이터는 Cultivation Service의 PostgreSQL에 저장합니다.

### 5. 회원 탈퇴는 Soft Delete + 이벤트 기반 후속 처리
User Service가 Soft Delete 처리 후 `UserDeletedEvent`를 발행하면, Auth Service(Refresh Token 삭제)와 Cultivation Service(재배 비활성화)가 각각 구독하여 처리합니다.

---

# ⚠️ 열려있는 이슈 (팀 확인 필요)

- **`/api/v1/sensors` 경로 충돌** — Datasource Service(장치 등록)와 Sensor Service(측정값 조회)가 같은 경로 프리픽스를 쓰고 있습니다. 자세한 내용과 대안은 [datasource-api.md](./02_API/datasource-api.md)의 상단 안내를 참고해 팀 논의 후 확정해주세요.

---

# ✍️ 문서 작성 규칙

새 문서를 추가할 때는 기존 문서와 동일한 형식을 유지해주세요.

- `01_Domain`: 역할 → 책임 → 주요 기능 → API → Database → Redis → 다른 서비스와의 통신 → Event → Sequence → 예외 상황 → 추후 개발 예정
- `02_API`: 개요(Base URL/인증방식) → 기능별 Request/Process/Response → Error Code → OpenFeign → Event
- `03_Database`: 개요 → ERD → Table → DDL → Index → 관계 → 고려 사항
- `04_sequence`: 개요 → Sequence(다이어그램) → 상세 과정 → 사용 Database → OpenFeign/RabbitMQ/MQTT → 예외 상황 → 고려 사항
