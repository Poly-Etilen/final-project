# 수확 시퀀스

## 개요

사용자가 재배 중 수확 결과를 기록하는 과정과, 재배를 종료하는 과정입니다. "수확 기록
저장"과 "재배 종료"는 서로 다른 시점에 호출되는 별개의 API입니다.

- **수확 기록 저장** — 재배가 `RUNNING`인 동안 한 번만 기록할 수 있습니다. 병 재배를
  전제로 재배(병) 하나에서 자란 버섯은 한 번에 수확하기 때문입니다. 기록 자체는 재배
  상태를 바꾸지 않습니다.
- **재배 종료** — 더 이상 수확 정보를 받지 않으며, 재배 상태를 `FINISHED`로 바꾸는
  별도의 단순한 동작입니다.

---

# Sequence — ① 수확 기록 저장 (재배당 한 번)

```text
Client (사진 촬영, 선택)
↓
API Gateway
↓
Cultivation Service
↓
Photo Storage 저장 (MinIO 또는 Local)
↓
AI Service (objectKey/storageType 전달)
↓
Vision 모델 분석
↓
growth_record 저장 (AI DB)
↓
Cultivation Service
↓
Client (분석 결과 확인)
↓
수확 기록 요청
↓
Cultivation Service
↓
harvest 행 저장 (재배당 한 건, 이미 있으면 거부)
↓
RabbitMQ (HarvestCompletedEvent)
↓
Notification Service
↓
Client 알림
```

재배 상태는 `RUNNING`으로 그대로 유지됩니다.

---

# Sequence — ② 재배 종료 (별도 시점, 수확 정보 없음)

```text
Client
↓
API Gateway
↓
Cultivation Service
↓
재배 상태 변경 (RUNNING → FINISHED) + 종료일 저장
↓
RabbitMQ (CultivationFinishedEvent)
↓
Notification Service
↓
Client 알림
```

---

# 상세 과정 — 수확 기록 저장

## 1. 생육 분석용 사진 업로드 (선택)

```http
POST /api/v1/cultivations/{cultivationId}/photos
```

Cultivation Service는 사진을 Photo Storage(`storage_type`에 따라 MinIO 또는 로컬)에
저장하고 `cultivation_photo` 행(`object_key`, `storage_type`)을 남깁니다.

---

## 2. AI Service 호출

Cultivation Service → OpenFeign → AI Service (`POST /api/v1/ai/analysis`)

```json
{
    "cultivationId": 3,
    "photoId": 7,
    "objectKey": "3/20260815-090000.jpg",
    "storageType": "MINIO"
}
```

---

## 3. Vision 모델 분석

AI Service의 Vision 모델이 사진을 분석합니다.

분석 항목: 균사 성장률, 갓 크기, 색상, 병충해 여부

```json
{
    "myceliumGrowthRate": 68.5,
    "capSize": "MEDIUM",
    "color": "정상",
    "diseaseDetected": false
}
```

---

## 4. 생육 점수 계산 및 저장

AI Service가 4가지 지표를 종합해 `growthScore`를 계산하고, 성장 단계와 예상 수확일을
함께 산출해 `growth_record`(AI DB)에 영구 저장합니다. 지표들은 `analysis_data`(JSONB)
하나에 담기고, 사진은 스냅샷 복사 없이 `cultivation_photo_id`(소프트 참조)만
저장됩니다. 동시에 Redis(`ai:{cultivationId}:analysis`, TTL 6시간)에도 캐싱합니다.
성장 단계가 `수확적기`이면 Cultivation Service가 `cultivation.mode`를 자동으로
`HARVEST`로 전환합니다 — 자세한 내용은 [growth-analysis.md](./growth-analysis.md) 참고.

```json
{
    "growthRecordId": 55,
    "growthScore": 82,
    "growthStage": "생장기",
    "expectedHarvestDate": "2026-08-20"
}
```

---

## 5. 분석 결과 반환

AI Service → Cultivation Service → Client

사용자는 분석 결과를 참고해 수확 시점을 결정합니다.

---

## 6. 수확 기록 요청

```http
POST /api/v1/cultivations/{cultivationId}/harvest
```

```json
{
    "harvestWeight": 1800,
    "memo": "상태 양호"
}
```

---

## 7. Harvest 저장

Cultivation Service는 같은 `cultivation_id`에 이미 `harvest` 행이 있는지 확인합니다
(`UNIQUE(cultivation_id)`). 없으면 새로 저장하고, 이미 있으면 거부합니다. 재배 상태는
바뀌지 않습니다.

---

## 8. 응답

```json
{
    "harvestId": 41,
    "harvestWeight": 1800,
    "harvestedAt": "2026-08-20T09:00:00",
    "productScore": null,
    "productGrade": null
}
```

---

## 9. Event 발행

Cultivation Service → RabbitMQ Publish → `HarvestCompletedEvent`

Notification Service 외에 AI Service도 이 이벤트를 구독합니다 — 인사이트 사례 적재
([insight.md](./insight.md))와 상품 등급 원점수 계산
([product-grade.md](./product-grade.md))이 비동기로 트리거되며, 이 문서의 응답
시점에는 아직 반영되지 않습니다.

---

## 10. Notification Service

RabbitMQ Subscribe → `cultivationId`와 `HARVEST_COMPLETED` 구독 종류로 등록된 활성
구독을 조회 → 구독마다 연결된 채널(Telegram/Discord)로 알림 전송

```
🍄 수확 기록 완료
느타리 1호기 수확이 기록되었습니다. 수확량: 1.8kg
```

---

# 상세 과정 — 재배 종료

## 1. 종료 요청

```http
PATCH /api/v1/cultivations/{cultivationId}/finish
```

## 2. 재배 종료 처리

Cultivation Service는 종료일을 저장하고 재배 상태를 `RUNNING → FINISHED`로 변경합니다.
`harvest` 행은 새로 생성하지 않으며, 이미 기록된 수확(있다면)이 그대로 수확 이력으로
남습니다. 수확 기록 없이 종료하는 것도 허용합니다.

## 3. 응답

```json
{
    "cultivationId": 3,
    "status": "FINISHED",
    "finishedAt": "2026-09-01T10:00:00"
}
```

## 4. Event 발행

Cultivation Service → RabbitMQ Publish → `CultivationFinishedEvent`

## 5. Notification Service

RabbitMQ Subscribe → `cultivationId`와 `CULTIVATION_FINISHED` 구독 종류로 등록된
활성 구독을 조회 → 구독마다 연결된 채널(Telegram/Discord)로 알림 전송

```
🍄 재배 종료
느타리 1호기 재배가 종료되었습니다. 수확량: 1.8kg
```

---

# 사용 Database

## PostgreSQL

```
cultivation (Cultivation DB)
harvest (Cultivation DB)
cultivation_photo (Cultivation DB)
growth_record (AI DB)
```

## Photo Storage

사용자가 업로드한 생육 사진(Cultivation Service가 저장, AI Service가 읽기 전용 조회)

---

# OpenFeign

```
Cultivation Service → AI Service (생육 분석 요청)
```

수확 기록 흐름에서만 호출합니다. 재배 종료 흐름은 다른 서비스를 동기 호출하지 않습니다.

---

# RabbitMQ

Publish

```
HarvestCompletedEvent (수확 기록마다)
CultivationFinishedEvent (재배 종료 시 한 번)
```

Subscribe

```
Notification Service
```

---

# 상태 변화

```
RUNNING
↓ 수확 기록 저장 (재배당 한 번, 상태 변화 없음)
↓ 재배 종료 요청
FINISHED
```

---

# 예외 상황

- 존재하지 않는 재배 / 권한 없는 재배 접근
- `FINISHED` 상태 재배에 수확 기록/사진 업로드 시도
- 이미 종료된 재배에 다시 종료 요청
- 사진 저장소 업로드 실패
- Vision 모델 분석 실패
- RabbitMQ 발행 실패 / Notification 전송 실패 (수확 기록/종료 자체에는 영향 없음)

---

# 고려 사항

- AI 생육 분석은 선택 사항이며, 건너뛰고 바로 수확 기록을 남길 수도 있습니다.
- `growthScore`/`myceliumGrowthRate`/`capSize`/`color`/`diseaseDetected`/`growthStage`는
  Vision 모델이 산출한 결과입니다.
- `harvest`는 `cultivation`당 최대 한 건입니다(`UNIQUE(cultivation_id)`). 이미 기록된
  재배에 다시 요청하면 거부됩니다.
- 종료된 재배는 재배 이력(`GET /cultivations/history`)에서 `harvestWeight`로 함께
  조회되며, 필요하면 `GET /cultivations/{cultivationId}/harvest`로 단건 조회할 수
  있습니다.
- `productScore`/`productGrade`는 이 응답 시점에는 항상 NULL입니다. `HarvestCompletedEvent`를
  구독한 AI Service가 비동기로 계산해 돌려준 뒤에야 채워집니다. 자세한 내용은
  [product-grade.md](./product-grade.md) 참고.
