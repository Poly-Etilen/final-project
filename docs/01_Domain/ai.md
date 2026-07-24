# AI Service

## 역할

AI Service는 LLM과 Vision 모델을 활용해 지능형 기능을 제공하는 서비스입니다. 생육 사진
분석, 자연어 챗봇(웹 WebSocket 채팅방 + Telegram/Discord), 일일 피드백(생육 추이 비교 +
환경 통계), "인사이트"(타인의 유사 재배 사례 기반 피드백), 버섯 가이드(효능/주의사항)를
담당합니다.

챗봇은 용도가 다른 두 채널로 나뉩니다. 웹은 WebSocket 기반 채팅방이며 '/' 명령어(조리법 등
공공데이터 조회, `/인사이트`)를 처리할 수 있고, Telegram/Discord는 알림 수신과 자연어
질의응답만 제공하는 채널입니다(명령어·멀티유저 채팅방 없음).

---

# 책임

- 생육 사진 Vision 분석
- AI 챗봇 — 웹(WebSocket 채팅방, '/' 명령어) / Telegram·Discord(자연어 질의응답), 대화
  이력 저장/조회
- 일일 피드백 생성 (생육 추이 비교 + 환경 통계, Daily Scheduler)
- 인사이트 사례 적재(수확 완료 시점) 및 조회
- 버섯 가이드(효능/주의사항) 생성
- 상품 등급 원점수 계산(수확 완료 시점) 및 Cultivation Service 전달

---

# 주요 기능

## AI 생육 분석 (Vision)

사용자가 업로드한 사진을 Vision 모델로 분석해 생육 점수/균사 성장률/갓 크기/색상/병충해/
성장 단계/예상 수확일을 반환합니다. 분석 결과는 Redis(`ai:{cultivationId}:analysis`,
TTL 6시간)로 빠른 재조회를 지원하는 동시에 `growth_record`에 영구 저장해 일일 피드백의
추이 비교에 사용합니다. `growth_record`는 지표들을 컬럼별로 나누지 않고
`analysis_data`(JSONB) 하나에 담아 저장하며, 분석에 사용한 사진은 파일 정보를
복사하지 않고 `cultivation_photo_id`(소프트 참조)만 저장합니다.

---

## AI 챗봇

사용자는 자연어로 재배 관련 질문을 할 수 있습니다. 대화는 대화방
단위(`chat_conversation`)와 발화 단위(`chat_message`) 두 테이블로 나눠 저장합니다 —
매 발화(질문/응답)는 소속 대화방 안에서 `sequence_number`가 증가하는 `chat_message`
행으로 한 행씩 쌓입니다. 채널마다 제공하는 기능이 다릅니다.

### 웹 챗봇 (WebSocket, `channel_type=APP`)

Client가 WebSocket으로 연결해 채팅방 형태로 대화합니다(연결 시 Bearer JWT로 인증).
`userId`는 JWT에서, `cultivationId`는 연결 시 지정합니다(선택).

일반 자연어 질문 외에 `/`로 시작하는 명령어를 처리합니다.

- `/`로 시작하는 명령어로 조리법 등 공공데이터 API 기반 정보를 조회해 응답합니다.
- `/인사이트` 명령어를 입력하면 [인사이트](#인사이트) 절의 후보 조회를 호출해 리스트를
  보여주고, 사용자가 그중 하나를 선택하면 후보 상세 조회로 이어집니다.

### Telegram/Discord 챗봇 (`channel_type=TELEGRAM`/`DISCORD`)

Notification Service가 보내는 알림을 수신하는 채널이면서, 동시에 "지금 환경은 어때?"와
같은 자연어 질문에 AI가 답변하는 채널입니다. 사용자가 봇에게 메시지를 보내면 웹훅으로
전달됩니다. AI Service는 발신자 Chat ID로 Notification Service의 `notification_endpoint`를
조회해 `cultivationId`를 알아냅니다(알림 채널이 재배 단위로 등록되므로). 이 경로로는 개별
사용자를 특정할 수 없어 `chat_conversation.user_id`는 NULL로 저장되지만, 대신 항상 특정
재배 맥락(Sensor 데이터 조회 포함)에서 답변할 수 있습니다. 매칭되는 endpoint가 없으면
"먼저 이 채널을 재배에 등록해주세요" 안내로 응답합니다.

웹 챗봇과 달리 `/` 명령어나 여러 사용자가 함께 쓰는 채팅방 기능은 제공하지 않습니다 —
관련 설계는 아직 논의 중이며 이 문서에서는 다루지 않습니다.

### 공통

답변 생성 시 Cultivation Service에서 현재 환경/통계를 조회하고, 필요하면 같은 호출로 해당
재배의 버섯 종류에 맞는 `mushroom_reference` 텍스트(효능/재배 가이드)를 함께 받아 LLM
컨텍스트로 사용합니다. 버섯 종류가 5종으로 고정되어 있어 `mushroomId`로 정확히
일치하는 한 건만 조회하면 되므로, 별도의 유사도 검색 없이 직접 조회로 충분합니다.

동일 질문 반복 호출을 줄이기 위해 Redis(`ai:{hash}`, TTL 24시간)에 응답을 캐싱합니다.
명령어 처리(공공데이터 조회, `/인사이트`)는 이 응답 캐시 대상이 아닙니다.

---

## 일일 피드백

사용자가 환경(온도/습도/CO₂/조도)을 직접 수정했을 때, 그 수정이 생육에 실제로 도움이
됐는지 매일 알려주는 동시에, 지난 24시간의 환경 통계(평균/최고/최저 온도·습도·CO₂·조도)를
함께 요약해 제공하는 기능입니다. 재배 기간이 한 달을 넘지 않는 도메인 특성상 별도의
주간/월간 리포트는 두지 않고, 이 기능 하나로 통합해 제공합니다.

Daily Scheduler가 매일 재배별로 `growth_record`(생육 추이), Cultivation Service의
`environment_setting`(환경 변경 이력), Cultivation Service의 InfluxDB 집계(최근 24시간 환경
통계)를 모아 LLM으로 피드백을 생성하고 `daily_feedback`에 저장합니다. 사진을 찍지 않아
비교할 `growth_record`가 없으면 생육 비교 부분만 LLM을 호출하지 않고 고정 문구로
대신합니다(이 경우도 `daily_feedback` 행은 생성되며, 환경 통계는 그대로 함께 저장됨).

---

## 인사이트

같은 버섯 종류 + 유사한 환경으로 재배했던 타인의 사례를 바탕으로 현재 재배에 도움이
되는 정보를 보여주는 기능입니다. 데이터 적재(수확 완료 시점)와 조회(on-demand)를
분리하며, 조회는 웹 챗봇의 `/인사이트` 명령어로 시작하는 2단계 흐름입니다.

- **적재**: Cultivation Service가 수확을 기록하면 발행하는 `HarvestCompletedEvent`를
  AI Service가 구독해, 그 수확의 재배에 대한 환경 평균(Cultivation Service 조회)과 최근
  생육 점수(`growth_record` 자체 조회)를 묶어 AI DB의 `insight` 테이블에 한 건
  저장합니다. 이때 한 줄 요약(`summary`)도 LLM으로 미리 생성해 함께 저장합니다. 별도
  배치나 임계치 없이 수확이 기록될 때마다 바로 반영됩니다.
- **① 후보 조회**: 버섯 종류(정확히 일치) + 온습도/CO2/조도 4개 항목이 모두 오차 범위
  내인 사례 중, 요청한 사용자가 속한 재배(OWNER/MEMBER 불문)를 제외하고 최신순으로
  최대 5개를 후보 리스트로 반환합니다. 재배 기간이 사용자마다 크게 달라 일자별 비교나
  단시간 스냅샷 비교로는 유의미한 차이가 나오지 않는다고 판단해, 재배 전체 기간의
  "기간 가중 평균"으로만 유사 사례를 찾습니다. 이 단계는 LLM을 호출하지 않습니다.
- **② 후보 상세 조회**: 사용자가 후보 하나를 선택하면, 그 사례의 재배가 겪은 날짜별
  환경 통계 + `daily_feedback` 내용을 날짜순으로 그대로 보여줍니다. 수확일에는 그날의
  `daily_feedback` 대신 적재 시점에 이미 만들어 둔 `insight.summary`로 대체합니다.
  추가 LLM 호출 없이 저장된 데이터를 그대로 조립해 보여주는 단계입니다.

자세한 흐름은 [insight.md](../04_sequence/insight.md) 참고.

---

## 상품 등급

수확이 기록되면(`HarvestCompletedEvent`) 인사이트 적재와 함께, 그 재배의 상품 등급
원점수(`productScore`, 0~100)를 계산합니다.

- 생육 점수 평균: 그 재배의 모든 `growth_record.analysis_data->>'growthScore'` 평균
  (자체 DB 조회, 호출 없음)
- 환경 유지 점수: 그 재배 기간 동안 측정 항목별로 추천 범위 안에 있었던 시간 비율의
  평균 — Cultivation Service에 새로 집계 조회(측정값 원본인 InfluxDB는 AI Service가
  직접 접근하지 않으므로, Cultivation Service가 항목별 비율을 미리 집계해 반환)

두 값을 합산해 만든 원점수 하나만 Cultivation Service에 전달합니다(둘을 따로 넘기지
않음). 등급(TOP/HIGH/MID/LOW) 매핑은 Cultivation Service가 자신의 `harvest` 테이블에
저장하면서 수행합니다 — AI Service는 등급 구간을 알 필요가 없습니다.

자세한 흐름은 [product-grade.md](../04_sequence/product-grade.md) 참고.

---

## 버섯 가이드

재배 생성 직후 버섯의 효능/재배 주의사항을 자연어로 보여줍니다. Cultivation Service의
`mushroom_reference` 텍스트 컬럼을 RAG 컨텍스트로 사용해 LLM이 자연스러운 문장으로
재구성합니다. `mushroomId`(5종 고정) 기준으로 캐싱(`ai:mushroom:{mushroomId}:guide`,
TTL 7일)해 반복 호출을 피합니다.

---

# API

## 생육 분석

POST /ai/analysis

---

## 챗봇

WS /ai/chat (웹 채팅방 연결, JWT 인증 — 기존 `POST /ai/chat` 단건 요청을 대체)

POST /ai/chat/telegram/webhook (내부용)

POST /ai/chat/discord/webhook (내부용)

GET /ai/chat/history

---

## 피드백/인사이트/가이드

GET /ai/feedback/daily

GET /ai/insight/candidates (후보 리스트, 최대 5개)

GET /ai/insight/candidates/{insightId} (후보 상세 — 날짜별 환경/피드백)

POST /ai/mushroom-guide

---

# Database

AI Service는 하나의 PostgreSQL Database를 사용합니다.

## Table

- chat_conversation
- chat_message
- growth_record
- daily_feedback
- insight

자세한 내용은 [ai-db.md](../03_Database/ai-db.md) 참고.

---

# Redis

- 챗봇 응답 캐시, 생육 분석 결과 캐시, 버섯 가이드 캐시, 인사이트 캐시

일일 피드백(환경 통계 포함)은 Redis에 캐시하지 않습니다. 하루에 한 번만 생성되어
`daily_feedback` 테이블에 바로 영구 저장되기 때문입니다.

자세한 내용은 [redis.md](../03_Database/redis.md) 참고.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Cultivation Service

- 생육 사진 조회, 챗봇/인사이트 조회 시 버섯 종류 조회, 센서 데이터 조회, 환경 변경
  이력/평균/일간 통계 조회, 버섯 참조 데이터(RAG 컨텍스트) 조회, 인사이트 후보 조회 시
  내가 속한 cultivation_id 목록 조회(제외용), 환경 준수율 집계 조회(상품 등급용), 상품
  등급 원점수 전달

### Notification Service

- Telegram/Discord 챗봇 웹훅 수신 시 발신자 → cultivationId 조회

### 외부 공공데이터 API

- 웹 챗봇의 `/` 명령어 처리 시 조리법 등 정보 조회(예: 식품안전나라 등 공공데이터 API)

---

## 호출받는 서비스

### Cultivation Service

- 생육 사진 Vision 분석 요청

### API Gateway

- 챗봇/피드백/인사이트/가이드 REST API 요청

---

# Event

## Publish

### DailyFeedbackCompletedEvent

Notification Service가 구독해 알림을 보냅니다.

## Subscribe

### HarvestCompletedEvent

Cultivation Service가 수확 기록 시 발행합니다. 인사이트 사례(`insight` 행) 생성과
상품 등급 원점수 계산을 함께 트리거합니다.

---

# Sequence

관련 시퀀스는 [growth-analysis.md](../04_sequence/growth-analysis.md),
[ai-chat.md](../04_sequence/ai-chat.md),
[daily-feedback.md](../04_sequence/daily-feedback.md),
[insight.md](../04_sequence/insight.md),
[product-grade.md](../04_sequence/product-grade.md) 참고.

---

# 예외 상황

- Vision 모델 분석 실패 / LLM 응답 실패
- chat_conversation/chat_message/growth_record/daily_feedback/insight 저장 실패
  (저장 실패해도 사용자 응답은 반환)
- Telegram/Discord 웹훅 수신 시 매칭되는 notification_endpoint 없음
- 웹 챗봇 WebSocket 연결 끊김 / 재연결
- `/` 명령어 처리 중 외부 공공데이터 API 호출 실패
- 인사이트 적재 시 Cultivation Service 호출 실패 (환경 평균 조회 실패 — 해당 수확의 insight
  적재는 건너뛰며, 이벤트 유실/실패에 대비한 재처리는 추후 개발 예정)
- 상품 등급 계산 시 Cultivation Service의 환경 준수율 집계 조회 실패, 또는 계산한
  원점수 전달 실패 (인사이트 적재와는 독립적으로 처리 — 한쪽이 실패해도 다른 쪽은
  계속 진행하며, 실패 시 `harvest.product_score`/`product_grade`는 NULL로 남음)

---

# 추후 개발 예정

- Telegram/Discord 챗봇의 `/` 명령어·멀티유저 채팅방 지원 여부 (설계 논의 중)
