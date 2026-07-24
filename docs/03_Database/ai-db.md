# AI Database

## 개요

AI Database는 AI 챗봇의 대화 이력(`chat_conversation`/`chat_message`), 생육 분석
이력(`growth_record`), 일일 피드백 이력(`daily_feedback`), "인사이트" 사례
이력(`insight`)을 관리하는 AI Service 소유의 PostgreSQL Database입니다.

`chat_message`만 같은 DB 안의 `chat_conversation`을 실제 FK로 참조하고, 나머지는
모두 `cultivation_id`/`user_id` 등으로 다른 서비스 데이터를 가리키는 소프트
참조입니다.

`ai:{hash}`/`ai:{cultivationId}:analysis` 같은 Redis 캐시는 "빠른 재조회/중복 호출
방지"용 성능 캐시이고, 이 DB는 "이력 조회"용 영구 저장소입니다. 역할이 달라 함께
유지됩니다.

이 문서는 외부 팀이 제공한 ERD(DDL)를 기준으로 전면 재작성되었습니다. 외부 ERD의
`mushroom_embedding` 기반 벡터 검색 기능은 이 프로젝트에서 채택하지 않아 제외했으며,
그 외의 구조 변경(대화-메시지 분리, `growth_record` JSONB화, `insight` 키 변경 등)은
모두 반영했습니다. 외부 ERD와 다르게 판단한 지점은 [고려 사항](#고려-사항)에
근거를 남겼습니다.

---

# ERD

```
chat_conversation
──────────────────────────────────────────────
PK  id
    user_id                    (nullable)
    cultivation_id             (nullable)
    channel_type
    external_conversation_id
    created_at
    updated_at

chat_message
──────────────────────────────────────────────
PK  id
FK  chat_conversation_id  ──▶ chat_conversation.id
    role
    content
    sequence_number
    created_at

growth_record
──────────────────────────────────────────────
PK  id
    cultivation_id
    cultivation_photo_id
    analysis_data      (JSONB)
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
    cultivation_id     (UNIQUE)
    mushroom_id
    avg_temperature
    avg_humidity
    avg_co2
    avg_light
    growth_score
    harvest_weight_grams
    summary
    created_at
```

---

# Table

## chat_conversation

대화방(세션) 단위 테이블입니다. `chat_log` 단일 테이블 구조를 대화-메시지로
분리하면서 새로 생겼습니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | O | 대화 주체 사용자 (Auth Service, 소프트 참조). `notification_endpoint`가 사용자 단위로 등록되므로 TELEGRAM/DISCORD도 발신자를 특정할 수 있어 채워짐 — NULL 허용은 방어적 설계이며 이유는 [고려 사항](#고려-사항) 참고 |
| cultivation_id | BIGINT | O | 대화 대상 재배 (Cultivation Service, 소프트 참조). APP은 선택적, TELEGRAM/DISCORD는 항상 채워짐 |
| channel_type | VARCHAR(20) | X | 대화 채널 (APP/TELEGRAM/DISCORD) |
| external_conversation_id | TEXT | X | Telegram/Discord는 해당 플랫폼의 대화(Chat) 식별자, APP은 자체 세션 식별자 |
| created_at | DATETIME | X | 대화방 생성 시각 |
| updated_at | DATETIME | X | 마지막 발화 시각(메시지가 추가될 때마다 갱신) |

같은 `(channel_type, external_conversation_id)`로 다시 들어오면 새 대화방을 만들지
않고 기존 행에 메시지를 이어 붙입니다(TELEGRAM/DISCORD는 같은 Chat ID로 계속
재사용, APP은 같은 WebSocket 세션이 유지되는 동안 재사용).

- **APP**: `user_id`는 Bearer JWT에서 가져오고, `cultivation_id`는 연결 시 지정한
  재배(선택)입니다.
- **TELEGRAM/DISCORD**: 발신자의 Chat ID로 `notification_endpoint`(사용자 단위 등록)를
  조회해 `user_id`를 바로 알아내고, 그 endpoint의 활성 `notification_subscription`
  목록에서 구독 중인 재배(`target_id`)가 정확히 하나면 `cultivation_id`도 함께
  채웁니다. 같은 endpoint가 여러 재배를 구독 중이어서 대상을 특정할 수 없는 경우는
  애초에 `chat_conversation`을 생성하지 않고 안내 문구로만 응답합니다.

---

## chat_message

개별 발화 하나당 한 행입니다. 기존 `chat_log`의 후신이며, `sender_role`은
`role`로 이름이 바뀌었습니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| chat_conversation_id | BIGINT | X | 소속 대화방 (같은 DB 내 실제 FK → `chat_conversation.id`) |
| role | VARCHAR(20) | X | 발화 주체 (USER/BOT) |
| content | TEXT | X | 발화 내용 |
| sequence_number | BIGINT | X | 대화방 내 발화 순서(1, 2, 3 ...) |
| created_at | DATETIME | X | 발화 시각 |

사용자 질문 하나에 챗봇 응답 하나가 따르면 `role`이 다른 두 행(`USER` →
`BOT`)이 같은 `chat_conversation_id`에 연속된 `sequence_number`로 쌓입니다.
`sequence_number`는 애플리케이션이 저장 직전 `chat_conversation_id` 기준 최댓값 + 1로
채우며, DB 시퀀스나 트리거를 별도로 두지 않습니다.

---

## growth_record

사용자가 생육 사진을 업로드할 때마다 Vision 분석 결과를 한 행씩 쌓는 이력 테이블입니다.
기존에는 분석 지표를 컬럼별로 정규화해 저장했지만, 이제는 JSONB 하나에 담습니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 분석 대상 재배 (소프트 참조) |
| cultivation_photo_id | BIGINT | X | 분석에 사용한 사진 (Cultivation DB `cultivation_photo.id`, 소프트 참조) |
| analysis_data | JSONB | X | 분석 결과 전체 (아래 예시 참고) |
| analyzed_at | DATETIME | X | 분석 시각 |

`analysis_data` 예시:

```json
{
    "growthScore": 85,
    "myceliumGrowthRate": 82.0,
    "capSize": "중(3.2cm)",
    "colorStatus": "정상",
    "diseaseStatus": "정상",
    "growthStage": "자실체 형성기",
    "expectedHarvestDate": "2026-08-20"
}
```

`growthScore`(생육 점수: 균사 40% + 갓크기 30% + 색상 15% + 병충해 15%),
`myceliumGrowthRate`, `capSize`, `colorStatus`, `diseaseStatus`, `growthStage`,
`expectedHarvestDate`가 모두 `POST /ai/analysis` 응답과 동일한 값으로 이 JSON 안에
담깁니다. 특정 필드만 조회하려면 PostgreSQL의 JSONB 연산자를 사용합니다(예:
`analysis_data->>'growthScore'`, `analysis_data->>'growthStage'`).

`cultivation_photo_id`는 사진 자체(파일 경로 등)를 스냅샷으로 복사하지 않고 ID만
저장합니다. 사진 원본이 필요하면 Cultivation Service에 별도 조회가 필요합니다 —
자세한 배경은 [고려 사항](#고려-사항) 참고.

---

## daily_feedback

Daily Scheduler가 매일 재배별로 생성하는 일일 피드백 결과입니다. 생육 추이 비교
내용과 함께, 그날의 환경 통계(최근 24시간 집계)도 같은 행에 저장합니다 — 재배
기간이 한 달을 넘지 않는 도메인 특성상 별도의 주간/월간 리포트 테이블을 두지 않고
이 테이블 하나로 통합했습니다. 외부 ERD 대비 이 프로젝트가 의도적으로 확장한
부분입니다([고려 사항](#고려-사항) 참고).

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

조회 시에는 `mushroom_id`(정확히 일치) + `avg_temperature`/`avg_humidity`/`avg_co2`/
`avg_light`(4개 항목 모두 오차 범위) 조건으로 후보를 찾아 최신순 최대 5개를 반환하고,
사용자가 그중 하나를 선택하면 같은 `cultivation_id`의 `daily_feedback`을 날짜순으로 함께
조회해 보여줍니다(수확일에는 `summary`로 대체). 자세한 흐름은
[insight.md](../04_sequence/insight.md) 참고.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 재배 참조 (소프트 참조), UNIQUE |
| mushroom_id | BIGINT | X | 버섯 참조 ID (검색 필터용, Cultivation Service 소프트 참조) |
| avg_temperature | NUMERIC(5,2) | X | 기간 가중 평균 온도 |
| avg_humidity | NUMERIC(5,2) | X | 기간 가중 평균 습도 |
| avg_co2 | NUMERIC(8,2) | X | 기간 가중 평균 CO₂ |
| avg_light | NUMERIC(6,2) | X | 기간 가중 평균 조도 |
| growth_score | INT | O | 생육 점수 (`growth_record` 기준 마지막 분석 결과) |
| harvest_weight_grams | NUMERIC(6,2) | X | 수확량(g) |
| summary | TEXT | X | 이 사례를 설명하는 한 문장 요약 (조회 시 LLM 컨텍스트로 재사용) |
| created_at | DATETIME | X | 저장 시각 |

`cultivation_id`는 UNIQUE 제약을 둡니다 — `harvest`가 `cultivation`과 1:1(재배당
한 건)이므로, 이벤트가 중복 전달되더라도 같은 재배의 사례가 두 번 적재되지 않도록
막는 효과는 기존 `harvest_id` UNIQUE와 기능적으로 동일합니다. `HarvestCompletedEvent`
payload 자체는 `harvestId`/`cultivationId`/`harvestWeight`를 그대로 담아 오지만,
`insight` 테이블에 저장할 때의 dedup 키는 `cultivation_id`입니다. 인사이트 검색은
벡터 유사도가 아니라 `mushroom_id`(정확히 일치) + `avg_temperature`(오차 범위)
필터로 수행하므로, 임베딩(Vector) 컬럼은 두지 않습니다.

`avg_temperature`~`avg_light`는 외부 ERD 기준 NOT NULL로 맞췄습니다(적재 시점에
Cultivation Service의 환경 평균 조회가 실패하면 애초에 이 행 자체를 저장하지 않고
건너뛰므로, 저장되는 행은 항상 값이 채워져 있습니다). `harvest_weight_grams`도
외부 ERD 기준으로 컬럼명·타입·NULL 여부를 맞췄습니다(기존 `harvest_weight
NUMERIC(6,1)`, nullable → `harvest_weight_grams NUMERIC(6,2)`, NOT NULL).

---

# DDL

```sql
CREATE TABLE chat_conversation (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT,

    cultivation_id BIGINT,

    channel_type VARCHAR(20) NOT NULL,

    external_conversation_id TEXT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_chat_conversation_channel_type CHECK (channel_type IN ('APP', 'TELEGRAM', 'DISCORD'))
);
```

```sql
CREATE TABLE chat_message (
    id BIGSERIAL PRIMARY KEY,

    chat_conversation_id BIGINT NOT NULL,

    role VARCHAR(20) NOT NULL,

    content TEXT NOT NULL,

    sequence_number BIGINT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_chat_message_conversation
        FOREIGN KEY (chat_conversation_id) REFERENCES chat_conversation(id),

    CONSTRAINT chk_chat_message_role CHECK (role IN ('USER', 'BOT')),

    CONSTRAINT uk_chat_message_conversation_seq UNIQUE (chat_conversation_id, sequence_number)
);
```

```sql
CREATE TABLE growth_record (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    cultivation_photo_id BIGINT NOT NULL,

    analysis_data JSONB NOT NULL,

    analyzed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
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

    cultivation_id BIGINT NOT NULL,

    mushroom_id BIGINT NOT NULL,

    avg_temperature NUMERIC(5,2) NOT NULL,

    avg_humidity NUMERIC(5,2) NOT NULL,

    avg_co2 NUMERIC(8,2) NOT NULL,

    avg_light NUMERIC(6,2) NOT NULL,

    growth_score INT,

    harvest_weight_grams NUMERIC(6,2) NOT NULL,

    summary TEXT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uk_insight_cultivation UNIQUE (cultivation_id)
);
```

`chat_message.chat_conversation_id`만 같은 DB 안의 실제 FK입니다. 나머지
`user_id`/`cultivation_id`/`cultivation_photo_id`/`mushroom_id`는 모두 다른
서비스(Auth/Cultivation) 소유 데이터를 가리키는 소프트 참조라 DB 레벨 FK를 걸지
않습니다.

---

# Index

```sql
CREATE UNIQUE INDEX uk_chat_conversation_external
ON chat_conversation(channel_type, external_conversation_id);
```

같은 채널의 같은 외부 대화(Telegram/Discord Chat, APP 세션)로 다시 들어왔을 때
새 대화방을 만들지 않고 기존 행을 찾아 재사용하기 위한 조회/유일성 보장 인덱스입니다.

```sql
CREATE INDEX idx_chat_conversation_cultivation
ON chat_conversation(cultivation_id, created_at DESC);
```

재배 기준 대화 이력 조회에 사용합니다. `cultivation_id`가 NULL인 행은 조회되지 않습니다.

```sql
CREATE INDEX idx_chat_conversation_user_channel
ON chat_conversation(user_id, channel_type, created_at DESC);
```

사용자별·채널별 대화방 목록 조회에 사용합니다. `user_id`가 NULL인 TELEGRAM/DISCORD
대화방은 조회되지 않습니다.

```sql
CREATE INDEX idx_chat_message_conversation_seq
ON chat_message(chat_conversation_id, sequence_number);
```

대화방 하나를 골라 발화 순서대로 이어 보여줄 때 사용합니다(`uk_chat_message_conversation_seq`
UNIQUE 제약이 사실상 이 인덱스를 겸하지만, 조회 패턴을 명시하기 위해 별도로 기재합니다).

```sql
CREATE INDEX idx_growth_record_cultivation
ON growth_record(cultivation_id, analyzed_at DESC);
```

일일 피드백이 최근 며칠간의 추이를 조회할 때 사용합니다. `analysis_data`가
JSONB로 바뀌었지만 현재 조회 패턴상 `cultivation_id` + `analyzed_at` 정렬이 핵심이라
이 인덱스는 그대로 유지합니다. `growth_stage` 등 JSONB 내부 필드로 자주 필터링하는
조회가 늘어나면 PostgreSQL의 GIN 인덱스나 표현식 인덱스(예:
`((analysis_data->>'growthStage'))`)를 추가하는 것을 검토할 수 있습니다 — 현재는
그런 조회 패턴이 없어 추가하지 않았습니다.

```sql
CREATE UNIQUE INDEX uk_insight_cultivation
ON insight(cultivation_id);
```

`insight`의 UNIQUE 제약을 위한 인덱스입니다(DDL의 `uk_insight_cultivation`
제약과 동일).

```sql
CREATE INDEX idx_insight_mushroom_temp
ON insight(mushroom_id, avg_temperature);
```

"인사이트" 검색이 버섯 종류(정확히 일치) + 온도(범위)로 1차 필터링하는 패턴이라 이
복합 인덱스가 필요합니다. `avg_humidity`/`avg_co2`/`avg_light`도 함께 오차 범위로
필터링하지만, 온도 필터로 이미 후보군이 충분히 좁혀질 것으로 예상되어 전용 인덱스는
따로 두지 않았습니다. 정렬은 `created_at` 최신순입니다.

---

# 관계

```
chat_message (AI Service)
    │ chat_conversation_id (실제 FK, 같은 DB)
    ▼
chat_conversation / growth_record / daily_feedback / insight (AI Service)

    │ user_id (chat_conversation만)
    ▼
Auth Service

    │ cultivation_id (chat_conversation/growth_record/daily_feedback/insight 공통)
    │ cultivation_photo_id (growth_record만)
    ▼
Cultivation Service
```

AI Service는 다른 서비스와 데이터베이스를 공유하지 않습니다. `insight`는 이 DB에만
존재하며 별도의 검색 사본을 두지 않습니다 — 검색 조건(버섯 종류 정확히 일치 + 온도
범위)이 이 테이블의 인덱스(`idx_insight_mushroom_temp`)로 충분히 처리됩니다.

---

# 고려 사항

- **`chat_conversation.channel_type`을 참조 테이블 FK 대신 VARCHAR CHECK enum으로
  유지**: 외부 ERD는 `channel_type_id`로 Notification DB의 채널 타입 참조 테이블을
  가리키지만, 이 프로젝트는 database-per-service 원칙상 다른 서비스 DB의 테이블을
  FK로 걸 수 없습니다. 그렇다고 AI DB에 자체 `channel_type` 참조 테이블을 또 두는
  것도 값이 3개(`APP`/`TELEGRAM`/`DISCORD`)뿐인 것에 비해 과합니다. 따라서 기존과
  같이 VARCHAR CHECK enum을 유지하며, 값의 의미만 Notification DB의 `channel_type`
  값과 맞추고 물리적 참조는 두지 않습니다.
- **`chat_conversation.user_id`를 NOT NULL이 아닌 NULL 허용으로 유지**: 외부 ERD는
  `user_id`를 NOT NULL로 정의합니다. `notification_endpoint`가 사용자 단위로
  재설계되면서 TELEGRAM/DISCORD 발신자도 이제 `user_id`를 특정할 수 있게 되어(이전
  설계에서는 재배 단위 채널이라 특정이 불가능했던 한계였음), 실질적으로는 거의 항상
  값이 채워집니다. 다만 웹훅 처리 중 Notification Service 호출 실패 등으로 사용자
  특정에 실패한 극히 예외적인 경로에 대비해 컬럼 자체는 NULL 허용으로 남겨둡니다 —
  스키마를 NOT NULL로 강제했다가 예외 상황에서 저장 자체가 막히는 것보다, 느슨하게
  두고 애플리케이션이 정상 경로에서는 항상 채우도록 하는 편이 안전하다고 판단했습니다.
- **`growth_record.cultivation_photo_id`를 스냅샷이 아닌 소프트 참조 ID만 저장**:
  기존에는 `object_key`/`storage_type`을 스냅샷으로 복사해 저장했습니다(AI DB가
  Cultivation DB의 `cultivation_photo`를 FK로 참조할 수 없다는 이유는 여전히 유효). 이번
  재설계에서는 `cultivation_photo_id`만 저장하도록 단순화했습니다 — 분석 결과
  조회 시 사진 자체가 필요한 경우가 드물고, 필요하면 Cultivation Service에 ID로
  별도 조회하면 되기 때문입니다. 다만 이 경우 사진 원본을 함께 보여주는 화면에서는
  Cultivation Service 호출이 추가로 필요하다는 점을 감안해야 합니다.
- **`growth_record.analysis_data`(JSONB)의 인덱스**: 현재 조회 패턴은 `cultivation_id`
  + `analyzed_at` 정렬(최근 추이 조회)이 핵심이라 기존 `idx_growth_record_cultivation`
  인덱스를 그대로 유지합니다. `growth_stage` 등 JSONB 내부 필드로 자주 필터링하는
  요구가 생기면 GIN 인덱스 또는 표현식 인덱스 추가를 검토합니다(현재는 실제로
  설계하지 않음).
- **`daily_feedback`의 환경 통계 컬럼은 외부 ERD 대비 의도적으로 확장된
  부분입니다**: 외부 ERD의 `daily_feedback`에는 `avg_temperature` 등 환경 통계
  컬럼이 없지만, 이 프로젝트는 "일일 리포트에 환경 통계 통합"이라는 이미 확정된
  기능이 있어 그대로 유지합니다.
- **`insight.cultivation_id` UNIQUE는 기존 `harvest_id` UNIQUE와 기능적으로
  동일합니다**: `harvest`가 `cultivation`과 1:1이므로, 중복 적재 방지 효과는
  그대로 유지되면서 참조 컬럼만 `cultivation_id`로 바뀌었습니다. `HarvestCompletedEvent`
  구독 처리 시 이벤트 payload의 `harvestId`는 그대로 두되, 저장/dedup 키로는
  `cultivation_id`를 사용합니다.
- `ai:{hash}`/`ai:{cultivationId}:analysis` Redis 캐시(성능 최적화용)와 이 DB(이력
  조회용)는 역할이 달라 함께 유지합니다.
- `chat_message`/`growth_record`/`daily_feedback`의 내용은 매번 새 행으로 쌓이며
  기존 행을 수정하지 않으므로 `updated_at`을 두지 않았습니다. `chat_conversation`은
  예외적으로 새 메시지가 추가될 때마다 `updated_at`이 갱신되는 대화방 메타데이터라
  `updated_at`을 둡니다.
- `daily_feedback`은 사진을 안 찍은 날도 반드시 한 행이 생성됩니다(`has_growth_data =
  false`) — "피드백 없음"과 "비교 데이터 없음"을 구분하기 위함입니다.
- `insight`와 `daily_feedback`은 같은 DB에 있어, 인사이트 후보 상세 조회 시 선택한
  `insight.cultivation_id`로 `daily_feedback`을 바로 조회할 수 있습니다(Cultivation
  Service 등 다른 서비스 호출 불필요).
- 데이터가 무한히 쌓이는 이력성 테이블들이라, 운영 단계에서는 오래된 데이터에 대한 보관
  주기 정책이 필요할 수 있습니다(추후 개발 예정).
