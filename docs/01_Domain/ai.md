# AI Service

## 역할

AI Service는 LLM과 Vision 모델을 활용해 지능형 기능을 제공하는 서비스입니다. 생육 사진
분석, 자연어 챗봇(웹/앱/Telegram/Discord), 주간 리포트, 일일 피드백, "인사이트"(타인의
유사 재배 사례 기반 피드백), 버섯 가이드(효능/주의사항)를 담당합니다.

---

# 책임

- 생육 사진 Vision 분석
- AI 챗봇 (APP/Telegram/Discord 다채널, 대화 이력 저장/조회)
- 주간 리포트 생성 (Weekly Scheduler push 기반)
- 일일 피드백 생성 (Daily Scheduler)
- 인사이트 사례 적재(수확 완료 시점) 및 조회
- 버섯 가이드(효능/주의사항) 생성

---

# 주요 기능

## AI 생육 분석 (Vision)

사용자가 업로드한 사진을 Vision 모델로 분석해 생육 점수/균사 성장률/갓 크기/색상/병충해/
성장 단계/예상 수확일을 반환합니다. 분석 결과는 Redis(`ai:{cultivationId}:analysis`,
TTL 6시간)로 빠른 재조회를 지원하는 동시에 `growth_record`에 영구 저장해 일일 피드백의
추이 비교에 사용합니다.

---

## AI 챗봇

사용자는 자연어로 재배 관련 질문을 할 수 있습니다. 매 발화(질문/응답)는 `chat_log`에
한 행씩 저장됩니다.

- **APP**: Client가 Bearer JWT로 인증된 `POST /ai/chat`을 직접 호출합니다. `userId`는
  JWT에서, `cultivationId`는 요청 파라미터로 지정합니다(선택).
- **Telegram/Discord**: 사용자가 봇에게 메시지를 보내면 웹훅으로 전달됩니다. AI Service는
  발신자 Chat ID로 Notification Service의 `notification_endpoint`를 조회해
  `cultivationId`를 알아냅니다(알림 채널이 재배 단위로 등록되므로). 이 경로로는 개별
  사용자를 특정할 수 없어 `user_id`는 NULL로 저장되지만, 대신 항상 특정 재배 맥락(Sensor
  데이터 조회 포함)에서 답변할 수 있습니다. 매칭되는 endpoint가 없으면 "먼저 이 채널을
  재배에 등록해주세요" 안내로 응답합니다.

답변 생성 시 Sensor Service에서 현재 환경/통계를 조회하고, 필요하면 같은 호출로 해당
재배의 버섯 종류에 맞는 `mushroom_reference` 텍스트(효능/재배 가이드)를 함께 받아 LLM
컨텍스트로 사용합니다. 버섯 종류가 5종으로 고정되어 있어 `mushroomType`으로 정확히
일치하는 한 건만 조회하면 되므로, 별도의 유사도 검색 없이 직접 조회로 충분합니다.

동일 질문 반복 호출을 줄이기 위해 Redis(`ai:{hash}`, TTL 24시간)에 응답을 캐싱합니다.

---

## 주간 리포트

Sensor Service의 Weekly Scheduler가 매주 집계 데이터를 먼저 전달(push)하면, AI Service가
그 자리에서 리포트를 생성해 Redis(`report:{cultivationId}:weekly`, TTL 24시간)에 저장한 뒤
`WeeklyReportCompletedEvent`를 발행합니다. 재배 기간이 한 달을 넘지 않아 월간 리포트는
제공하지 않습니다.

---

## 일일 피드백

사용자가 환경(온도/습도/CO₂/조도)을 직접 수정했을 때, 그 수정이 생육에 실제로 도움이
됐는지 매일 알려주는 기능입니다. Daily Scheduler가 매일 재배별로 `growth_record`(생육 추이)와
Sensor Service의 `environment_setting`(환경 변경 이력)을 비교해 LLM으로 피드백을 생성하고
`daily_feedback`에 저장합니다. 사진을 찍지 않아 비교할 `growth_record`가 없으면 LLM을
호출하지 않고 고정 문구로 피드백을 생성합니다(이 경우도 `daily_feedback` 행은 생성됨).

---

## 인사이트

같은 버섯 종류 + 유사한 온도로 재배했던 타인의 사례를 바탕으로 현재 재배 상태를
피드백하는 기능입니다. 데이터 적재(수확 완료 시점)와 조회(on-demand)를 분리합니다.

- Cultivation Service가 수확을 기록하면 발행하는 `HarvestCompletedEvent`를 AI Service가
  구독해, 그 수확의 재배에 대한 환경 평균(Sensor Service 조회)과 최근 생육 점수
  (`growth_record` 자체 조회)를 묶어 AI DB의 `insight` 테이블에 한 건 저장합니다. 별도
  배치나 임계치 없이 수확이 기록될 때마다 바로 반영됩니다.
- 사용자 조회(`GET /ai/insight`)는 요청 시점에 Redis 캐시 미스 시에만 Sensor
  Service(현재 재배의 환경 평균 조회) + 자체 DB의 `insight` 테이블 검색(버섯 종류 정확히
  일치 + 온도 오차 범위, SQL 필터) + LLM 요약을 거칩니다. 사례 수가 많지 않고 검색
  조건이 정확한 값 매칭/범위 비교라서, 별도의 벡터 검색 없이 인덱스가 걸린 SQL 조회로
  충분합니다.

---

## 버섯 가이드

재배 생성 직후 버섯의 효능/재배 주의사항을 자연어로 보여줍니다. Sensor Service의
`mushroom_reference` 텍스트 컬럼을 RAG 컨텍스트로 사용해 LLM이 자연스러운 문장으로
재구성합니다. `mushroomType`(5종 고정) 기준으로 캐싱(`ai:mushroom:{mushroomType}:guide`,
TTL 7일)해 반복 호출을 피합니다.

---

# API

## 생육 분석

POST /ai/analysis

---

## 챗봇

POST /ai/chat

POST /ai/chat/telegram/webhook (내부용)

POST /ai/chat/discord/webhook (내부용)

GET /ai/chat/history

---

## 리포트/피드백/인사이트/가이드

GET /ai/report

GET /ai/feedback/daily

GET /ai/insight

POST /ai/mushroom-guide

---

# Database

AI Service는 하나의 PostgreSQL Database를 사용합니다.

## Table

- chat_log
- growth_record
- daily_feedback
- insight

자세한 내용은 [ai-db.md](../03_Database/ai-db.md) 참고.

---

# Redis

- 챗봇 응답 캐시, 생육 분석 결과 캐시, 리포트 캐시, 버섯 가이드 캐시, 인사이트 캐시

자세한 내용은 [redis.md](../03_Database/redis.md) 참고.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Cultivation Service

- 생육 사진 조회, 챗봇/인사이트 조회 시 버섯 종류 조회

### Sensor Service

- 센서 데이터 조회, 환경 변경 이력/평균 조회, 버섯 참조 데이터(RAG 컨텍스트) 조회

### Notification Service

- Telegram/Discord 챗봇 웹훅 수신 시 발신자 → cultivationId 조회

---

## 호출받는 서비스

### Cultivation Service

- 생육 사진 Vision 분석 요청

### Sensor Service

- Weekly Scheduler가 집계한 주간 통계 전달(push)

### API Gateway

- 챗봇/리포트/피드백/인사이트/가이드 REST API 요청

---

# Event

## Publish

### WeeklyReportCompletedEvent / DailyFeedbackCompletedEvent

Notification Service가 구독해 알림을 보냅니다.

## Subscribe

### HarvestCompletedEvent

Cultivation Service가 수확 기록 시 발행합니다. 인사이트 사례(`insight` 행) 생성을
트리거합니다.

---

# Sequence

관련 시퀀스는 [growth-analysis.md](../04_sequence/growth-analysis.md),
[ai-chat.md](../04_sequence/ai-chat.md), [ai-report.md](../04_sequence/ai-report.md),
[daily-feedback.md](../04_sequence/daily-feedback.md),
[insight.md](../04_sequence/insight.md) 참고.

---

# 예외 상황

- Vision 모델 분석 실패 / LLM 응답 실패
- chat_log/growth_record/daily_feedback/insight 저장 실패 (저장 실패해도 사용자 응답은 반환)
- Telegram/Discord 웹훅 수신 시 매칭되는 notification_endpoint 없음
- 인사이트 적재 시 Sensor Service 호출 실패 (환경 평균 조회 실패 — 해당 수확의 insight
  적재는 건너뛰며, 이벤트 유실/실패에 대비한 재처리는 추후 개발 예정)

---

# 추후 개발 예정

- 챗봇 봇 명령어로 다른 재배 지정
