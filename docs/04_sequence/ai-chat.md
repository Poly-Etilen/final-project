# AI 챗봇 시퀀스

## 개요

사용자가 자연어로 재배 관련 질문을 하면 AI가 현재 센서 데이터와 유사 재배 사례를
참고해 답변을 생성하는 과정입니다. 동일한 질문에 대한 반복 호출을 줄이기 위해 Redis에
응답을 캐싱합니다.

챗봇은 용도가 다른 두 채널로 나뉩니다. 웹(`APP`) 채널은 WebSocket으로 연결하는 채팅방이며
`/`로 시작하는 명령어(조리법 등 공공데이터 조회, `/인사이트`)를 처리합니다. 아래 "1~9"는
웹(`APP`) 채널 기준이며, Telegram/Discord 채널의 흐름은 맨 아래 "채널별 챗봇" 섹션을
참고하세요 — 이 채널은 명령어나 멀티유저 채팅방 없이 알림 수신 + 자연어 질의응답만
제공합니다. 대화는 대화방(`chat_conversation`)과 발화(`chat_message`) 두 테이블(AI DB)로
나눠 저장됩니다. 같은 `(channel_type, external_conversation_id)`로 다시 들어오면 새
대화방을 만들지 않고 기존 `chat_conversation` 행을 재사용하며, 매 발화(사용자 질문,
챗봇 응답)는 그 대화방 아래 `chat_message` 행으로 `sequence_number`가 증가하며 한 행씩
영구 저장됩니다. `GET /ai/chat/history`로 이전 대화를 조회할 수 있습니다.

---

# Sequence

```text
Client
↓
API Gateway (WebSocket Upgrade)
↓
AI Service — WebSocket 연결 수립 (JWT 인증)
↓
메시지 수신 → '/' 명령어 여부 판별
├── 명령어 아님 → 일반 질문 처리 (Redis 조회 → Cache Miss 시 Cultivation Service 조회 → LLM)
├── '/인사이트' → 인사이트 후보 조회(최대 5개) → 선택 시 후보 상세 조회 (insight.md 참고)
└── 그 외 '/' 명령어 → 공공데이터 API 조회
↓
Redis 저장(일반 질문만) + chat_conversation 조회/생성 + chat_message 저장
↓
Client (WebSocket으로 응답 전송)
```

---

# 상세 과정

## 1. 웹소켓 연결 및 챗봇 질문 전송

```http
WS /api/v1/ai/chat
```

Client가 WebSocket 연결을 수립하면(JWT로 인증), 이후 연결이 유지되는 동안 채팅방처럼
여러 메시지를 주고받습니다. 연결 시 `cultivationId`를 지정할 수 있습니다(선택).

```json
{
    "cultivationId": 3,
    "message": "왜 성장이 느린가요?"
}
```

---

## 1-1. '/' 명령어 판별

메시지가 `/`로 시작하면 일반 질문 대신 명령어로 처리합니다.

- `/인사이트` → 인사이트 후보 조회를 호출해 최대 5개 리스트를 보여주고, 사용자가 그중
  하나를 선택하면 후보 상세(날짜별 환경/피드백)를 이어서 보여줍니다. 자세한 흐름은
  [insight.md](./insight.md) 참고.
- 그 외 명령어(예: `/조리법 표고버섯`) → 조리법 등 공공데이터 API를 조회해 결과를
  정리해 응답합니다.

명령어 처리 결과도 `chat_conversation`/`chat_message`에는 저장되지만, Redis 응답
캐시(`ai:{hash}`) 대상은 아닙니다(질문 다양성이 낮고 원본 데이터가 자주 바뀔 수 있어
매번 새로 조회합니다).

명령어가 아니면 2번부터 이어지는 기존 일반 질문 흐름을 그대로 따릅니다.

---

## 2. Redis 캐시 조회

```
Key: ai:{hash}
```

`hash`는 `cultivationId`와 질문 내용을 조합해 생성합니다. 존재하면 즉시 반환합니다.

---

## 3. Cache Miss

Redis에 없으면 Cultivation Service를 OpenFeign으로 호출합니다.

---

## 4. 센서 데이터 조회

Cultivation Service가 현재 환경(Redis)과 최근 통계(InfluxDB)를 조회합니다.

조회 항목: 현재 온도/습도/CO₂/조도, 최근 7일 평균값, 목표 환경 대비 유지율

---

## 5. 버섯 참조 데이터 조회 (재배가 지정된 경우)

`cultivationId`가 있으면 AI Service가 Cultivation Service에서 버섯 참조 ID
(`mushroomId`)를 조회하고, 이어서 같은 서비스의 `mushroom_reference`에서
`mushroomId`로 정확히 일치하는 한 건을 조회해 특성/효능/재배 가이드 텍스트를 LLM
컨텍스트로 사용합니다. 버섯 종류가 5종 고정이라 항상 정확히 한 건만 조회하면
되므로, 별도의 유사도 검색 없이 직접 조회로 충분합니다. `cultivationId`가 없는
일반 질문(예: 단순 인사)은 이 단계를 건너뜁니다.

---

## 6. LLM 질의

입력: 사용자 질문, 버섯 종류, 현재 센서 데이터, 목표 환경, 버섯 참조 텍스트(선택)

---

## 7. 답변 생성

```
현재 습도가 목표보다 5% 낮은 상태가 지속되고 있어 생육 속도가 느려졌을 수 있습니다.
가습기 가동 주기를 조금 더 짧게 설정하는 것을 권장합니다.
```

---

## 8. Redis 저장 + chat_conversation/chat_message 영구 저장

```
Key: ai:{hash}
TTL: 24시간
```

Redis 캐시 저장과 별개로, `channel_type=APP` + `external_conversation_id`(WebSocket
세션 식별자)로 기존 `chat_conversation`을 조회하고 없으면 새로 만듭니다. 이어서 그
대화방 아래 사용자 질문(`role=USER`)과 챗봇 응답(`role=BOT`) 두 `chat_message` 행을
`sequence_number`를 하나씩 증가시키며 저장하고, `chat_conversation.updated_at`도
함께 갱신합니다. Redis는 일반 질문의 캐시 미스 시에만 채워지지만, `chat_message`는
명령어 처리 결과를 포함해 캐시 히트 여부와 무관하게 매 요청마다 저장됩니다.

```sql
INSERT INTO chat_message (chat_conversation_id, role, content, sequence_number)
VALUES
    (101, 'USER', '왜 성장이 느린가요?', 5),
    (101, 'BOT', '현재 습도가 목표보다 5% 낮은 상태가 지속되고 있어...', 6);
```

---

## 9. 응답 반환

WebSocket 연결을 통해 Client에게 챗봇 답변을 전송합니다. 연결은 유지되며, Client는 이어서
다음 메시지를 계속 보낼 수 있습니다.

---

# 채널별 챗봇 (Telegram / Discord)

Telegram/Discord에서는 사용자가 봇에게 직접 메시지를 보내면, Telegram Bot API/Discord가
AI Service의 웹훅으로 그 메시지를 전달합니다(Client가 API Gateway를 거쳐 직접 호출하는
APP 채널과 진입 경로가 다릅니다).

```text
사용자 (Telegram/Discord 앱에서 봇에게 메시지 전송)
↓
Telegram Bot API / Discord
↓
AI Service (POST /ai/chat/telegram/webhook 또는 POST /ai/chat/discord/webhook)
↓
Notification Service OpenFeign 호출 (발신자 Chat ID로 등록된 notification_endpoint 조회
→ 해당 endpoint의 활성 notification_subscription 목록에서 target_id 확인)
├── endpoint 매칭 없음 → "먼저 이 채널을 등록해주세요" 고정 안내 문구 응답 (LLM 미호출)
├── 구독된 재배(target_id)가 정확히 1개 → cultivationId 확정, chat_conversation 조회/생성
│   (channel_type=TELEGRAM/DISCORD, external_conversation_id=Chat ID,
│   user_id=endpoint.user_id, cultivation_id=확정된 값) 후 chat_message 저장 (role=USER)
└── 구독된 재배가 0개 또는 2개 이상 → 어떤 재배에 대한 질문인지 특정할 수 없어 안내
    문구 응답 (LLM 미호출)
↓
Redis 조회 (ai:{hash}) → Cache Miss 시 Cultivation Service 조회 (현재 환경/통계) + LLM 호출
↓
chat_message 저장 (role=BOT, 같은 chat_conversation_id)
↓
Telegram Bot API / Discord Webhook으로 응답 전송
```

이 채널에는 웹 채널의 `/` 명령어나 채팅방(여러 사용자가 함께 보는 대화) 기능이 없습니다
— Notification Service의 알림 수신과 자연어 질의응답만 제공하며, 관련 확장 여부는 아직
논의 중입니다.

`notification_endpoint`가 이제 사용자 단위로 등록되므로(재배 단위였던 이전 설계와
다름), 발신자 Chat ID로 `notification_endpoint`를 조회하면 그 채널을 등록한
`user_id`를 바로 알 수 있습니다 — 이전처럼 `chat_conversation.user_id`를 NULL로 둘
필요가 없어졌습니다. 다만 "어떤 재배에 대한 질문인가"는 그 endpoint에 연결된
`notification_subscription`(target_id) 목록으로 판단해야 하며, 같은 endpoint가 여러
재배를 구독 중이면 특정할 수 없습니다(고정 안내 문구로 응답). Notification Service를
호출하는 이유는 이미 채널 등록/구독용으로 저장된 `notification_endpoint`/
`notification_subscription`을 챗봇 발신자·대상 재배 식별에도 재사용하기 위해서이며,
별도의 채널-재배 매핑을 새로 두지 않았습니다. 같은 Chat ID(`external_conversation_id`)로
다시 메시지가 오면 새 대화방을 만들지 않고 기존 `chat_conversation`을 계속
재사용합니다.

---

# 사용 Database

## PostgreSQL

```
chat_conversation (AI DB) — 대화방 단위, 채널+외부 식별자로 조회/생성
chat_message (AI DB) — 매 발화 영구 저장 (chat_conversation_id FK)
```

## Redis

AI 응답 캐시

## InfluxDB

센서 통계 조회 (Cultivation Service 경유)

---

# OpenFeign

```
AI Service → Cultivation Service (현재 환경/통계 조회, 버섯 종류 조회, mushroom_reference 조회)
AI Service → Notification Service (Telegram/Discord 웹훅 수신 시, 발신자 → cultivationId 조회)
```

---

# 외부 연동

```
AI Service → 공공데이터 API (웹 채널 '/' 명령어 처리 시, 예: 조리법 조회)
```

---

# RabbitMQ

사용하지 않습니다. 챗봇은 사용자 요청 기반의 동기 처리입니다.

---

# 예외 상황

- Redis 장애 / Cultivation Service 호출 실패
- LLM 응답 실패
- `chat_conversation`/`chat_message` 저장 실패 (저장에 실패해도 챗봇 응답 자체는 반환)
- Telegram/Discord 웹훅으로 메시지가 왔지만 등록된 `notification_endpoint`가 없는 경우
- Notification Service 호출 실패 (발신자 → cultivationId 조회 실패)
- 웹 채널 WebSocket 연결 끊김 / 재연결
- '/' 명령어 처리 중 공공데이터 API 호출 실패

---

# 고려 사항

- 챗봇 질문은 매번 다를 수 있어 캐시 히트율이 낮을 수 있습니다.
- 버섯 참조 데이터 조회는 재배가 지정된 질문에서만 수행해 불필요한 호출을 줄입니다.
  버섯 종류가 5종 고정이라 벡터 검색이 아닌 정확한 값 매칭으로 충분합니다.
- 매 발화는 `chat_message`에 영구 저장되며, `GET /ai/chat/history`로 채널별/재배별 대화
  이력을 조회할 수 있습니다. 다만 LLM이 답변을 생성할 때 과거 대화를 컨텍스트로
  참고하지는 않습니다 — 매 질문은 여전히 독립적으로 처리됩니다.
- 기존 `chat_log` 단일 테이블을 `chat_conversation`(대화방)/`chat_message`(발화)로
  분리했습니다. 같은 `(channel_type, external_conversation_id)`로 다시 들어오면
  새 대화방을 만들지 않고 기존 대화방에 발화를 계속 이어 붙입니다.
- `chat_conversation.user_id`는 외부 ERD 기준 NOT NULL이지만, 이 프로젝트는
  Telegram/Discord처럼 발신자 개인을 특정할 수 없는 경로가 실재해 NULL을
  허용합니다 — 자세한 판단 근거는 [ai-db.md](../03_Database/ai-db.md)의
  고려 사항 참고.
- InfluxDB 원본 데이터가 아닌 집계된 통계만 LLM에 전달해 토큰 사용량을 줄입니다.
- Telegram/Discord 채널은 사용자가 채널을 등록(`notification_endpoint`)하고 특정
  재배를 구독(`notification_subscription`)하는 절차가 선행되어야 사용할 수 있습니다.
  미등록/미구독 상태거나 구독한 재배가 여러 개면 안내 문구만 받고 LLM은 호출되지
  않습니다.
- `notification_endpoint`가 사용자 단위이므로, 같은 재배에 접근하는 여러 멤버가 각자
  자신의 Telegram/Discord 채널로 독립적으로 챗봇을 사용할 수 있습니다(이전 설계처럼
  채널을 공유하지 않음) — "누가 보냈는지"도 이제 `endpoint.user_id`로 구분됩니다.
- 웹 채널의 `/` 명령어는 일반 질문과 달리 Redis 응답 캐시를 사용하지 않습니다 — 명령어별
  결과(공공데이터, 인사이트)가 자주 바뀔 수 있어 매번 새로 조회합니다.
- Telegram/Discord 채널에 `/` 명령어나 멀티유저 채팅방을 확장할지는 아직 결정되지
  않았습니다. 이 문서는 현재 결정된 범위(알림 수신 + 자연어 질의응답)만 다룹니다.
