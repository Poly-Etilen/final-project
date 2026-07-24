# AI Database

## 개요

AI Database는 AI 챗봇의 대화 이력(`chat_log`), 생육 분석 이력(`growth_record`), 일일
피드백 이력(`daily_feedback`), "인사이트" 사례 이력(`insight`)을 관리하는 AI Service 소유의
PostgreSQL Database입니다. 네 테이블 모두 `cultivation_id`로만 다른 서비스 데이터를
참조하며, 서로 간 FK도 없습니다(같은 DB 안에서도 소프트 참조).

`ai:{hash}`/`ai:{cultivationId}:analysis` 같은 Redis 캐시는 "빠른 재조회/중복 호출
방지"용 성능 캐시이고, 이 DB는 "이력 조회"용 영구 저장소입니다. 역할이 달라 함께
유지됩니다.

---

# ERD

```
chat_log
──────────────────────────────────────────────
PK  id
    user_id           (nullable)
    cultivation_id    (nullable)
    sender_role
    content
    channel_type
    created_at

growth_record
──────────────────────────────────────────────
PK  id
    cultivation_id
    object_key
    storage_type
    growth_score
    mycelium_growth_rate
    cap_size
    color_status
    disease_status
    growth_stage
    expected_harvest_date
    analyzed_at

daily_feedback
──────────────────────────────────────────────
PK  id
    cultivation_id
    feedback_date
    has_growth_data
    content
    avg_temperature
    avg_humidity
    avg_co2
    avg_light
    max_temperature
    min_temperature
    created_at

    UNIQUE(cultivation_id, feedback_date)

insight
──────────────────────────────────────────────
PK  id
    harvest_id        (UNIQUE)
    cultivation_id
    mushroom_type
    avg_temperature
    avg_humidity
    avg_co2
    avg_light
    growth_score
    harvest_weight
    summary
    created_at
```

---

# Table

## chat_log

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | O | 대화 주체 사용자 (Auth Service, 소프트 참조). APP 채널은 항상 채워지고, TELEGRAM/DISCORD는 NULL |
| cultivation_id | BIGINT | O | 대화 대상 재배 (Cultivation Service, 소프트 참조). APP은 선택적, TELEGRAM/DISCORD는 항상 채워짐 |
| sender_role | VARCHAR(20) | X | 발화 주체 (USER/BOT) |
| content | TEXT | X | 발화 내용 |
| channel_type | VARCHAR(20) | X | 대화 채널 (APP/TELEGRAM/DISCORD) |
| created_at | DATETIME | X | 발화 시각 |

발화 하나당 한 행으로 쌓입니다. 사용자 질문 하나에 챗봇 응답 하나가 따르면 `sender_role`이
다른 두 행(`USER` → `BOT`)이 쌓입니다.

- **APP**: `user_id`는 Bearer JWT에서 가져오고, `cultivation_id`는 요청 시 지정한
  재배(선택)입니다.
- **TELEGRAM/DISCORD**: 발신자의 Chat ID로 `notification_endpoint`를 조회해
  `cultivation_id`를 알아냅니다(알림 채널이 재배 단위로 등록되어 있으므로). 이 경로로는
  개별 사용자를 특정할 수 없어 `user_id`는 NULL입니다.

---

## growth_record

사용자가 생육 사진을 업로드할 때마다 Vision 분석 결과를 한 행씩 쌓는 이력 테이블입니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 분석 대상 재배 (소프트 참조) |
| object_key | VARCHAR(500) | X | 분석에 사용한 사진의 저장소 내 경로 (Cultivation Service `photo.object_key` 복사본) |
| storage_type | VARCHAR(10) | X | 사진 저장 위치 (MINIO/LOCAL) |
| growth_score | INT | X | 생육 점수 (균사 40% + 갓크기 30% + 색상 15% + 병충해 15%) |
| mycelium_growth_rate | NUMERIC(4,1) | X | 균사 성장률(%) |
| cap_size | VARCHAR(30) | O | 갓 크기 (예: "중(3.2cm)") |
| color_status | VARCHAR(20) | X | 색상 상태 |
| disease_status | VARCHAR(20) | X | 병충해 상태 |
| growth_stage | VARCHAR(30) | X | 성장 단계 |
| expected_harvest_date | DATE | O | 예상 수확일 |
| analyzed_at | DATETIME | X | 분석 시각 |

`object_key`/`storage_type`은 Cultivation Service가 분석 요청 시 전달한 값을 그대로
복사해 저장합니다 — AI DB는 Cultivation DB의 `photo`를 FK로 참조할 수 없으므로, 분석
시점의 사진 위치를 스냅샷으로 남기는 방식입니다. `growth_score`~`expected_harvest_date`는
`POST /ai/analysis` 응답과 동일한 값입니다.

---

## daily_feedback

Daily Scheduler가 매일 재배별로 생성하는 일일 피드백 결과입니다. 생육 추이 비교
내용과 함께, 그날의 환경 통계(최근 24시간 집계)도 같은 행에 저장합니다 — 재배
기간이 한 달을 넘지 않는 도메인 특성상 별도의 주간/월간 리포트 테이블을 두지 않고
이 테이블 하나로 통합했습니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 대상 재배 (소프트 참조) |
| feedback_date | DATE | X | 이 피드백이 다루는 날짜 |
| has_growth_data | BOOLEAN | X | 그날 `growth_record`가 있었는지 여부 |
| content | TEXT | X | 피드백 본문 (생육 비교 + 환경 통계 해석) |
| avg_temperature | NUMERIC(5,2) | O | 최근 24시간 평균 온도 |
| avg_humidity | NUMERIC(5,2) | O | 최근 24시간 평균 습도 |
| avg_co2 | NUMERIC(8,2) | O | 최근 24시간 평균 CO₂ |
| avg_light | NUMERIC(6,2) | O | 최근 24시간 평균 조도 |
| max_temperature | NUMERIC(5,2) | O | 최근 24시간 최고 온도 |
| min_temperature | NUMERIC(5,2) | O | 최근 24시간 최저 온도 |
| created_at | DATETIME | X | 생성 시각 |

`has_growth_data`가 false이면 `content`의 생육 비교 부분에는 고정 안내 문구가
들어갑니다. 환경 통계 컬럼들은 `has_growth_data`와 무관하게 Cultivation Service의
InfluxDB 집계 조회 결과로 매일 채워지며, 그날 측정값이 전혀 없었던 경우(예: 재배
생성 당일)에만 NULL로 남습니다. `UNIQUE(cultivation_id, feedback_date)`로 하루에
한 번만 생성되도록 합니다.

---

## insight

"인사이트" 기능(같은 버섯 종류 + 유사한 환경으로 재배했던 타인의 사례를 보여주는 기능)의
사례 데이터입니다. Cultivation Service가 수확을 기록하면 발행하는 `HarvestCompletedEvent`를
AI Service가 구독해, 그 수확 건 하나당 행 하나를 즉시 저장합니다. 별도의 배치나 임계치
없이 수확 시점마다 반영됩니다.

조회 시에는 `mushroom_type`(정확히 일치) + `avg_temperature`/`avg_humidity`/`avg_co2`/
`avg_light`(4개 항목 모두 오차 범위) 조건으로 후보를 찾아 최신순 최대 5개를 반환하고,
사용자가 그중 하나를 선택하면 같은 `cultivation_id`의 `daily_feedback`을 날짜순으로 함께
조회해 보여줍니다(수확일에는 `summary`로 대체). 자세한 흐름은
[insight.md](../04_sequence/insight.md) 참고.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| harvest_id | BIGINT | X | 수확 참조 (Cultivation Service, 소프트 참조), UNIQUE |
| cultivation_id | BIGINT | X | 재배 참조 (소프트 참조) |
| mushroom_type | VARCHAR(20) | X | 버섯 종류 (검색 필터용) |
| avg_temperature | NUMERIC(5,2) | O | 기간 가중 평균 온도 |
| avg_humidity | NUMERIC(5,2) | O | 기간 가중 평균 습도 |
| avg_co2 | NUMERIC(8,2) | O | 기간 가중 평균 CO₂ |
| avg_light | NUMERIC(6,2) | O | 기간 가중 평균 조도 |
| growth_score | INT | O | 생육 점수 (`growth_record` 기준 마지막 분석 결과) |
| harvest_weight | NUMERIC(6,1) | O | 수확량(g) |
| summary | TEXT | X | 이 사례를 설명하는 한 문장 요약 (조회 시 LLM 컨텍스트로 재사용) |
| created_at | DATETIME | X | 저장 시각 |

`harvest_id`는 UNIQUE 제약을 둡니다 — 이벤트가 중복 전달되더라도 같은 수확 건이 두 번
적재되지 않도록 막기 위함입니다. `harvest`가 `cultivation`과 1:1이라(재배당 한 건)
`cultivation_id`도 사실상 함께 유일하지만, insight의 발생 단위 자체는 "수확 건"이라
`harvest_id`를 기준 참조로 둡니다. 인사이트 검색은 벡터 유사도가 아니라
`mushroom_type`(정확히 일치) + `avg_temperature`(오차 범위) 필터로 수행하므로, 임베딩
(Vector) 컬럼은 두지 않습니다.

---

# DDL

```sql
CREATE TABLE chat_log (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT,

    cultivation_id BIGINT,

    sender_role VARCHAR(20) NOT NULL,

    content TEXT NOT NULL,

    channel_type VARCHAR(20) NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_chat_log_sender_role CHECK (sender_role IN ('USER', 'BOT')),

    CONSTRAINT chk_chat_log_channel_type CHECK (channel_type IN ('APP', 'TELEGRAM', 'DISCORD'))
);
```

```sql
CREATE TABLE growth_record (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    object_key VARCHAR(500) NOT NULL,

    storage_type VARCHAR(10) NOT NULL,

    growth_score INT NOT NULL,

    mycelium_growth_rate NUMERIC(4,1) NOT NULL,

    cap_size VARCHAR(30),

    color_status VARCHAR(20) NOT NULL,

    disease_status VARCHAR(20) NOT NULL,

    growth_stage VARCHAR(30) NOT NULL,

    expected_harvest_date DATE,

    analyzed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_growth_record_storage_type CHECK (storage_type IN ('MINIO', 'LOCAL'))
);
```

```sql
CREATE TABLE daily_feedback (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    feedback_date DATE NOT NULL,

    has_growth_data BOOLEAN NOT NULL,

    content TEXT NOT NULL,

    avg_temperature NUMERIC(5,2),

    avg_humidity NUMERIC(5,2),

    avg_co2 NUMERIC(8,2),

    avg_light NUMERIC(6,2),

    max_temperature NUMERIC(5,2),

    min_temperature NUMERIC(5,2),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uk_daily_feedback_cultivation_date UNIQUE (cultivation_id, feedback_date)
);
```

```sql
CREATE TABLE insight (
    id BIGSERIAL PRIMARY KEY,

    harvest_id BIGINT NOT NULL,

    cultivation_id BIGINT NOT NULL,

    mushroom_type VARCHAR(20) NOT NULL,

    avg_temperature NUMERIC(5,2),

    avg_humidity NUMERIC(5,2),

    avg_co2 NUMERIC(8,2),

    avg_light NUMERIC(6,2),

    growth_score INT,

    harvest_weight NUMERIC(6,1),

    summary TEXT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uk_insight_harvest UNIQUE (harvest_id)
);
```

네 테이블 모두 `cultivation_id`가 다른 DB(Cultivation Service)를 가리키는 소프트 참조라
DB 레벨 FK는 걸지 않습니다.

---

# Index

```sql
CREATE INDEX idx_chat_log_cultivation
ON chat_log(cultivation_id, created_at DESC);
```

재배 기준 대화 이력 조회에 사용합니다. `cultivation_id`가 NULL인 행은 조회되지 않습니다.

```sql
CREATE INDEX idx_chat_log_user_channel
ON chat_log(user_id, channel_type, created_at DESC);
```

사용자별·채널별 대화 이력 조회에 사용합니다. `user_id`가 NULL인 TELEGRAM/DISCORD 발화는
조회되지 않습니다.

```sql
CREATE INDEX idx_growth_record_cultivation
ON growth_record(cultivation_id, analyzed_at DESC);
```

일일 피드백이 최근 며칠간의 추이를 조회할 때 사용합니다.

```sql
CREATE INDEX idx_insight_mushroom_temp
ON insight(mushroom_type, avg_temperature);
```

"인사이트" 검색이 버섯 종류(정확히 일치) + 온도(범위)로 1차 필터링하는 패턴이라 이
복합 인덱스가 필요합니다. `avg_humidity`/`avg_co2`/`avg_light`도 함께 오차 범위로
필터링하지만, 온도 필터로 이미 후보군이 충분히 좁혀질 것으로 예상되어 전용 인덱스는
따로 두지 않았습니다. 정렬은 `created_at` 최신순입니다.

---

# 관계

```
chat_log / growth_record / daily_feedback / insight (AI Service)

    │ user_id (chat_log만, APP 채널 발화에만 존재)
    ▼
Auth Service

    │ cultivation_id (네 테이블 공통)
    │ harvest_id (insight만)
    ▼
Cultivation Service
```

AI Service는 다른 서비스와 데이터베이스를 공유하지 않습니다. `insight`는 이 DB에만
존재하며 별도의 검색 사본을 두지 않습니다 — 검색 조건(버섯 종류 정확히 일치 + 온도
범위)이 이 테이블의 인덱스(`idx_insight_mushroom_temp`)로 충분히 처리됩니다.

---

# 고려 사항

- `ai:{hash}`/`ai:{cultivationId}:analysis` Redis 캐시(성능 최적화용)와 이 DB(이력
  조회용)는 역할이 달라 함께 유지합니다.
- `chat_log`/`growth_record`/`daily_feedback`의 내용은 매번 새 행으로 쌓이며 기존 행을
  수정하지 않으므로 `updated_at`을 두지 않았습니다.
- `daily_feedback`은 사진을 안 찍은 날도 반드시 한 행이 생성됩니다(`has_growth_data =
  false`) — "피드백 없음"과 "비교 데이터 없음"을 구분하기 위함입니다.
- `daily_feedback`의 환경 통계 컬럼(`avg_temperature`~`min_temperature`)은 별도의
  주간/월간 리포트 테이블을 두지 않고 이 테이블에 통합한 결과입니다.
- `insight.harvest_id`는 UNIQUE 제약으로 같은 수확 건의 중복 적재를 막습니다.
- `insight`와 `daily_feedback`은 같은 DB에 있어, 인사이트 후보 상세 조회 시 선택한
  `insight.cultivation_id`로 `daily_feedback`을 바로 조회할 수 있습니다(Cultivation
  Service 등 다른 서비스 호출 불필요).
- 데이터가 무한히 쌓이는 이력성 테이블들이라, 운영 단계에서는 오래된 데이터에 대한 보관
  주기 정책이 필요할 수 있습니다(추후 개발 예정).
