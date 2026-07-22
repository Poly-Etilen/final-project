# 일일 피드백 생성 시퀀스

## 개요

사용자가 재배 환경(온도/습도/CO₂/조도)을 직접 수정했을 때, 그 수정이 실제 생육에
도움이 됐는지를 매일 알려주는 기능입니다. 예를 들어 `mushroom_reference`의 추천
온도보다 사용자가 임의로 온도를 높였는데, 그 이후 생육 점수가 개선되는 추세라면 이를
짚어줍니다.

AI Service의 Daily Scheduler가 매일 재배별로 생육 분석 이력(`growth_record`)과 환경
변경 이력(Sensor Service의 `environment_setting`)을 비교해 피드백을 생성합니다. 주간
리포트([ai-report.md](./ai-report.md))와 마찬가지로 push 모델이며, 사용자가 생성을
요청하지 않습니다.

전날 사용자가 생육 사진을 찍지 않았다면 비교할 `growth_record`가 없으므로, 그 경우에는
LLM을 호출하지 않고 고정된 안내 문구로 피드백을 대신합니다. 이 경우에도 그날의
`daily_feedback` 행은 반드시 생성됩니다(건너뛰지 않음) — 사용자가 "오늘은 피드백이 아예
없다"와 "오늘은 사진이 없어서 비교를 못 했다"를 구분할 수 있도록 하기 위해서입니다.

"성장 데이터 없음" 여부는 매일 독립적으로 그날 기준으로 판단합니다. 전날 사진을 못
찍었다고 다음날 판단에 영향을 주지는 않습니다.

---

# Sequence

```text
Daily Scheduler (AI Service, 매일 23:00)
↓
RUNNING 상태 재배 순회
↓
재배별로 growth_record 조회 (오늘 날짜 기준 존재 여부 + 최근 며칠 추이)
↓
Sensor Service OpenFeign 호출 (environment_setting 최근 변경 이력)
├── 오늘 growth_record 있음 → LLM 호출 (환경 변경 vs 생육 추이 상관관계 해석)
└── 오늘 growth_record 없음 → LLM 미호출, 고정 안내 문구 사용
↓
daily_feedback 저장 (PostgreSQL)
↓
RabbitMQ Publish (DailyFeedbackCompletedEvent) → Notification Service
```

---

# 상세 과정

## 1. Daily Scheduler 실행

AI Service의 Daily Scheduler가 매일 정해진 시각(23:00)에 실행되어, `RUNNING` 상태인
재배를 순회합니다.

---

## 2. growth_record 조회

재배별로 오늘 날짜에 해당하는 `growth_record`가 있는지 확인하고, 최근 며칠 치 추이도
함께 조회합니다.

```sql
SELECT * FROM growth_record
WHERE cultivation_id = ?
ORDER BY analyzed_at DESC
LIMIT 7;
```

---

## 3. environment_setting 조회

AI Service → OpenFeign → Sensor Service (`GET
/api/v1/sensors/cultivations/{cultivationId}/environment-history`)

```json
{
    "history": [
        { "measurementType": "TEMPERATURE", "min": 20.5, "max": 23.5, "createdAt": "2026-08-10T09:00:00" },
        { "measurementType": "TEMPERATURE", "min": 22.5, "max": 25.5, "createdAt": "2026-08-13T14:00:00" }
    ]
}
```

`environment_setting`이 항목별로 여러 행이 누적되는 구조이기 때문에, 최근 N일간의
변경 이력을 그대로 받아 "언제 얼마나 바뀌었는지"를 확인할 수 있습니다.

---

## 4-A. growth_record가 있는 경우

AI Service → LLM

입력: 최근 며칠간의 `growth_record` 추이(growthScore, myceliumGrowthRate 등), 같은
기간의 `environment_setting` 변경 이력

```json
{
    "hasGrowthData": true,
    "content": "온도를 22℃에서 24℃로 높인 이후 균사 성장률이 평균 6%p 개선되는 추세입니다. 현재 설정을 유지해도 좋아 보입니다."
}
```

---

## 4-B. growth_record가 없는 경우

LLM을 호출하지 않고 고정 문구를 사용합니다.

```json
{
    "hasGrowthData": false,
    "content": "전날 사진이 없어 피드백을 남길 수 없습니다."
}
```

---

## 5. daily_feedback 저장

```json
{
    "cultivationId": 3,
    "feedbackDate": "2026-08-15",
    "hasGrowthData": true,
    "content": "온도를 22℃에서 24℃로 높인 이후 균사 성장률이 평균 6%p 개선되는 추세입니다."
}
```

`(cultivation_id, feedback_date)` UNIQUE 제약으로 하루에 한 번만 생성됩니다.

---

## 6. 완료 이벤트 발행

AI Service → RabbitMQ Publish(`DailyFeedbackCompletedEvent`) → Notification Service
(사용자 알림)

---

## 7. 사용자 조회 (별도 흐름)

위 흐름과 독립적으로, 사용자는 언제든 지금까지 쌓인 일일 피드백을 조회할 수 있습니다.

```http
GET /api/v1/ai/feedback/daily?cultivationId=3
```

AI Service가 `daily_feedback`을 `cultivation_id`, 최신순으로 조회해 반환합니다.

---

# 사용 Database

## PostgreSQL

```
growth_record (AI DB, 조회 전용)
daily_feedback (AI DB)
```

---

# OpenFeign

```
AI Service → Sensor Service (environment_setting 최근 변경 이력 조회)
```

---

# RabbitMQ

Publish

```
DailyFeedbackCompletedEvent (AI Service, 매일 재배별 생성 완료 시)
```

Subscribe

```
Notification Service
```

---

# 예외 상황

- Daily Scheduler 실행 실패 (해당 날짜는 `daily_feedback`이 생성되지 않음)
- Sensor Service 호출 실패 (환경 변경 이력 조회 실패 — 환경 비교 없이 생육 추이만으로
  피드백을 생성하거나, 해당 재배는 다음날 재시도)
- LLM 응답 실패
- `daily_feedback` 저장 실패
- `DailyFeedbackCompletedEvent` 발행 실패 (피드백 자체는 저장되어 조회는 가능하지만
  알림이 가지 않음)

---

# 고려 사항

- 일일 피드백 생성은 사용자 요청이 아니라 Daily Scheduler가 트리거합니다. 사용자는
  조회만 합니다.
- 사진을 안 찍은 날도 `daily_feedback` 행은 반드시 생성됩니다(`hasGrowthData = false`,
  고정 문구). "피드백 없음"과 "비교 데이터 없음"을 구분하기 위해서입니다.
- Redis 캐시를 사용하지 않습니다. 하루에 한 번만 생성되고 바로 영구 저장되므로 별도
  캐시가 필요 없습니다.
- `growth_record`는 `ai:{cultivationId}:analysis` Redis 캐시(TTL 6시간, 생육 분석
  직후 빠른 재조회용)와 역할이 다릅니다. Redis 캐시는 만료되지만 `growth_record`는
  영구 보관되어 이 시퀀스처럼 여러 날짜에 걸친 추이 비교에 사용됩니다.
