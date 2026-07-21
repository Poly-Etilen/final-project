# AI Database

## 개요

AI Database는 AI 챗봇의 대화 이력(`chat_message`), 생육 분석 이력(`growth_record`), 일일
피드백 이력(`daily_feedback`)을 관리합니다.

기존에는 AI Service가 별도 Database 없이 Redis 캐시(`ai:{hash}`, 동일 질문 재요청 시
LLM 재호출 방지용)만 사용했습니다. 이 캐시는 성능 최적화 용도일 뿐 대화 이력을 보여주기
위한 저장소가 아니라서, 사용자가 챗봇과 나눈 이전 대화를 다시 확인할 방법이 없었습니다.

> ℹ️ **변경 이력**: 챗봇 대화 이력 조회 기능이 추가되면서 AI Service가 처음으로 PostgreSQL
> DB를 갖게 되었습니다. 기존 `ai:{hash}` Redis 캐시(응답 재사용용)는 그대로 유지되며, 이번에
> 추가된 `chat_message`(대화 이력 원본)와 역할이 겹치지 않습니다. (자세한 내용은
> [ai.md](../01_Domain/ai.md), [ai-api.md](../02_API/ai-api.md) 참고)

> ℹ️ **변경 이력**: `growth_record`, `daily_feedback` 테이블이 추가되었습니다. "일일 피드백"
> 기능(재배 환경을 사용자가 수정했을 때, 그 이후 생육이 실제로 어떻게 달라졌는지 매일
> 알려주는 기능)을 만들려면 생육 분석 결과의 이력이 필요한데, 기존 `ai:{cultivationId}:analysis`
> Redis 캐시는 TTL 6시간이라 하루만 지나도 사라져 "어제와 오늘을 비교"하는 것이 불가능했습니다.
> `growth_record`는 이 Redis 캐시와 별개로 모든 분석 결과를 영구 보관하는 이력 테이블이고,
> `daily_feedback`은 Daily Scheduler가 매일 생성하는 피드백 결과 자체를 저장합니다. (자세한
> 내용은 [ai.md](../01_Domain/ai.md), [daily-feedback.md](../04_sequence/daily-feedback.md),
> [cultivation-db.md](./cultivation-db.md)의 `environment_setting` 참고)

---

# ERD

```
chat_message
──────────────────────────────────────────────
PK  id
    user_id
    cultivation_id
    message
    answer
    created_at

growth_record
──────────────────────────────────────────────
PK  id
    cultivation_id
    image_url
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
    created_at
```

세 테이블 모두 `cultivation_id`로만 연결되며, 서로 간 FK는 없습니다(같은 DB 안에서도 소프트
참조).

---

# Table

## chat_message

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | X | 질문한 사용자 (Auth Service의 userId, 소프트 참조) |
| cultivation_id | BIGINT | X | 질문 대상 재배 (Cultivation Service의 cultivationId, 소프트 참조) |
| message | VARCHAR(1000) | X | 사용자 질문 |
| answer | TEXT | X | LLM 응답 |
| created_at | TIMESTAMP | X | 질의 시각 |

`user_id`/`cultivation_id`는 각각 Auth Service, Cultivation Service의 데이터를 가리키지만
DB 레벨 FK가 아닌 소프트 참조입니다. `user_id`는 요청 시 Bearer JWT에서, `cultivation_id`는
`POST /ai/chat` 요청 본문에서 그대로 가져와 저장합니다.

---

## growth_record

사용자가 생육 사진을 업로드할 때마다 Vision 분석 결과를 한 행씩 쌓는 이력 테이블입니다.
`ai:{cultivationId}:analysis` Redis 캐시(TTL 6시간, "최근 분석 결과 재조회"용)와 달리
만료되지 않고 계속 보관되며, 일일 피드백이 여러 날짜의 생육 추이를 비교하는 데 사용합니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 분석 대상 재배 (Cultivation Service의 cultivationId, 소프트 참조) |
| image_url | VARCHAR(500) | X | 분석에 사용한 사진 URL (Cultivation Service가 MinIO에 저장한 값을 그대로 기록) |
| growth_score | INT | X | 생육 점수 (균사 40% + 갓크기 30% + 색상 15% + 병충해 15%) |
| mycelium_growth_rate | INT | X | 균사 성장률(%) |
| cap_size | VARCHAR(30) | O | 갓 크기 (예: "중(3.2cm)") |
| color_status | VARCHAR(20) | X | 색상 상태 (정상/변색/갈변 등) |
| disease_status | VARCHAR(20) | X | 병충해 상태 (정상/의심/감염) |
| growth_stage | VARCHAR(30) | X | 성장 단계 (균사기/자실체 형성기/성장기/수확 적기) |
| expected_harvest_date | DATE | O | 예상 수확일 |
| analyzed_at | TIMESTAMP | X | 분석 시각 |

`growth_score`~`expected_harvest_date`는 `POST /ai/analysis` 응답과 동일한 값을 그대로
저장합니다. 별도로 재계산하지 않습니다.

---

## daily_feedback

Daily Scheduler가 매일 재배별로 생성하는 일일 피드백 결과입니다.

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| cultivation_id | BIGINT | X | 대상 재배 (소프트 참조) |
| feedback_date | DATE | X | 이 피드백이 다루는 날짜 |
| has_growth_data | BOOLEAN | X | 그날 `growth_record`가 있었는지 여부 |
| content | TEXT | X | 피드백 본문 (LLM 생성 문장, 또는 사진이 없을 때의 고정 안내 문구) |
| created_at | TIMESTAMP | X | 생성 시각 |

`has_growth_data`가 `false`이면 `content`에는 "전날 사진이 없어 피드백을 남길 수 없습니다"
같은 고정 문구가 들어갑니다. 사진이 있었던 날은 `content`에 환경 변경 이력과 생육 추이를
비교한 LLM 생성 문장이 들어갑니다. `(cultivation_id, feedback_date)`는 하루에 한 번만
생성되므로 UNIQUE 제약을 둡니다.

---

# DDL

```sql
CREATE TABLE chat_message (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT NOT NULL,

    cultivation_id BIGINT NOT NULL,

    message VARCHAR(1000) NOT NULL,

    answer TEXT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

```sql
CREATE TABLE growth_record (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    image_url VARCHAR(500) NOT NULL,

    growth_score INT NOT NULL,

    mycelium_growth_rate INT NOT NULL,

    cap_size VARCHAR(30),

    color_status VARCHAR(20) NOT NULL,

    disease_status VARCHAR(20) NOT NULL,

    growth_stage VARCHAR(30) NOT NULL,

    expected_harvest_date DATE,

    analyzed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

```sql
CREATE TABLE daily_feedback (
    id BIGSERIAL PRIMARY KEY,

    cultivation_id BIGINT NOT NULL,

    feedback_date DATE NOT NULL,

    has_growth_data BOOLEAN NOT NULL DEFAULT FALSE,

    content TEXT NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uk_daily_feedback_cultivation_date
        UNIQUE (cultivation_id, feedback_date)
);
```

세 테이블 모두 `cultivation_id`가 다른 DB(Cultivation Service)를 가리키는 소프트 참조라
DB 레벨 FK는 걸지 않습니다. (`chat_message`와 동일한 기존 컨벤션)

---

# Index

```sql
CREATE INDEX idx_chat_message_cultivation
ON chat_message(cultivation_id, created_at DESC);
```

대화 이력 조회(`GET /ai/chat/history`)가 "특정 재배의, 최신순" 조건으로 조회되기 때문에
이 복합 인덱스 하나로 충분합니다.

```sql
CREATE INDEX idx_growth_record_cultivation
ON growth_record(cultivation_id, analyzed_at DESC);
```

일일 피드백이 "특정 재배의 특정 날짜에 분석 기록이 있는지", "최근 며칠간의 추이"를 조회할 때
사용합니다.

```sql
CREATE INDEX idx_daily_feedback_cultivation
ON daily_feedback(cultivation_id, feedback_date DESC);
```

`GET /ai/feedback/daily`(일일 피드백 목록 조회)가 "특정 재배의, 최신순" 조건으로 조회되기
때문입니다. `uk_daily_feedback_cultivation_date` UNIQUE 제약이 이미 `(cultivation_id,
feedback_date)` 조합에 대한 인덱스 역할도 겸합니다.

---

# 관계

```
chat_message / growth_record / daily_feedback (AI Service)

    │ user_id (chat_message만)
    ▼
Auth Service (userId로 질문자 식별)

    │ cultivation_id (세 테이블 공통)
    ▼
Cultivation Service (cultivationId로 재배 식별)
```

AI Service는 다른 서비스와 데이터베이스를 공유하지 않으며, `userId`/`cultivationId`를 통해서만
간접적으로 참조됩니다.

---

# 다른 서비스와의 연관

## Cultivation Service

`cultivation_id`로 어떤 재배에 대한 질문/분석/피드백인지 식별합니다. 재배 소유자 검증(요청자가
실제로 그 재배의 소유자인지)은 AI Service가 아니라 API Gateway/JWT 기반 인가와 별개로,
필요하다면 Cultivation Service 조회를 통해 확인해야 합니다(현재 문서에는 이 검증 흐름이
별도로 정의되어 있지 않으며, 추후 정리가 필요합니다).

일일 피드백을 생성할 때는 `cultivation_id` 기준으로 Cultivation Service의
`environment_setting`(최근 변경 이력)도 함께 조회합니다. (자세한 내용은
[daily-feedback.md](../04_sequence/daily-feedback.md) 참고)

---

# 고려 사항

- `ai:{hash}` Redis 캐시(질문 해시 기준, TTL 24시간)는 "동일 질문 반복 시 LLM 재호출 방지"용
  성능 캐시이고, `chat_message`는 "대화 이력 조회"용 영구 저장소입니다. 둘은 목적이 달라
  함께 유지됩니다.
- `ai:{cultivationId}:analysis` Redis 캐시(TTL 6시간)는 "방금 분석한 결과를 재요청 없이 즉시
  재조회"용 성능 캐시이고, `growth_record`는 "생육 추이 이력"용 영구 저장소입니다. 마찬가지로
  역할이 달라 함께 유지됩니다.
- `message`/`answer`, `growth_record`의 분석 결과, `daily_feedback`의 피드백 내용은 모두 매번
  새 행으로 쌓입니다. 기존 행을 수정하지 않으므로 세 테이블 다 `updated_at`은 두지 않았습니다.
- `daily_feedback`은 하루 사진을 안 찍었어도 반드시 한 행이 생성됩니다(`has_growth_data =
  false`, 고정 안내 문구). 건너뛰지 않고 항상 생성하는 이유는 사용자가 "오늘은 피드백이 아예
  없다"와 "오늘은 사진이 없어서 비교를 못 했다"를 구분할 수 있게 하기 위함입니다.
- 데이터가 무한히 쌓이는 이력성 테이블들이라, 운영 단계에서는 오래된 데이터에 대한 보관 주기
  정책이 필요할 수 있습니다(추후 개발 예정).
