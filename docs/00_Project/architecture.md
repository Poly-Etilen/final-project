# 시스템 아키텍처

## 서비스 구성

8개 서비스(API Gateway 포함)로 구성합니다.

- API Gateway
- Auth Service (인증/회원 프로필/탈퇴)
- Cultivation Service (재배/수확/사진)
- AI Service (챗봇/생육 분석/리포트/일일 피드백/인사이트/버섯 가이드)
- Rule Engine Service (MQTT 수신/Collector, 검증, 규칙 평가, 자동 제어, 센서 오류 감지)
- Sensor Service (센서 장치/목표 환경/버섯 참조 데이터 관리, 측정값 저장/조회, 통계·차트, 주간 리포트 집계)
- Notification Service
- DatasourceGenerator

버섯 종류가 5종 고정이고, 인사이트 검색도 정확한 값/범위 필터로 충분해 별도의
Embedding Service나 벡터 검색 인프라(Elasticsearch)는 두지 않습니다. 관련 조회는
모두 PostgreSQL에 대한 직접 쿼리(OpenFeign 또는 자체 DB 조회)로 처리합니다.

Rule Engine Service와 Sensor Service는 기본적으로 RabbitMQ(`EnvironmentMeasuredEvent`)로
연결됩니다. 단, 목표 환경 범위(`environment_setting`) Redis 캐시가 없을 때(TTL 만료,
재시작 직후 등)에만 Rule Engine Service가 Sensor Service를 OpenFeign으로 예외적으로
호출하는 fallback이 있습니다.

---

## 데이터베이스

### PostgreSQL (서비스별 전용 DB)

- Auth DB — `users`, `oauth_user`
- Cultivation DB — `cultivation`, `harvest`, `photo`
- Sensor DB — `measurement_type`, `sensor`, `sensor_type`, `environment_setting`,
  `mushroom_reference`, `mushroom_reference_threshold`
- Notification DB — `notification_event`, `notification_delivery`, `notification_endpoint`
- AI DB — `chat_log`, `growth_record`, `daily_feedback`, `insight`

같은 DB 안의 관계는 실제 FK로 강제하고, 서비스 경계를 넘는 참조(`cultivation_id`
등)는 DB 레벨 FK 없이 순수 값으로만 둡니다.

---

### Redis

- Refresh Token / 이메일 인증번호 (Auth Service)
- AI 응답·생육 분석·리포트·인사이트·버섯 가이드 캐시 (AI Service)
- 목표 환경 범위 캐시 (Rule Engine Service)
- 최신 센서 데이터 (Sensor Service)

---

### InfluxDB

센서 시계열 데이터 저장 (Temperature, Humidity, CO₂, Light)

---

### Photo Storage (MinIO / Local)

사용자가 업로드한 생육 사진 저장. `photo`/`growth_record`는 전체 URL이 아닌
`object_key`(저장소 내 상대 경로) + `storage_type`(MINIO/LOCAL)만 저장해, 저장소를
바꾸더라도 기존 데이터를 다시 쓸 필요가 없습니다.

- Cultivation Service가 업로드
- AI Service가 Vision 분석을 위해 읽기 전용으로 조회

---

## 메시지 브로커

### MQTT

센서 데이터 수집

### RabbitMQ

서비스 간 비동기 이벤트 전달

---

## AI

### 환경 추천 (Cultivation/Sensor Service, AI 미사용)

```
Cultivation Service
↓
Sensor Service에 OpenFeign 호출 → mushroom_reference 조회 (PostgreSQL)
↓
환경 추천
```

버섯 종류가 공공데이터 기준 5가지로 고정되어 있어 Vector Search/LLM 없이 직접
조회합니다.

---

### 챗봇 / 버섯 가이드

```
Spring AI
↓
Sensor Service에서 mushroom_reference 직접 조회 (mushroomType 정확히 일치, RAG 컨텍스트)
↓
LLM
↓
챗봇 답변 / 버섯 가이드
```

버섯 종류가 5종 고정이라 재배의 `mushroomType` 하나에 대응하는 참조 텍스트를 그대로
조회해 LLM 컨텍스트로 사용하며, 별도의 벡터 검색은 필요하지 않습니다. 챗봇은
웹/앱(APP) 채널뿐 아니라 Telegram/Discord 봇으로도 사용할 수 있습니다.

---

### 생육 분석 (Vision)

```
사용자가 촬영한 사진 업로드 (Cultivation Service → Photo Storage)
↓
AI Service Vision 모델
↓
균사 성장률 / 갓 크기 / 색상 / 병충해 판별
↓
LLM (결과 해석 및 개선 방안 생성)
↓
growth_record 영구 저장 (AI DB)
```

---

### 일일 피드백

```
Daily Scheduler (AI Service, 매일 23시)
↓
growth_record(생육 추이) + environment_setting(환경 변경 이력) 비교
↓
LLM 해석 (또는 사진 없으면 고정 문구)
↓
daily_feedback 저장
```

---

### 인사이트 (타인의 유사 재배 사례 기반 피드백)

```
수확 완료 (Cultivation Service → HarvestCompletedEvent)
↓
AI Service가 구독 → 환경 평균/생육 점수 조합 → insight 테이블 저장 (AI DB)

사용자 요청 시 (조회, 적재와 별개)
↓
insight 테이블 SQL 검색 (mushroom_type 정확히 일치 + avg_temperature 오차 범위)
↓
LLM (유사 사례 요약)
```

일일 피드백(자기 자신의 이력 비교)과 달리 타인의 사례와 비교하며, 스케줄 기반
push가 아닌 사용자 요청 시점(on-demand)에만 조회됩니다. 사례 적재도 배치가 아니라
수확이 기록될 때마다 즉시 이루어지며, 검색 조건이 정확한 값/범위 필터라 별도의
벡터 검색 없이 인덱스가 걸린 SQL 조회로 충분합니다.
