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
2. **인사이트 조회** (사용자 요청 시점, on-demand): 적재와 완전히 별개로, 웹 챗봇에서
   `/인사이트` 명령어를 입력한 시점에만 검색이 이루어집니다. 일일 피드백/AI 리포트처럼
   미리 생성해두지 않습니다. 조회는 2단계입니다 — 먼저 조건에 맞는 유사 사례 후보를
   최대 5개 리스트로 보여주고, 사용자가 그중 하나를 선택하면 그 사례의 상세(날짜별 환경
   통계 + 일일 피드백, 수확일에는 인사이트 요약)를 보여줍니다.

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

## ②-A 인사이트 후보 조회

```text
Client (웹 챗봇, '/인사이트' 명령어 입력)
↓
AI Service (WebSocket)
↓
Redis 캐시 조회 (ai:{cultivationId}:insight:candidates)
├── Cache Hit → 즉시 반환
└── Cache Miss ↓
Cultivation Service OpenFeign 호출
  GET /cultivations/{cultivationId} (버섯 종류 조회)
  GET /../environment-average (환경 평균 조회)
  GET /cultivations/mine (내가 속한 cultivation_id 목록 조회, 제외용)
↓
insight 테이블 SQL 검색 (AI DB, mushroomType 정확히 일치 + 온습도/CO2/조도 4개 모두
오차 범위 + 내 cultivation_id 제외, 최신순 LIMIT 5)
├── 매칭 사례 있음 → Redis 캐시 저장 (TTL 24시간) → 후보 리스트 반환
└── 매칭 사례 없음 → 고정 안내 문구 반환 (캐시하지 않음)
↓
Client (후보 리스트, LLM 미호출)
```

---

## ②-B 인사이트 후보 상세 조회

```text
Client (후보 리스트 중 하나 선택, insightId 전달)
↓
AI Service (WebSocket)
↓
insight 테이블 조회 (선택한 건, cultivation_id + 기준일 확인)
↓
daily_feedback 테이블 조회 (AI DB, 같은 cultivation_id, feedback_date <= 기준일, 날짜순)
↓
날짜별 결과 조립
├── 수확일 이전 각 날짜 → 그날의 avg_temperature/humidity/co2/light + content 그대로 사용
└── 수확일 → daily_feedback 대신 insight.summary(적재 시점에 이미 생성된 한 줄 요약) 사용
↓
Client (날짜별 상세, 추가 LLM 호출 없음)
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

## 5. '/인사이트' 명령어 입력

적재와 완전히 별개로, 웹 챗봇에서 사용자가 원할 때 입력합니다(자세한 명령어 처리
흐름은 [ai-chat.md](./ai-chat.md) 참고).

```json
{ "message": "/인사이트", "cultivationId": 27 }
```

---

## 6. 현재 재배 버섯 종류 + 환경 평균 + 내 cultivation 목록 조회 (Redis Cache Miss 시)

```http
GET /api/v1/cultivations/27
GET /api/v1/cultivations/27/environment-average
GET /api/v1/cultivations/mine
```

`mine` 엔드포인트는 재배 멤버 기능과 함께 추가될 내부 API로, 요청자가 OWNER/MEMBER로
속한 모든 `cultivation_id`를 반환합니다 — 인사이트 후보에서 본인이 관리하는 재배를
제외하는 데 사용합니다. 수확 목록 조회와 달리, 재배가 아직 `RUNNING` 상태여도(harvest가
없어도) 지금까지의 평균으로 조회할 수 있습니다.

---

## 7. insight 테이블 SQL 검색 (후보 최대 5개)

```sql
SELECT * FROM insight
WHERE mushroom_type = 'OYSTER'
  AND avg_temperature BETWEEN 20.6 AND 23.6
  AND avg_humidity BETWEEN 84.2 AND 94.2
  AND avg_co2 BETWEEN 730 AND 830
  AND avg_light BETWEEN 340 AND 380
  AND cultivation_id NOT IN (12, 27, 31)  -- 내가 속한 cultivation_id 목록
ORDER BY created_at DESC
LIMIT 5;
```

`mushroom_type`(정확히 일치) + 온습도/CO2/조도 4개 항목(오차 범위) 조건이며, 코사인
유사도 기반 Vector Search가 아니라 인덱스(`idx_insight_mushroom_temp`)로 1차 필터링한
뒤 나머지 3개 항목은 추가 조건으로 거릅니다. 사례 수가 많지 않을 것으로 예상되어
별도 복합 인덱스는 두지 않습니다. 정렬 기준은 유사도가 아니라 `created_at`
최신순입니다.

---

## 8-A. 매칭 사례가 있는 경우

LLM을 호출하지 않고 검색된 후보 리스트를 그대로 반환합니다 — 이 단계는 "찾아서
보여주는" 목록 조회이며, 자연어 요약은 후보를 선택한 뒤(②-B)에만 이루어집니다.

```json
{
    "candidates": [
        { "insightId": 41, "mushroomType": "OYSTER", "growthScore": 88, "harvestWeight": 3200, "createdAt": "2026-08-10T09:00:00" },
        { "insightId": 39, "mushroomType": "OYSTER", "growthScore": 82, "harvestWeight": 2900, "createdAt": "2026-08-05T09:00:00" }
    ]
}
```

결과를 Redis에 캐시합니다(`ai:{cultivationId}:insight:candidates`, TTL 24시간).

---

## 8-B. 매칭 사례가 없는 경우

고정 문구를 사용합니다. 이 응답은 캐시하지 않습니다 — 다음 요청 시점에는 그 사이 새로운
수확이 기록되어 매칭될 수도 있기 때문입니다.

```json
{
    "candidates": [],
    "message": "아직 비교할 수 있는 유사 사례가 충분하지 않습니다."
}
```

---

## 9. 후보 선택 및 상세 조회

사용자가 후보 리스트 중 하나를 선택하면(`insightId` 전달), AI Service는 해당 사례의
`cultivation_id`로 `daily_feedback`을 조회합니다.

```sql
SELECT feedback_date, avg_temperature, avg_humidity, avg_co2, avg_light, content
FROM daily_feedback
WHERE cultivation_id = 12
  AND feedback_date <= '2026-08-10'  -- 선택한 insight 행의 created_at 날짜
ORDER BY feedback_date ASC;
```

추가 LLM 호출 없이, 조회된 각 날짜의 값을 그대로 클라이언트에 전달합니다. 다만 수확일
(위 쿼리의 상한일)에 해당하는 날짜만 그날의 `daily_feedback.content` 대신 `insight.summary`
(적재 시점에 이미 생성해 둔 한 줄 요약)로 교체합니다.

```json
{
    "days": [
        { "date": "2026-07-25", "avgTemperature": 21.5, "avgHumidity": 88.0, "avgCo2": 770, "avgLight": 355, "content": "환경이 안정적으로 유지되고 있습니다." },
        { "date": "2026-07-26", "avgTemperature": 22.0, "avgHumidity": 89.5, "avgCo2": 790, "avgLight": 360, "content": "..." },
        { "date": "2026-08-10", "avgTemperature": 21.8, "avgHumidity": 89.2, "avgCo2": 780.5, "avgLight": 360.0, "content": "느타리버섯, 평균 온도 21.8℃·습도 89% 환경에서 생육 점수 88점으로 3.2kg 수확 (1차 수확)" }
    ]
}
```

이 조회는 캐시하지 않습니다 — `cultivation_id` + 날짜 범위로 조회하는 가벼운 인덱스
조회라 매번 새로 조회해도 부담이 적고, 캐싱 대상으로 삼을 만큼 자주 반복 조회되는
패턴도 아닙니다.

---

# 사용 Database

## PostgreSQL

```
cultivation.mushroom_type (Cultivation DB, 조회 전용)
environment_setting (Cultivation DB, 조회 전용) — 환경 평균 계산의 원본
growth_record (AI DB, 조회 전용) — 생육 점수 조회용
insight (AI DB, 쓰기 + 조회) — 인사이트 사례
daily_feedback (AI DB, 조회 전용) — 후보 상세(날짜별 환경/피드백) 조회용
```

## Redis

```
ai:{cultivationId}:insight:candidates (TTL 24시간) — 매칭 사례가 없으면 캐시하지 않음
```

후보 상세(②-B)는 캐시하지 않습니다. `cultivation_id` + 날짜 범위 인덱스 조회로
충분히 가벼워, 캐싱으로 얻는 이득보다 관리 비용이 더 큽니다.

---

# OpenFeign

```
AI Service → Cultivation Service
  GET /cultivations/{cultivationId}
  GET /../environment-average
  GET /cultivations/mine (내가 속한 cultivation_id 목록, 후보 제외용)
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
- 후보 조회 시 Cultivation Service 호출 실패 (버섯 종류/환경 평균/내 cultivation 목록
  조회 실패)
- 후보 조회 시 매칭되는 유사 사례 없음 — 고정 안내 문구 반환 (오류 아님)
- 후보 상세 조회 시 존재하지 않거나 이미 만료된 `insightId` 선택

---

# 고려 사항

- 인사이트 사례 적재는 `HarvestCompletedEvent` 구독으로 트리거되며, 인사이트 조회만
  사용자가 직접 요청합니다.
- 환경 평균은 "기간 가중 평균"입니다. Cultivation Service가 `environment_setting` 이력을
  조회해 계산하며, 별도 컬럼에 저장하지 않고 조회 시점마다 계산합니다.
- 인사이트 조회는 진행 중인 재배(`RUNNING`)에 대해서도 가능합니다.
- 애초에 재배 기간 자체가 사용자마다 크게 달라 일자별(1일차/2일차...) 비교나 단시간
  스냅샷 비교는 유의미한 차이를 만들기 어렵다고 판단해 채택하지 않았습니다. 대신 재배
  전체 기간의 "기간 가중 평균"으로 유사 사례를 찾고, 사례 하나를 골라 그 사례가 실제로
  겪은 날짜별 기록을 그대로 보여주는 방식을 택했습니다.
- 후보 리스트는 LLM을 호출하지 않습니다. LLM이 생성한 자연어 요약(`insight.summary`)은
  이미 적재 시점에 만들어 저장해 둔 값을 후보 상세 조회 시 수확일 항목에 재사용할
  뿐이며, 조회 과정에서 추가로 LLM을 호출하지 않습니다.
- 매칭 사례가 없는 후보 리스트 응답은 캐시하지 않습니다 — 다음 요청 시점에는 새로운
  수확이 기록되어 매칭될 수 있기 때문입니다.
- 후보 검색 조건은 정확한 값 매칭(`mushroom_type`)과 4개 범위 비교(온습도/CO2/조도)로,
  별도의 벡터 검색 없이 인덱스가 걸린 SQL 조회로 충분합니다. 온도 외 3개 항목은
  전용 인덱스 없이 추가 조건으로 거르며, 사례 수가 늘어나 성능이 문제가 되면 복합
  인덱스 추가를 검토합니다.
- 후보 상세(②-B)는 캐시하지 않습니다 — `cultivation_id` + 날짜 범위의 가벼운 조회이며,
  다른 사용자의 과거 기록이라 조회 빈도도 낮을 것으로 예상됩니다.
