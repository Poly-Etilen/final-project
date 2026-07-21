# AI Service

## 역할

AI Service는 LLM과 RAG(Retrieval-Augmented Generation)를 활용하여 사용자에게 지능형 서비스를 제공합니다.

센서 데이터를 분석하여 생육 상태를 예측하며, AI 리포트와 챗봇 기능을 제공합니다.

> ℹ️ **변경 이력**: 재배 환경 추천은 원래 AI Service가 Embedding/Vector Search/LLM을 통해
> 생성했지만, 공공데이터 기준 버섯 종류가 5가지로 고정되어 있어 항상 같은 값이 나오는 조회에는
> AI가 불필요하다고 판단해 Cultivation Service의 `mushroom_reference` 참조 테이블 조회로
> 이전했습니다. AI Service는 더 이상 환경 추천을 담당하지 않습니다.

> ℹ️ **변경 이력**: 재배 생성 직후 사용자에게 버섯 효능/재배 시 주의사항을 자연어로 보여주는
> "버섯 가이드" 기능이 추가되었습니다. 환경 추천(수치, 범위)과 달리 이 기능은 사람이 읽기 좋은
> 설명 문서를 생성하는 것이 목적이라 LLM을 그대로 활용하기로 했습니다. 다만 버섯 종류가
> 5가지로 고정되어 있는 것은 환경 추천 때와 동일하므로, 재배(cultivation)가 아닌
> **버섯 종류(mushroomType) 기준으로 캐싱**하여 동일 종류에 대해 매번 LLM을 호출하지 않도록
> 했습니다.

> ℹ️ **변경 이력**: `mushroom_reference`(Cultivation DB)에 특성/효능/재배 가이드/추가 정보
> 원문 텍스트가 추가되면서, "버섯 가이드"는 이 원문을 무(無)에서 생성하지 않고 **RAG 컨텍스트로
> 활용**합니다. AI Service가 Cultivation Service를 OpenFeign으로 호출해(`GET
> /api/v1/mushroom-references/{mushroomType}`) 원문 텍스트를 가져온 뒤, LLM이 이를 참고해
> 더 자연스러운 문장의 `benefits`/`precautions`로 다듬어 응답합니다. 원문을 그대로 반환하지
> 않는 이유는, 공공데이터 원문이 항목별로 파편화되어 있어 사용자에게는 자연어로 통합된 설명이
> 더 읽기 좋기 때문입니다.

> ℹ️ **변경 이력**: AI 챗봇의 대화 이력을 조회하는 기능이 추가되었습니다. 기존에는 `ai:{hash}`
> Redis 캐시(동일 질문 재요청 시 LLM 재호출 방지용)만 있어서 사용자가 이전 대화를 다시 볼 수
> 없었습니다. 이번 변경으로 매 질의응답을 `chat_message` 테이블에 저장하며, AI Service가
> 처음으로 PostgreSQL DB를 갖게 되었습니다. `ai:{hash}` 캐시는 역할이 겹치지 않아 그대로
> 유지합니다. (자세한 내용은 [ai-db.md](../03_Database/ai-db.md) 참고)

> ℹ️ **변경 이력**: 월간 리포트를 폐기했습니다. 버섯 재배 기간이 한 달을 넘지 않아 "월간"
> 단위가 의미가 없다고 판단했습니다. 또한 AI 리포트 생성 방식을 "사용자 요청 시 그 자리에서
> 생성"(pull)에서 **Sensor Service의 Weekly Scheduler가 먼저 집계 데이터를 전달하고 AI
> Service가 미리 리포트를 만들어 두는 방식**(push)으로 재정리했습니다. 두 방식이 문서에 섞여
> 있어 서로 모순됐던 부분을 정리한 것입니다. (자세한 내용은
> [ai-report.md](../04_sequence/ai-report.md) 참고)

> ℹ️ **변경 이력**: "일일 피드백" 기능이 추가되었습니다. 사용자가 재배 환경(온도 등)을
> 수정했을 때, 그 수정이 실제로 생육에 긍정적이었는지를 매일 알려주는 기능입니다. 이를 위해
> AI Service에 `growth_record`(생육 분석 결과 이력), `daily_feedback`(일일 피드백 결과)
> 테이블이 추가되었고, Daily Scheduler가 매일 재배별로 환경 변경 이력(Cultivation Service의
> `environment_setting`)과 생육 추이(`growth_record`)를 비교해 피드백을 생성합니다. 사용자가
> 전날 사진을 찍지 않았다면 비교할 데이터가 없으므로, 그 경우에도 피드백 자체는 생성하되
> 고정된 안내 문구를 남깁니다. (자세한 내용은 [ai-db.md](../03_Database/ai-db.md),
> [daily-feedback.md](../04_sequence/daily-feedback.md),
> [cultivation-db.md](../03_Database/cultivation-db.md)의 `environment_setting` 참고)

---

# 책임

- AI 생육 분석
- AI 챗봇
- 일일 피드백 생성
- AI 리포트 생성
- 버섯 가이드(효능/주의사항) 생성
- 프롬프트 관리
- AI 응답 캐싱

---

# 주요 기능

## AI 생육 분석

환경 데이터만으로 추정하지 않고, 사용자가 직접 촬영하여 업로드한 사진을 Vision 모델로 분석하여
실제 생육 상태를 판단합니다.

카메라 센서가 자동으로 촬영하는 방식이 아니라, 사용자가 앱/웹에서 사진을 찍어 업로드하면
그 사진을 AI가 학습된 기준(성장 단계별 학습 데이터)과 비교하여 분석하는 방식입니다.

Vision 모델은 AI Service 내부에 포함되며, LLM과는 별도로 동작합니다.

### 분석 흐름

1. Cultivation Service가 사용자로부터 업로드받은 사진의 URL을 AI Service에 전달합니다.

사진 원본은 Cultivation Service가 MinIO에 저장한 상태이며, AI Service는 전달받은 URL로 이미지를 조회합니다.

---

2. AI Service의 Vision 모델이 사진을 분석합니다.

Vision 모델은 사전에 다양한 성장 단계의 사진으로 학습되어 있으며,
입력된 사진을 학습된 패턴과 비교하여 아래 지표를 산출합니다.

```json
{
    "myceliumGrowthRate":82,
    "capSize":"중(3.2cm)",
    "colorStatus":"정상",
    "diseaseStatus":"정상"
}
```

분석 항목

- 균사 성장률 : 균사가 배지를 덮은 비율
- 갓 크기 : 버섯 갓의 지름
- 색상 분석 : 정상 / 변색 / 갈변 등 색상 상태
- 병충해 분석 : 정상 / 의심 / 감염 여부

---

3. AI Service가 4가지 지표를 종합하여 생육 점수를 계산합니다.

```
생육 점수 = 균사 성장률 40% + 갓 크기 점수 30% + 색상 점수 15% + 병충해 점수 15%
```

병충해가 감지된 경우 생육 점수를 별도로 감점합니다.

---

4. 성장 단계(균사기 / 자실체 형성기 / 성장기 / 수확 적기)를 기준으로 예상 수확 시기를 추정합니다.

---

5. LLM은 Vision 모델이 산출한 지표(생육 점수, 성장 단계, 병충해 여부)를 입력받아
   결과를 해석하는 자연어 설명과 개선 방안만 생성합니다.
   지표 자체를 새로 추정하지 않습니다.

6. 분석 결과는 `ai:{cultivationId}:analysis` Redis 캐시(TTL 6시간, 빠른 재조회용)에 저장되는
   것과 별개로, `growth_record` 테이블(PostgreSQL)에도 한 행으로 영구 저장됩니다. Redis 캐시는
   TTL이 지나면 사라지지만, `growth_record`는 "일일 피드백"이 여러 날짜의 생육 추이를 비교하는
   데 계속 사용되므로 만료되지 않습니다.

### 제공 정보

- 생육 점수
- 균사 성장률
- 갓 크기
- 색상 상태
- 병충해 여부
- 성장 단계
- 예상 수확 시기
- 환경 개선 사항 (LLM 생성)

---

## 버섯 가이드 생성 (효능/주의사항)

재배를 생성한 직후, 선택한 버섯 종류의 효능과 재배 시 주의사항을 자연어 문서로 보여줍니다.

`mushroom_reference.description`(짧은 한 줄 참고 문구, Cultivation Service가 재배 생성 응답에
그대로 포함)과는 별개입니다. 버섯 가이드는 그보다 훨씬 자세한 설명(효능, 주의사항)을 LLM으로
생성하는 별도 기능이며, Client가 재배 생성 이후 AI Service를 직접 호출해서 받습니다
(Cultivation Service를 거치지 않습니다. 다만 AI Service는 응답을 만들기 위해 내부적으로
Cultivation Service를 OpenFeign으로 호출합니다).

`mushroom_reference`에 저장된 characteristics(특성)/health_benefits(효능)/cultivation_guide
(재배 가이드)/additional_info(추가 정보) 원문을 Cultivation Service로부터 가져와 LLM
프롬프트의 RAG 컨텍스트로 사용합니다. LLM은 이 원문을 그대로 반환하지 않고, 자연스러운 문장의
`benefits`/`precautions`로 재구성합니다.

버섯 종류는 공공데이터 기준 5가지로 고정되어 있어 같은 종류라면 항상 같은 내용이 나오므로,
`mushroomType` 기준으로 캐싱해 동일 종류에 대한 반복 LLM 호출을 피합니다. (재배 환경 추천을
AI에서 뺀 이유와 같은 문제이지만, 이번에는 수치가 아닌 "설명 문서"를 만드는 것이라 LLM을
그대로 사용하기로 했습니다.)

### 제공 정보

- 효능 (buffs)
- 재배 시 주의사항 (precautions)

---

## AI 챗봇

사용자는 자연어로 재배 관련 질문을 할 수 있습니다.

예시

- 왜 성장이 느린가요?
- 현재 환경은 적절한가요?
- 언제 수확하면 좋을까요?
- 생산량을 늘리려면 어떻게 해야 하나요?

매 질의응답은 `chat_message` 테이블에 저장됩니다.

---

## AI 챗봇 대화 이력 조회

특정 재배에 대해 사용자가 챗봇과 나눈 이전 질문/답변을 최신순으로 조회합니다.

---

## 일일 피드백

사용자가 재배 환경(온도/습도/CO₂/조도)을 직접 수정했을 때, 그 수정이 생육에 실제로 도움이
됐는지를 매일 알려주는 기능입니다. 예를 들어 mushroom_reference의 추천 온도보다 사용자가
임의로 온도를 높였는데, 그 이후 생육 점수가 개선되는 추세라면 이를 짚어줍니다.

Daily Scheduler가 매일 재배별로 아래 두 데이터를 비교합니다.

- `growth_record`(AI Service 자체 DB): 최근 며칠간의 생육 분석 결과 추이
- `environment_setting`(Cultivation Service): 최근 환경 변경 이력

사진을 찍지 않아 그날의 `growth_record`가 없으면 비교할 데이터가 없으므로, LLM을 호출하지
않고 "전날 사진이 없어 피드백을 남길 수 없습니다"라는 고정 문구로 피드백을 생성합니다. 이
경우에도 그날의 `daily_feedback` 행 자체는 생성됩니다(건너뛰지 않음).

### 제공 정보

- 피드백 문장 (환경 변경과 생육 추이의 상관관계, 또는 사진 없음 안내)
- 대상 날짜
- 비교 데이터 존재 여부

---

## 일일 피드백 조회

특정 재배에 대해 지금까지 생성된 일일 피드백을 최신순으로 조회합니다.

---

## AI 리포트

Sensor Service의 Weekly Scheduler가 집계한 주간 데이터를 전달받아 AI 리포트를 생성합니다.
사용자가 요청하는 시점이 아니라, Scheduler가 매주 먼저 리포트를 만들어 둡니다(push).
재배 기간이 한 달을 넘지 않아 월간 리포트는 만들지 않습니다.

제공 내용

- 환경 유지율
- 평균 환경 데이터
- 자동 제어 횟수
- 이상 환경 발생 내역
- 개선 제안

---

# API

## 생육 분석

POST /ai/analysis

---

## 버섯 가이드 생성

POST /ai/mushroom-guide

---

## AI 챗봇

POST /ai/chat

---

## AI 챗봇 대화 이력 조회

GET /ai/chat/history

---

## 일일 피드백 조회

GET /ai/feedback/daily

일일 피드백 생성 자체는 사용자가 호출하는 API가 아니라 Daily Scheduler가 매일 자동으로
수행합니다. 이 엔드포인트는 이미 생성된 피드백을 조회만 합니다.

---

## AI 리포트 조회

GET /ai/report

리포트 생성 자체는 사용자가 호출하는 API가 아니라 Sensor Service의 Weekly Scheduler가
전달한 데이터를 받아 AI Service가 매주 자동으로 생성합니다. 이 엔드포인트는 가장 최근 생성된
리포트를 조회만 합니다.

---

# Database

AI Service는 하나의 PostgreSQL Database를 사용합니다.

### Table

- chat_message (챗봇 대화 이력)
- growth_record (생육 분석 결과 이력)
- daily_feedback (일일 피드백 이력)

자세한 내용은 [ai-db.md](../03_Database/ai-db.md) 참고.

---

# Redis

AI 응답 캐시를 저장합니다.

캐시 대상

- AI 분석 결과 (`growth_record`에도 영구 저장되는 것과 별개로, 빠른 재조회용 캐시)
- AI 리포트 (Weekly Scheduler가 push로 미리 채워둠, 사용자 요청 시 생성하지 않음)
- AI 챗봇 응답
- 버섯 가이드 (mushroomType 기준, cultivationId와 무관)

일일 피드백은 Redis에 캐시하지 않습니다. 하루에 한 번만 생성되고 `daily_feedback`에 바로
영구 저장되므로 별도 캐시가 필요하지 않습니다.

---

# MinIO

AI Service는 Vision 분석을 위해 사진을 조회합니다.

사진을 직접 저장하지는 않으며, Cultivation Service가 저장한 이미지를 읽기 전용으로 사용합니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Embedding Service

- AI 챗봇의 유사 재배 사례 검색 (선택적 호출)

---

### Sensor Service

- 센서 데이터 조회 (AI 챗봇에서 참고용으로 사용)

주간 데이터는 더 이상 AI Service가 요청 시점에 조회하지 않습니다. Sensor Service의 Weekly
Scheduler가 먼저 집계해 전달합니다(아래 "호출받는 서비스" 참고).

---

### Cultivation Service

- 버섯 가이드 생성 시 `GET /api/v1/mushroom-references/{mushroomType}` 호출 (RAG 컨텍스트 조회, 캐시 미스 시에만)
- 일일 피드백 생성 시 재배의 `environment_setting`(최근 변경 이력) 조회 (Daily Scheduler 실행 시)

---

## 호출받는 서비스

### Cultivation Service

- 생육 사진 Vision 분석 요청 (사진 URL 포함)

### Sensor Service

- Weekly Scheduler가 집계한 주간 통계를 전달(push)하며, AI Service는 이를 받아 리포트를
  생성합니다.

### API Gateway

- AI 챗봇 요청
- 버섯 가이드 요청 (Client가 직접 호출, Cultivation Service를 거치지 않음)
- AI 리포트 조회, 일일 피드백 조회 요청

---

# Event

## 발행 이벤트

### WeeklyReportCompletedEvent

Sensor Service의 Weekly Scheduler가 집계한 데이터를 받아 AI 리포트 생성이 완료되면 발행합니다.

구독 서비스: Notification Service

---

### DailyFeedbackCompletedEvent

Daily Scheduler가 재배별 일일 피드백 생성을 완료하면 발행합니다.

```json
{
    "cultivationId": 3,
    "feedbackDate": "2026-08-15",
    "hasGrowthData": true,
    "createdAt": "2026-08-15T23:00:00"
}
```

구독 서비스: Notification Service

---

생육 분석(Vision), 챗봇은 사용자 요청에 대한 동기 응답으로 결과가 즉시 전달되므로 별도 이벤트를 발행하지 않습니다.

---

# Scheduler

## Daily Scheduler

매일 정해진 시각(예: 23:00)에 RUNNING 상태인 재배를 순회하며 일일 피드백을 생성합니다.

```
RUNNING 상태 재배 순회

↓

재배별로 growth_record(최근 며칠) 조회

↓

Cultivation Service OpenFeign 호출 (environment_setting 최근 변경 이력 조회)

↓

그날 growth_record 있음 → LLM으로 환경 변경과 생육 추이의 상관관계 해석
그날 growth_record 없음 → LLM 호출 없이 고정 안내 문구 사용

↓

daily_feedback 저장 (PostgreSQL)

↓

RabbitMQ Publish (DailyFeedbackCompletedEvent)
```

Sensor Service의 Weekly Scheduler와 달리, AI Service가 직접 스케줄러를 갖습니다. 필요한
데이터(생육 분석 이력, 환경 변경 이력)가 모두 AI Service 자신 또는 Cultivation Service에
있어 Sensor Service를 거칠 필요가 없기 때문입니다.

---

# Sequence

## 버섯 가이드 생성

Client (재배 생성 완료 직후)

↓

AI Service

↓

Redis 캐시 조회 (ai:mushroom:{mushroomType}:guide)

↓

Cache Hit → 즉시 반환

↓

Cache Miss → Cultivation Service OpenFeign 호출 (`GET /api/v1/mushroom-references/{mushroomType}`, RAG 컨텍스트 조회)

↓

LLM 호출 (원문을 참고해 효능/주의사항 생성) → Redis 캐시 저장 (TTL 7일) → 반환

↓

Client

---

## AI 챗봇

Client

↓

AI Service

↓

Redis 캐시 조회 (ai:{hash})

↓

Cache Miss → Sensor Service 조회 + Embedding Service 유사 사례 검색(선택) → LLM

↓

chat_message 저장 (PostgreSQL)

↓

Client

---

## AI 챗봇 대화 이력 조회

Client

↓

AI Service

↓

chat_message 조회 (cultivation_id, 최신순)

↓

Client

---

## AI 리포트 생성

Weekly Scheduler (Sensor Service)

↓

주간 데이터 집계 (InfluxDB)

↓

AI Service (OpenFeign, push)

↓

LLM

↓

Redis 저장 (report:{cultivationId}:weekly) + RabbitMQ Publish (WeeklyReportCompletedEvent)

사용자는 이 흐름과 별개로 `GET /ai/report`로 이미 생성된 리포트를 언제든 조회할 수 있습니다.

---

## AI 리포트 조회

Client

↓

AI Service

↓

Redis 조회 (report:{cultivationId}:weekly)

↓

Client

---

## 일일 피드백

Daily Scheduler (AI Service)

↓

growth_record 조회 (그날 분석 결과 존재 여부 확인)

↓

Cultivation Service OpenFeign 호출 (environment_setting 최근 변경 이력)

↓

그날 growth_record 있음 → LLM 호출 (환경 변경 vs 생육 추이 상관관계 해석)
그날 growth_record 없음 → 고정 문구 사용 (LLM 미호출)

↓

daily_feedback 저장 (PostgreSQL)

↓

RabbitMQ Publish (DailyFeedbackCompletedEvent) → Notification Service

---

## 일일 피드백 조회

Client

↓

AI Service

↓

daily_feedback 조회 (cultivation_id, 최신순)

↓

Client

---

# 예외 상황

- Embedding 검색 실패
- LLM 응답 실패 (버섯 가이드 생성 실패 포함)
- 버섯 가이드 생성 시 Cultivation Service 호출 실패 (RAG 컨텍스트 조회 실패)
- Redis Cache 조회 실패
- chat_message 저장 실패 (PostgreSQL) — 저장에 실패해도 챗봇 응답 자체는 사용자에게 반환합니다
- 다른 사용자의 재배에 대한 대화 이력 조회 시도
- Sensor 데이터 부족
- 등록된 사진 없음
- Vision 모델 분석 실패
- growth_record 저장 실패 (PostgreSQL) — 저장에 실패해도 생육 분석 응답 자체는 사용자에게 반환합니다
- 일일 피드백 생성 시 Cultivation Service 호출 실패 (environment_setting 조회 실패)
- daily_feedback 저장 실패 (PostgreSQL)
- Weekly Scheduler로부터 데이터 전달 실패 (해당 주는 리포트가 갱신되지 않고 이전 리포트가 Redis에 남아있음)
- API 호출 시간 초과

---

# 추후 개발 예정

- 사용자 맞춤형 프롬프트
- AI 응답 품질 평가
- 모델 교체(OpenAI, Gemini 등)
- 다국어 지원
- AI 재배 코칭