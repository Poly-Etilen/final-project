# 상품 등급 시퀀스

## 개요

수확이 기록되면 그 재배의 생육 점수 평균과 환경 유지 정도를 종합해 상품
등급(TOP/HIGH/MID/LOW)을 매기는 기능입니다. 인사이트 적재와 마찬가지로
`HarvestCompletedEvent`를 트리거로 사용하지만, 두 처리는 서로 독립적입니다 — 상품
등급 계산이 실패해도 인사이트 적재에는 영향이 없고, 그 반대도 마찬가지입니다.

점수 계산 자체는 AI Service가 담당하지만, 등급 구간(몇 점 이상이 TOP인지 등)은
Cultivation Service가 소유합니다. AI Service는 생육 점수 평균과 환경 유지 점수를
합산한 원점수(`productScore`, 0~100) 하나만 계산해 Cultivation Service에 전달하고,
그 원점수를 등급으로 매핑해 `harvest` 행에 저장하는 것은 Cultivation Service의
역할입니다 — 등급 정책이 바뀌어도 AI Service 쪽 로직을 건드릴 필요가 없도록 하기
위함입니다.

---

# Sequence

```text
Cultivation Service (수확 기록) → HarvestCompletedEvent 발행
↓
AI Service (구독)
↓
growth_record 평균 조회 (자체 DB, AI DB — 호출 없음)
↓
Cultivation Service OpenFeign 호출
  GET /cultivations/{cultivationId}/environment-compliance
  (측정 항목별로 추천 범위 안에 있었던 시간 비율 집계)
↓
원점수(productScore) 계산 = (생육 점수 평균 + 환경 유지 점수) / 2
↓
Cultivation Service OpenFeign 호출
  PATCH /cultivations/{cultivationId}/harvest/product-score
↓
Cultivation Service: 원점수 → 등급 매핑 후 harvest 행에 저장
  (product_score, product_grade)
```

---

# 상세 과정

## 1. HarvestCompletedEvent 구독

Cultivation Service가 수확을 기록하면(재배당 한 번) 발행하는 이벤트를 AI Service가
구독합니다. 인사이트 적재와 같은 이벤트를 공유하지만, 처리 로직은 독립적입니다(한쪽
실패가 다른 쪽에 영향을 주지 않음).

```json
{
    "harvestId": 41,
    "cultivationId": 12,
    "harvestWeight": 3200
}
```

---

## 2. growth_record 평균 조회

같은 `cultivationId`의 모든 생육 분석 결과 평균을 자체 DB(AI DB)에서 계산합니다.
`growth_score`는 `analysis_data`(JSONB) 안에 있으므로 JSONB 연산자로 꺼내 평균을
계산합니다.

```sql
SELECT AVG((analysis_data->>'growthScore')::int) AS growth_score_avg
FROM growth_record
WHERE cultivation_id = 12;
```

```json
{ "growthScoreAvg": 85.0 }
```

---

## 3. 환경 준수율 집계 조회

이벤트에 담긴 `cultivationId`로 Cultivation Service에 환경 준수율을 조회합니다.
InfluxDB(측정값 원본)는 Cultivation Service만 접근할 수 있어, AI Service는 원시
시계열이 아니라 이미 집계된 비율만 전달받습니다.

```http
GET /api/v1/cultivations/12/environment-compliance
```

```json
{
    "temperatureCompliance": 92.5,
    "humidityCompliance": 88.0,
    "co2Compliance": 95.0,
    "lightCompliance": 90.0
}
```

각 값은 재배 시작부터 수확까지 전체 기간 중, 해당 측정 항목의 실측값이
`mushroom_reference_threshold`의 추천 범위(`threshold_min`~`threshold_max`) 안에
있었던 시간의 비율(%)입니다. Cultivation Service가 InfluxDB 측정값을 조회해 계산하며,
별도 컬럼에 저장하지 않고 요청 시점마다 계산합니다.

---

## 4. 환경 유지 점수 및 원점수 계산

4개 항목 준수율의 평균을 환경 유지 점수로 삼고, 생육 점수 평균과 단순 평균해
원점수를 계산합니다.

```
환경 유지 점수 = (92.5 + 88.0 + 95.0 + 90.0) / 4 = 91.375
productScore   = (85.0 + 91.375) / 2 = 88.2
```

---

## 5. 원점수 전달

계산한 원점수 하나만 Cultivation Service에 전달합니다 — 생육 점수 평균과 환경 유지
점수를 각각 넘기지 않습니다.

```http
PATCH /api/v1/cultivations/12/harvest/product-score
```

```json
{ "productScore": 88.2 }
```

---

## 6. 등급 매핑 및 저장

Cultivation Service는 전달받은 원점수를 아래 구간으로 매핑해 `harvest` 행의
`product_score`/`product_grade`에 저장합니다.

| product_score | product_grade |
|----------------|----------------|
| 95점 이상 | TOP (최상) |
| 80점 이상 | HIGH (상) |
| 60점 이상 | MID (중) |
| 60점 미만 | LOW (하) |

```json
{
    "harvestId": 41,
    "productScore": 88.2,
    "productGrade": "HIGH"
}
```

---

# 사용 Database

## PostgreSQL

```
growth_record (AI DB, 조회 전용) — 생육 점수 평균 계산용
harvest (Cultivation DB, 쓰기) — product_score/product_grade 저장
```

## InfluxDB

```
environment (Cultivation DB, 조회 전용) — 환경 준수율 집계의 원본. AI Service는
직접 접근하지 않고, Cultivation Service가 집계한 결과만 전달받습니다.
```

---

# OpenFeign

```
AI Service → Cultivation Service
  GET /cultivations/{cultivationId}/environment-compliance
  PATCH /cultivations/{cultivationId}/harvest/product-score
```

---

# RabbitMQ

```
AI Service → Subscribe: HarvestCompletedEvent (Cultivation Service 발행)
```

인사이트 적재와 같은 이벤트를 공유하되, 두 처리는 서로 독립적으로 실행됩니다.

---

# 예외 상황

- 환경 준수율 조회(`GET .../environment-compliance`) 실패 시 상품 등급 계산 자체를
  건너뜁니다. `harvest.product_score`/`product_grade`는 NULL로 남으며, 재처리는
  추후 개발 예정입니다.
- 원점수 전달(`PATCH .../harvest/product-score`) 실패 시에도 마찬가지로 NULL로
  남습니다.
- 생육 사진을 한 번도 찍지 않고 수확한 재배는 `growth_record`가 없어 생육 점수
  평균을 계산할 수 없습니다 — 이 경우 상품 등급 계산 자체를 건너뜁니다. 환경 유지
  점수만으로 등급을 매길지 여부는 추후 결정합니다.
- 0~100 범위를 벗어난 원점수가 전달되면 Cultivation Service가 거부합니다.

---

# 고려 사항

- 원점수는 생육 점수 평균과 환경 유지 점수를 단순 평균(50:50)으로 우선 설계했습니다.
  가중치 조정이 필요하면 추후 튜닝합니다.
- 인사이트 적재와 상품 등급 계산은 같은 `HarvestCompletedEvent`를 구독하지만 서로
  독립적으로 처리됩니다 — 하나가 실패해도 다른 하나는 계속 진행합니다.
- 등급 구간(95/80/60점 컷)과 매핑 로직은 Cultivation Service가 소유합니다. AI
  Service는 등급 정책을 알 필요 없이 원점수 계산만 담당해, 등급 기준이 바뀌어도 AI
  Service 쪽 로직은 영향받지 않습니다.
- 환경 준수율 집계는 재배 시작부터 수확까지 전체 기간 기준입니다. 생육 환경과 수확
  환경을 구분해 다르게 반영할지는 아직 논의되지 않았습니다.
