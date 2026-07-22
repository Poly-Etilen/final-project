# AI 챗봇 시퀀스

## 개요

사용자가 자연어로 재배 관련 질문을 하면 AI가 현재 센서 데이터와 유사 재배 사례를
참고해 답변을 생성하는 과정입니다. 동일한 질문에 대한 반복 호출을 줄이기 위해 Redis에
응답을 캐싱합니다.

챗봇은 웹/앱(`APP`) 채널뿐 아니라 Telegram/Discord 봇으로도 사용할 수 있습니다. 아래
"1~9"는 APP 채널 기준이며, Telegram/Discord 채널의 흐름은 맨 아래 "채널별 챗봇" 섹션을
참고하세요. 매 발화(사용자 질문, 챗봇 응답)는 `chat_log` 테이블(AI DB)에 한 행씩 영구
저장되며, `GET /ai/chat/history`로 이전 대화를 조회할 수 있습니다.

---

# Sequence

```text
Client
↓
API Gateway
↓
AI Service
↓
Redis 조회
↓
Cache Miss
↓
Sensor Service — 현재 환경 / 통계 조회
↓
(재배가 지정된 경우) Cultivation Service — 버섯 종류 조회
→ Sensor Service — mushroom_reference 정확 조회 (RAG 컨텍스트)
↓
AI Service → LLM — 답변 생성
↓
Redis 저장 + chat_log 저장
↓
Client
```

---

# 상세 과정

## 1. 챗봇 질문 요청

```http
POST /api/v1/ai/chat
```

```json
{
    "cultivationId": 3,
    "message": "왜 성장이 느린가요?"
}
```

---

## 2. Redis 캐시 조회

```
Key: ai:{hash}
```

`hash`는 `cultivationId`와 질문 내용을 조합해 생성합니다. 존재하면 즉시 반환합니다.

---

## 3. Cache Miss

Redis에 없으면 Sensor Service를 OpenFeign으로 호출합니다.

---

## 4. 센서 데이터 조회

Sensor Service가 현재 환경(Redis)과 최근 통계(InfluxDB)를 조회합니다.

조회 항목: 현재 온도/습도/CO₂/조도, 최근 7일 평균값, 목표 환경 대비 유지율

---

## 5. 버섯 참조 데이터 조회 (재배가 지정된 경우)

`cultivationId`가 있으면 AI Service가 Cultivation Service에서 버섯 종류
(`mushroomType`)를 조회하고, 이어서 Sensor Service에서 `mushroomType`으로 정확히
일치하는 `mushroom_reference` 한 건을 조회해 특성/효능/재배 가이드 텍스트를 LLM
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

## 8. Redis 저장 + chat_log 영구 저장

```
Key: ai:{hash}
TTL: 24시간
```

Redis 캐시 저장과 별개로, `chat_log`에 사용자 질문(`senderRole=USER`)과 챗봇 응답
(`senderRole=BOT`) 두 행이 `channelType=APP`으로 저장됩니다. Redis는 캐시 미스 시에만
채워지지만, `chat_log`는 캐시 히트 여부와 무관하게 매 요청마다 저장됩니다.

---

## 9. 응답 반환

Client에게 챗봇 답변을 반환합니다.

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
Notification Service OpenFeign 호출 (발신자 Chat ID로 등록된 notification_endpoint 조회)
├── 매칭 없음 → "먼저 이 채널을 재배에 등록해주세요" 고정 안내 문구 응답 (LLM 미호출)
└── 매칭 있음 → cultivationId 확인, chat_log 저장 (senderRole=USER, cultivationId=조회된 값, userId=NULL)
↓
Redis 조회 (ai:{hash}) → Cache Miss 시 Sensor Service 조회 (현재 환경/통계) + LLM 호출
↓
chat_log 저장 (senderRole=BOT, 같은 cultivationId)
↓
Telegram Bot API / Discord Webhook으로 응답 전송
```

`notification_endpoint`가 재배 단위로 등록되므로, Telegram/Discord 챗봇도 항상 특정
`cultivationId` 맥락에서 답변합니다(Sensor Service 조회 등 APP 채널과 동일한 흐름
수행). 다만 이 경로로는 개별 사용자를 특정할 수 없어 `chat_log.user_id`는 NULL로
저장됩니다. Notification Service를 호출하는 이유는 이미 알림 채널 등록용으로 저장된
`notification_endpoint`를 챗봇 발신자 식별에도 재사용하기 위해서이며, 별도의
채널-재배 매핑을 새로 두지 않았습니다.

---

# 사용 Database

## PostgreSQL

```
chat_log (AI DB) — 매 발화 영구 저장
```

## Redis

AI 응답 캐시

## InfluxDB

센서 통계 조회 (Sensor Service 경유)

---

# OpenFeign

```
AI Service → Sensor Service (현재 환경/통계 조회, mushroom_reference 조회)
AI Service → Cultivation Service (재배가 지정된 경우 버섯 종류 조회)
AI Service → Notification Service (Telegram/Discord 웹훅 수신 시, 발신자 → cultivationId 조회)
```

---

# RabbitMQ

사용하지 않습니다. 챗봇은 사용자 요청 기반의 동기 처리입니다.

---

# 예외 상황

- Redis 장애 / Sensor Service 호출 실패 / Cultivation Service 호출 실패
- LLM 응답 실패
- `chat_log` 저장 실패 (저장에 실패해도 챗봇 응답 자체는 반환)
- Telegram/Discord 웹훅으로 메시지가 왔지만 등록된 `notification_endpoint`가 없는 경우
- Notification Service 호출 실패 (발신자 → cultivationId 조회 실패)

---

# 고려 사항

- 챗봇 질문은 매번 다를 수 있어 캐시 히트율이 낮을 수 있습니다.
- 버섯 참조 데이터 조회는 재배가 지정된 질문에서만 수행해 불필요한 호출을 줄입니다.
  버섯 종류가 5종 고정이라 벡터 검색이 아닌 정확한 값 매칭으로 충분합니다.
- 매 발화는 `chat_log`에 영구 저장되며, `GET /ai/chat/history`로 채널별/재배별 대화
  이력을 조회할 수 있습니다. 다만 LLM이 답변을 생성할 때 과거 대화를 컨텍스트로
  참고하지는 않습니다 — 매 질문은 여전히 독립적으로 처리됩니다.
- InfluxDB 원본 데이터가 아닌 집계된 통계만 LLM에 전달해 토큰 사용량을 줄입니다.
- Telegram/Discord 채널은 재배에 알림 채널을 등록(`notification_endpoint`)하는 절차가
  선행되어야 사용할 수 있습니다. 미등록 채널은 안내 문구만 받고 LLM은 호출되지
  않습니다.
- `notification_endpoint`가 재배 단위이므로, 같은 재배에 접근하는 여러 사용자가 같은
  Telegram/Discord 채널을 공유해서 사용합니다 — "누가 보냈는지"는 구분하지 않는 것이
  의도된 설계입니다.
