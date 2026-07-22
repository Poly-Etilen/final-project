# 인사이트 시퀀스

## 개요

"일일 피드백"이 **내 재배**의 어제와 오늘을 비교하는 기능이라면, 인사이트는 같은 버섯
종류이면서 유사한(오차 범위 내) 온도로 재배했던 **타인의 완료된 재배 사례**를 바탕으로
현재 내 재배 상태를 피드백해주는 기능입니다.

이 기능은 서로 독립적인 두 흐름으로 구성됩니다.

1. **인사이트 사례 적재** (이벤트 기반, push): Cultivation Service가 수확(harvest)을
   기록할 때마다 발행하는 `HarvestCompletedEvent`를 AI Service가 구독해, 그 수확 건의
   환경 평균(Cultivation Service 조회)과 최근 생육 점수(`growth_record` 자체 조회)를 묶어
   AI DB의 `insight` 테이블에 즉시 한 건 저장합니다. 별도 배치나 임계치 없이 수확이
   기록되는 즉시 반영됩니다.
2. **인사이트 조회** (사용자 요청 시점, on-demand): 적재와 완전히 별개로, 사용자가
   특정 재배에 대해 인사이트를 요청한 시점에만 검색/요약이 이루어집니다. 일일 피드백/AI
   리포트처럼 미리 생성해두지 않습니다.

두 흐름이 섞이지 않도록 유의해야 합니다 — 적재가 "데이터를 쌓는" 흐름이라면, 조회는
이미 쌓인 데이터에서 "찾아서 보여주는" 흐름입니다.

---

# Sequence

## ① 인사이트 사례 적재

```text
Cultivation Service (수확 기록) → HarvestCompletedEvent 발행
↓
AI Service (구독)
↓
Cultivation Service OpenFeign 호출 (GET /../environment-average, 해당 재배의 환경 평균 조회)
↓
AI Service 자체 growth_record 조회 (최근 생육 점수)
↓
insight 테이블 저장 (AI DB, PostgreSQL)
```

---

## ② 인사이트 조회

```text
Client (사용자가 특정 재배의 인사이트 요청)
↓
API Gateway
↓
AI Service
↓
Redis 캐시 조회 (ai:{cultivationId}:insight)
├── Cache Hit → 즉시 반환
└── Cache Miss ↓
Cultivation Service OpenFeign 호출 (GET /cultivations/{cultivationId}, 버섯 종류 조회
+ GET /../environment-average, 환경 평균 조회)
↓
insight 테이블 SQL 검색 (AI DB, mushroomType 정확히 일치 + 온도 오차 범위)
├── 매칭 사례 있음 → LLM 요약 → Redis 캐시 저장 (TTL 24시간) → 반환
└── 매칭 사례 없음 → LLM 미호출, 고정 안내 문구 반환 (캐시하지 않음)
↓
Client
```

---

# 상세 과정

## 1. HarvestCompletedEvent 구독

Cultivation Service가 수확(flush)을 기록할 때마다 `HarvestCompletedEvent`를
발행하며, AI Service는 이 이벤트를 구독해 인사이트 사례 적재를 트리거합니다. 별도의
스케줄러나 임계치 없이, 이벤트를 받을 때마다 바로 처리합니다.

```json
{
    "harvestId": 41,
    "cultivationId": 12,
    "flushNo": 1,
    "harvestWeight": 3200
}
```

---

## 2. 환경 평균 조회

이벤트에 담긴 `cultivationId`로 Cultivation Service에 환경 평균을 조회합니다.

```http
GET /api/v1/cultivations/12/environment-average
```

```json
{ "avgTemperature": 21.8, "avgHumidity": 89.2, "avgCo2": 780.5, "avgLight": 360.0 }
```

`avgTemperature`~`avgLight`는 Cultivation Service가 `environment_setting` 이력 전체를
기간 가중 평균(각 설정값이 적용되었던 기간의 길이로 가중)하여 계산한 값입니다.

---

## 3. growth_record 조회

같은 `cultivationId`에 대해 AI Service 자체 DB의 `growth_record`에서 마지막(가장
최근) 생육 분석 결과의 `growthScore`를 조회합니다.

```sql
SELECT growth_score FROM growth_record
WHERE cultivation_id = ?
ORDER BY analyzed_at DESC
LIMIT 1;
```

---

## 4. insight 테이블 저장

조합한 데이터를 AI DB의 `insight` 테이블에 저장합니다. 각 행은 재배가 아닌 개별
수확(harvest, flush) 단위이며, 같은 재배의 서로 다른 flush는 각각 독립된 행으로
저장됩니다.

```sql
INSERT INTO insight
    (harvest_id, cultivation_id, flush_no, mushroom_type,
     avg_temperature, avg_humidity, avg_co2, avg_light,
     growth_score, harvest_weight, summary)
VALUES
    (41, 12, 1, 'OYSTER', 21.8, 89.2, 780.5, 360.0, 88, 3200,
     '느타리버섯, 평균 온도 21.8℃·습도 89% 환경에서 생육 점수 88점으로 3.2kg 수확 (1차 수확)');
```

`summary`(자연어 요약)는 이 단계에서 AI Service가 미리 만들어 함께 저장해 둡니다 —
조회 시점에 LLM 컨텍스트로 재사용합니다. `harvest_id`는 UNIQUE 제약으로 같은 이벤트가
중복 전달되어도 중복 적재되지 않도록 막습니다.

---

## 5. 인사이트 조회 요청

적재와 완전히 별개로, 사용자가 원할 때 호출합니다.

```http
GET /api/v1/ai/insight?cultivationId=27
```

---

## 6. 현재 재배 버섯 종류 + 환경 평균 조회 (Redis Cache Miss 시)

버섯 종류와 환경 평균 모두 Cultivation Service에서 조회합니다.

```http
GET /api/v1/cultivations/27
GET /api/v1/cultivations/27/environment-average
```

수확 목록 조회와 달리, 재배가 아직 `RUNNING` 상태여도(harvest가 없어도) 지금까지의
평균으로 조회할 수 있습니다.

---

## 7. insight 테이블 SQL 검색

```sql
SELECT * FROM insight
WHERE mushroom_type = 'OYSTER'
  AND avg_temperature BETWEEN 20.6 AND 23.6
  AND cultivation_id != 27
ORDER BY harvest_id DESC
LIMIT 10;
```

`mushroom_type`(정확히 일치) + `avg_temperature`(오차 범위) 조건이며, 코사인 유사도
기반 Vector Search가 아니라 인덱스(`idx_insight_mushroom_temp`)를 활용한 SQL
term/range 필터입니다.

---

## 8-A. 매칭 사례가 있는 경우

AI Service → LLM

입력: 현재 재배의 환경 평균 + 버섯 종류, 매칭된 유사 사례들의 환경 평균/생육 점수/수확량

```json
{
    "similarCaseCount": 8,
    "summary": "비슷한 온도(21.5~22.5℃)로 느타리버섯을 재배한 다른 사례 8건과 비교했을 때, 현재 재배의 생육 점수는 평균보다 다소 높은 편입니다."
}
```

결과를 Redis에 캐시합니다(`ai:{cultivationId}:insight`, TTL 24시간).

---

## 8-B. 매칭 사례가 없는 경우

LLM을 호출하지 않고 고정 문구를 사용합니다. 이 응답은 캐시하지 않습니다 — 다음 요청
시점에는 그 사이 새로운 수확이 기록되어 매칭될 수도 있기 때문입니다.

```json
{
    "similarCaseCount": 0,
    "summary": "아직 비교할 수 있는 유사 사례가 충분하지 않습니다."
}
```

---

# 사용 Database

## PostgreSQL

```
cultivation.mushroom_type (Cultivation DB, 조회 전용)
environment_setting (Cultivation DB, 조회 전용) — 환경 평균 계산의 원본
growth_record (AI DB, 조회 전용) — 생육 점수 조회용
insight (AI DB, 쓰기 + 조회) — 인사이트 사례
```

## Redis

```
ai:{cultivationId}:insight (TTL 24시간) — 매칭 사례가 없으면 캐시하지 않음
```

---

# OpenFeign

```
AI Service → Cultivation Service
  GET /cultivations/{cultivationId}
  GET /../environment-average
```

---

# RabbitMQ

```
AI Service → Subscribe: HarvestCompletedEvent (Cultivation Service 발행)
```

인사이트 사례 적재는 이 이벤트 구독으로 트리거되며, 조회는 사용자 요청에 대한 동기
응답이라 별도 이벤트가 필요하지 않습니다.

---

# 예외 상황

- 인사이트 적재 시 Cultivation Service 호출 실패 (환경 평균 조회 실패 — 해당 수확의
  insight 적재는 건너뛰며, 이벤트 유실/실패에 대비한 재처리는 추후 개발 예정)
- `insight` 테이블 저장 실패
- 인사이트 조회 시 Cultivation Service 호출 실패
- 인사이트 조회 시 매칭되는 유사 사례 없음 — LLM 미호출, 고정 안내 문구 반환 (오류
  아님)
- LLM 응답 실패

---

# 고려 사항

- 인사이트 사례 적재는 `HarvestCompletedEvent` 구독으로 트리거되며, 인사이트 조회만
  사용자가 직접 요청합니다.
- 환경 평균은 "기간 가중 평균"입니다. Cultivation Service가 `environment_setting` 이력을
  조회해 계산하며, 별도 컬럼에 저장하지 않고 조회 시점마다 계산합니다.
- 인사이트 조회는 진행 중인 재배(`RUNNING`)에 대해서도 가능합니다.
- 인사이트 "조회 결과"(LLM 요약)는 Redis에만 캐시하고 PostgreSQL에 영구 저장하지
  않습니다. 인사이트 "사례 데이터" 자체(`insight` 테이블)는 PostgreSQL에 영구
  저장됩니다 — 전자는 조회 시점의 요약 결과, 후자는 적재 시점에 쌓인 원본 사례입니다.
- 매칭 사례가 없는 응답은 캐시하지 않습니다.
- 검색 조건이 정확한 값 매칭(`mushroom_type`)과 범위 비교(`avg_temperature`)라서,
  별도의 벡터 검색 없이 인덱스가 걸린 SQL 조회로 충분합니다.
