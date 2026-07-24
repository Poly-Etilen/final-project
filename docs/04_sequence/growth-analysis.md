# 생육 분석 시퀀스

## 개요

사용자가 재배 중인 버섯을 직접 촬영해 업로드하면, AI가 Vision 모델로 사진을 분석해
생육 상태를 판단하는 과정입니다. 카메라 센서가 자동으로 촬영하는 방식이 아니라, 사용자가
앱/웹에서 사진을 찍어 업로드하는 방식입니다.

Vision 모델은 사전에 학습된 성장 단계별 이미지 패턴과 업로드된 사진을 비교해 균사
성장률/갓 크기/색상/병충해 여부를 판단합니다. LLM은 이 지표를 새로 추정하지 않고, 해석
문장과 개선 방안만 생성합니다.

[harvest.md](./harvest.md)의 수확 전 생육 분석과 동일한 로직을 사용하지만, 이 시퀀스는
수확 여부와 관계없이 재배 상세 화면에서 사용자가 원할 때마다 실행할 수 있습니다.

분석 결과의 성장 단계(`growthStage`, `growth_record.analysis_data->>'growthStage'`에
저장됨)가 `수확적기`이면, Cultivation Service는 이 응답을 받은 같은 요청 처리 안에서
`cultivation.mode`를 `GROWTH → HARVEST`로 자동 전환합니다. 사용자가 직접 조작하지
않는 시스템 판단이며, 이 흐름이 재배당 한 번뿐인 수확 시점을 사용자에게 미리
안내하는 유일한 트리거입니다.

---

# Sequence

```text
Client (사진 촬영)
↓
API Gateway
↓
Cultivation Service
↓
Photo Storage 저장 (MinIO 또는 Local)
↓
cultivation_photo 메타데이터 저장 (PostgreSQL, Cultivation DB)
↓
AI Service (objectKey/storageType 전달)
↓
Vision 모델 분석
↓
생육 점수 계산
↓
LLM 해석
↓
Redis 저장 + growth_record 저장 (PostgreSQL, AI DB)
↓
Cultivation Service
└── growthStage가 "수확적기"이면 cultivation.mode = HARVEST로 전환
    (전환 시 CultivationModeChangedEvent 발행)
↓
Client
```

---

# 상세 과정

## 1. 사진 업로드 요청

```http
POST /api/v1/cultivations/{cultivationId}/photos
```

Multipart Form으로 이미지를 전달합니다.

---

## 2. Photo Storage 저장

Cultivation Service → Photo Storage(`storage_type` 설정에 따라 MinIO 또는 로컬) 저장

저장 후 `object_key`(예: `3/20260815-090000.jpg`)를 발급받습니다. 전체 URL이 아닌
저장소 내 상대 경로만 발급받으며, 실제 접근 URL/경로 변환은 조회 시점에 서비스 설정
기준으로 계산합니다.

---

## 3. cultivation_photo 메타데이터 저장

`cultivation_photo` 테이블(Cultivation DB)에 저장합니다.

```json
{
    "cultivationId": 3,
    "objectKey": "3/20260815-090000.jpg",
    "storageType": "MINIO",
    "uploadedAt": "2026-08-15T09:00:00"
}
```

---

## 4. AI Service 호출

Cultivation Service → OpenFeign → AI Service (`POST /api/v1/ai/analysis`)

```json
{
    "cultivationId": 3,
    "photoId": 7,
    "objectKey": "3/20260815-090000.jpg",
    "storageType": "MINIO"
}
```

`objectKey`/`storageType`은 AI Service가 Vision 분석을 위해 Photo Storage에서 사진을
읽어오는 데만 사용하는 값이며, `growth_record`에는 저장하지 않습니다. `growth_record`에는
`photoId`가 `cultivation_photo_id`로 저장됩니다(아래 8번 참고).

---

## 5. Vision 모델 분석

AI Service의 Vision 모델이 `objectKey`로 Photo Storage에서 사진을 조회해 분석합니다.

분석 항목: 균사 성장률, 갓 크기, 색상, 병충해 여부

```json
{
    "myceliumGrowthRate": 82.0,
    "capSize": "MEDIUM",
    "color": "정상",
    "diseaseDetected": false
}
```

---

## 6. 생육 점수 계산

AI Service가 4가지 지표를 종합해 점수를 계산합니다.

```
생육 점수 = 균사 성장률 40% + 갓 크기 점수 30% + 색상 점수 15% + 병충해 점수 15%
```

병충해가 감지된 경우 별도로 감점합니다. 성장 단계(균사기/자실체 형성기/생장기/수확
적기)를 기준으로 예상 수확일을 추정합니다.

---

## 7. LLM 해석

AI Service → LLM

입력: 버섯 종류, 생육 점수, 균사 성장률/갓 크기/색상/병충해 여부, 성장 단계

LLM은 위 지표를 새로 추정하지 않고 해석 문장과 개선 방안만 생성합니다.

```json
{
    "growthScore": 85,
    "growthStage": "자실체 형성기",
    "expectedHarvestDate": "2026-08-20",
    "improvement": "균사 성장은 양호하나 조도가 다소 낮게 유지되고 있어 갓 형성이 더딜 수 있습니다. LED 점등 시간을 늘려보세요."
}
```

---

## 8. Redis 저장 + growth_record 저장

Redis: `ai:{cultivationId}:analysis`, TTL 6시간 — 이후 같은 결과의 재조회는 즉시 반환됩니다.

PostgreSQL(AI DB): 이와 별개로 분석 결과를 `growth_record`에 한 행으로 영구 저장합니다.
`growthScore`~`expectedHarvestDate`는 컬럼별로 나누지 않고 `analysis_data`(JSONB)
하나에 담기고, 사진 자체는 스냅샷 복사 없이 `cultivation_photo_id`(=4번 단계의
`photoId`)만 소프트 참조로 저장합니다.

```sql
INSERT INTO growth_record (cultivation_id, cultivation_photo_id, analysis_data, analyzed_at)
VALUES (
    3, 7,
    '{
        "growthScore": 85,
        "myceliumGrowthRate": 82.0,
        "capSize": "중(3.2cm)",
        "colorStatus": "정상",
        "diseaseStatus": "정상",
        "growthStage": "자실체 형성기",
        "expectedHarvestDate": "2026-08-20"
    }'::jsonb,
    '2026-08-15T09:00:05'
);
```

Redis 캐시는 TTL이 지나면 사라지지만 `growth_record`는 만료되지 않고 계속 쌓여, 일일
피드백([daily-feedback.md](./daily-feedback.md))이 여러 날짜의 생육 추이를 비교하는 데
사용됩니다.

---

## 9. 결과 반환

AI Service → Cultivation Service → Client

사용자는 방금 찍은 사진에 대한 생육 점수와 분석 결과를 즉시 확인합니다.

---

## 10. 모드 전환 (조건부)

Cultivation Service는 AI Service로부터 받은 분석 결과(7번에서 LLM이 산출하고
`growth_record.analysis_data->>'growthStage'`로 저장된 `growthStage`)가 `수확적기`이고
현재 `cultivation.mode`가 아직 `GROWTH`이면, 9번에서
Client에 응답을 반환하기 전(같은 요청 처리, 같은 트랜잭션 안)에 `mode`를 `HARVEST`로
바꾸고 `CultivationModeChangedEvent`를 발행합니다. 그 외의 `growthStage`(균사기/
자실체 형성기/생장기)에서는 아무 것도 하지 않습니다.

```json
{ "cultivationId": 3, "mode": "HARVEST" }
```

재배당 수확이 한 번뿐이라 이 전환은 되돌아가지 않습니다 — 한 번 `HARVEST`가 되면
그 재배가 종료될 때까지 유지됩니다.

---

# 사용 Database

## Photo Storage

사용자가 업로드한 사진 저장 (Cultivation Service가 저장, AI Service가 읽기 전용 조회)

## PostgreSQL

```
cultivation_photo (Cultivation DB)
growth_record (AI DB)
cultivation (Cultivation DB, mode 컬럼 — growthStage가 "수확적기"일 때만 갱신)
```

## Redis

AI 생육 분석 결과 캐시 (`ai:{cultivationId}:analysis`, TTL 6시간)

---

# OpenFeign

```
Cultivation Service → AI Service
```

---

# RabbitMQ

Publish

```
CultivationModeChangedEvent (growthStage가 "수확적기"로 판정되어 mode가
GROWTH → HARVEST로 바뀔 때만, Cultivation Service 발행)
```

사진 업로드와 생육 분석 자체는 사용자 요청 기반의 동기 처리이며, 이 이벤트는
그중 모드가 실제로 바뀐 경우에만 예외적으로 발행됩니다.

Subscribe

```
Notification Service (CultivationModeChangedEvent, 수확 임박 알림)
```

Notification Service는 이벤트에 실린 `cultivationId`와 `CULTIVATION_MODE_CHANGED`
구독 종류로 등록된 활성 구독을 조회해, 구독마다 연결된 채널(Telegram/Discord)로
알림을 전송합니다.

---

# 예외 상황

- 존재하지 않는 재배 / 이미 종료된 재배
- 지원하지 않는 파일 형식
- Photo Storage 저장 실패
- Vision 모델 분석 실패 / LLM 응답 실패
- Redis 장애
- `growth_record` 저장 실패 (저장에 실패해도 분석 응답 자체는 사용자에게 반환)
- `CultivationModeChangedEvent` 발행 실패 (알림만 지연될 뿐, `mode` 전환 자체는 같은
  트랜잭션에서 이미 반영되어 영향받지 않음)

---

# 고려 사항

- 생육 점수, 균사 성장률, 갓 크기, 색상, 병충해 여부, 성장 단계는 Vision 모델이 산출한
  결과이며 LLM이 새로 추정하지 않습니다. LLM은 해석 문장과 개선 방안 생성에만 사용합니다.
- 사진은 카메라 센서가 아닌 사용자가 앱/웹에서 직접 촬영해 업로드합니다.
- 사진 업로드와 분석은 하나의 요청(`POST .../photos`)으로 함께 처리됩니다.
- 분석 결과는 Redis(빠른 재조회용, TTL 6시간)와 `growth_record`(영구 보관, 일일 피드백용)
  두 곳에 함께 저장됩니다. 사진 자체(`cultivation_photo`)는 Cultivation DB에 이력으로
  계속 보관되며, `growth_record`는 그 ID(`cultivation_photo_id`)만 소프트 참조로
  저장합니다 — 분석 결과 조회 시 사진 원본이 필요하면 Cultivation Service에 별도
  조회가 필요합니다.
- `growth_record.analysis_data`는 JSONB 하나에 지표를 모두 담으므로, 지표 구성이
  바뀌어도(필드 추가/제거) 스키마 마이그레이션 없이 유연하게 대응할 수 있습니다.
- `mode` 전환은 이 시퀀스(생육 분석)에서만 트리거됩니다. 사용자가 직접 모드를
  바꾸는 API는 없습니다 — 시스템이 Vision 분석 결과만으로 자동 판단합니다.
- `mode`가 `HARVEST`로 바뀌어도 `environment_setting`(목표 환경 범위)은 자동으로
  바뀌지 않습니다. 수확기 환경 임계값 데이터를 아직 확보하지 못해서이며, 데이터가
  준비되면 자동 전환 로직을 추후 추가할 예정입니다.
