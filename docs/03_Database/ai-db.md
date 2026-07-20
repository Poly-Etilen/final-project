# AI Database

## 개요

AI Database는 AI 챗봇의 대화 이력을 `chat_message` 테이블 하나로 관리합니다.

기존에는 AI Service가 별도 Database 없이 Redis 캐시(`ai:{hash}`, 동일 질문 재요청 시
LLM 재호출 방지용)만 사용했습니다. 이 캐시는 성능 최적화 용도일 뿐 대화 이력을 보여주기
위한 저장소가 아니라서, 사용자가 챗봇과 나눈 이전 대화를 다시 확인할 방법이 없었습니다.

> ℹ️ **변경 이력**: 챗봇 대화 이력 조회 기능이 추가되면서 AI Service가 처음으로 PostgreSQL
> DB를 갖게 되었습니다. 기존 `ai:{hash}` Redis 캐시(응답 재사용용)는 그대로 유지되며, 이번에
> 추가된 `chat_message`(대화 이력 원본)와 역할이 겹치지 않습니다. (자세한 내용은
> [ai.md](../01_Domain/ai.md), [ai-api.md](../02_API/ai-api.md) 참고)

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
```

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

---

# Index

```sql
CREATE INDEX idx_chat_message_cultivation
ON chat_message(cultivation_id, created_at DESC);
```

대화 이력 조회(`GET /ai/chat/history`)가 "특정 재배의, 최신순" 조건으로 조회되기 때문에
이 복합 인덱스 하나로 충분합니다.

---

# 관계

```
chat_message (AI Service)

    │ user_id
    ▼
Auth Service (userId로 질문자 식별)

    │ cultivation_id
    ▼
Cultivation Service (cultivationId로 재배 식별)
```

AI Service는 다른 서비스와 데이터베이스를 공유하지 않으며, `userId`/`cultivationId`를 통해서만
간접적으로 참조됩니다.

---

# 다른 서비스와의 연관

## Cultivation Service

`cultivation_id`로 어떤 재배에 대한 질문인지 식별합니다. 재배 소유자 검증(요청자가 실제로
그 재배의 소유자인지)은 AI Service가 아니라 API Gateway/JWT 기반 인가와 별개로,
필요하다면 Cultivation Service 조회를 통해 확인해야 합니다(현재 문서에는 이 검증 흐름이
별도로 정의되어 있지 않으며, 추후 정리가 필요합니다).

---

# 고려 사항

- `ai:{hash}` Redis 캐시(질문 해시 기준, TTL 24시간)는 "동일 질문 반복 시 LLM 재호출 방지"용
  성능 캐시이고, `chat_message`는 "대화 이력 조회"용 영구 저장소입니다. 둘은 목적이 달라
  함께 유지됩니다.
- `message`/`answer`는 매 질의응답마다 새 행으로 쌓입니다. 기존 행을 수정하지 않으므로
  `updated_at`은 두지 않았습니다.
- 데이터가 무한히 쌓이는 이력성 테이블이라, 운영 단계에서는 오래된 대화에 대한 보관 주기
  정책이 필요할 수 있습니다(추후 개발 예정).
