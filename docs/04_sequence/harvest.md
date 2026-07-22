# 수확 시퀀스

## 개요

사용자가 재배 중 수확 결과를 기록하는 과정과, 재배를 종료하는 과정입니다. "수확 기록
저장"과 "재배 종료"는 서로 다른 시점에 호출되는 별개의 API입니다.

- **수확 기록 저장** — 재배가 `RUNNING`인 동안 사용자가 원할 때마다 여러 번 반복할 수
  있습니다(같은 배지에서 1차/2차/3차로 여러 번 수확하는 "flush"를 표현). 재배 상태는
  바뀌지 않습니다.
- **재배 종료** — 더 이상 수확 정보를 받지 않으며, 재배 상태를 `FINISHED`로 바꾸는
  별도의 단순한 동작입니다.

---

# Sequence — ① 수확 기록 저장 (여러 번 가능)

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
flush_no 자동 채번 + harvest 행 저장
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
저장하고 `photo` 행(`object_key`, `storage_type`)을 남깁니다.

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
함께 산출해 `growth_record`(AI DB)에 영구 저장합니다. 동시에 Redis
(`ai:{cultivationId}:analysis`, TTL 6시간)에도 캐싱합니다.

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
POST /api/v1/cultivations/{cultivationId}/harvests
```

```json
{
    "harvestWeight": 1800,
    "memo": "1차 수확, 상태 양호"
}
```

몇 번째 수확인지(`flushNo`)는 사용자가 입력하지 않습니다.

---

## 7. Harvest 저장

Cultivation Service는 같은 `cultivation_id`의 기존 `harvest` 중 최댓값 + 1로 `flush_no`를
채번하고, `harvest` 행을 새로 저장합니다. 재배 상태는 바뀌지 않습니다.

---

## 8. 응답

```json
{
    "harvestId": 41,
    "flushNo": 1,
    "harvestWeight": 1800,
    "harvestedAt": "2026-08-20T09:00:00"
}
```

---

## 9. Event 발행

Cultivation Service → RabbitMQ Publish → `HarvestCompletedEvent`

---

## 10. Notification Service

RabbitMQ Subscribe → 등록된 채널로 알림 전송

```
🍄 수확 기록 완료
느타리 1호기 1차 수확이 기록되었습니다. 수확량: 1.8kg
```

---

# 상세 과정 — 재배 종료

## 1. 종료 요청

```http
PATCH /api/v1/cultivations/{cultivationId}/finish
```

## 2. 재배 종료 처리

Cultivation Service는 종료일을 저장하고 재배 상태를 `RUNNING → FINISHED`로 변경합니다.
`harvest` 행은 새로 생성하지 않으며, 지금까지 기록된 모든 flush가 그대로 수확 이력으로
남습니다.

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

```
🍄 재배 종료
느타리 1호기 재배가 종료되었습니다. 총 수확 2회 (합계 3.2kg)
```

---

# 사용 Database

## PostgreSQL

```
cultivation (Cultivation DB)
harvest (Cultivation DB)
photo (Cultivation DB)
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
↓ 수확 기록 저장 (여러 번 반복 가능, 상태 변화 없음)
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
- `flush_no`는 사용자가 지정하지 않고 Cultivation Service가 자동 채번합니다.
- 종료된 재배는 재배 이력(`GET /cultivations/history`)에서 합산값
  (`harvestCount`/`totalHarvestWeight`)으로 조회되며, 개별 수확 내역은
  `GET /cultivations/{cultivationId}/harvests`로 따로 조회합니다.
