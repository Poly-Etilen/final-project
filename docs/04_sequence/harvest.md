# 수확 시퀀스

## 개요

사용자가 재배를 종료하고 수확 결과를 기록하는 과정입니다.

종료 전 사용자가 직접 촬영한 사진을 AI Vision 모델로 분석하여 생육 점수와 예상 수확 시기를 제공하며,
사용자가 실제 수확량과 메모를 입력하면 재배가 종료됩니다.

---

# Sequence

```text
Client (사진 촬영, 선택)

↓

API Gateway

↓

Cultivation Service

↓

MinIO 저장

↓

AI Service (image_url 전달)

↓

Vision 모델 분석

↓

생육 점수 계산

↓

LLM

↓

결과 해석 및 개선 방안 생성

↓

Cultivation Service

↓

Client

↓

수확 정보 입력

↓

Cultivation Service

↓

PostgreSQL 저장

↓

재배 상태 변경

↓

RabbitMQ

↓

Notification Service

↓

Client 알림
```

---

# 상세 과정

## 1. 생육 분석용 사진 업로드 (선택)

사용자는 수확 전 현재 생육 상태를 확인하기 위해 사진을 찍어 업로드할 수 있습니다.

예시

```http
POST /cultivations/{cultivationId}/photos
```

Cultivation Service는 사진을 MinIO에 저장하고 photo 메타데이터를 저장합니다.

---

## 2. AI Service 호출

Cultivation Service

↓

OpenFeign

↓

AI Service

전달 데이터

```json
{
    "cultivationId":3,
    "mushroomType":"OYSTER",
    "imageUrl":"https://minio/mushroom-photos/3/20260815-090000.jpg"
}
```

---

## 3. Vision 모델 분석

AI Service의 Vision 모델이 사진을 분석합니다.

Vision 모델은 사전에 학습된 성장 단계별 이미지 패턴과 비교하여 판단합니다.

분석 항목

- 균사 성장률
- 갓 크기
- 색상 분석
- 병충해 분석

출력

```json
{
    "myceliumGrowthRate":97,
    "capSize":"대(5.1cm)",
    "colorStatus":"정상",
    "diseaseStatus":"정상"
}
```

---

## 4. 생육 점수 계산

AI Service가 4가지 지표를 종합하여 점수를 계산합니다.

```
생육 점수 = 균사 성장률 40% + 갓 크기 점수 30% + 색상 점수 15% + 병충해 점수 15%
```

갓 크기가 수확 기준 크기 이상이면 성장 단계를 "수확 적기"로 판단합니다.

---

## 5. LLM 해석

AI Service

↓

LLM

입력

- 버섯 종류
- 생육 점수
- 균사 성장률 / 갓 크기 / 색상 / 병충해 여부
- 성장 단계

LLM은 위 지표를 새로 추정하지 않고, 해석 문장과 개선 방안만 생성합니다.

출력

```json
{
    "growthScore":94,
    "myceliumGrowthRate":97,
    "capSize":"대(5.1cm)",
    "colorStatus":"정상",
    "diseaseStatus":"정상",
    "growthStage":"수확 적기",
    "improvement":"현재 상태가 매우 양호하여 지금 수확하는 것을 권장합니다."
}
```

---

## 6. 분석 결과 반환

AI Service

↓

Cultivation Service

↓

Client

사용자는 생육 분석 결과를 참고하여 수확 시점을 결정합니다.

---

## 7. 수확 요청

사용자는

- 수확량
- 메모

를 입력합니다.

예시

```http
PATCH /api/v1/cultivations/{cultivationId}/finish
```

```json
{
    "harvestWeight":3200,
    "memo":"생육 상태 양호"
}
```

---

## 8. Harvest 저장

Cultivation Service는

harvest 테이블에 수확 정보를 저장합니다.

```
harvest
```

---

## 9. 재배 종료

Cultivation 상태 변경

```
RUNNING

↓

FINISHED
```

종료일을 저장합니다.

---

## 10. 응답

Client에게

```json
{
    "message":"Cultivation Finished"
}
```

전달

---

## 11. Event 발행

Cultivation Service

↓

RabbitMQ Publish

```json
{
    "cultivationId":3,
    "harvestWeight":3200,
    "finishedAt":"2026-08-15T09:00:00"
}
```

발행 이벤트

- CultivationFinishedEvent
- HarvestCompletedEvent

---

## 12. Notification Service

RabbitMQ Subscribe

↓

사용자에게 알림 전송

예시

```
🍄 수확 완료

느타리 1호기 재배가 종료되었습니다.
수확량: 3.2kg
```

---

# 사용 Database

## PostgreSQL

```
cultivation

harvest

photo
```

---

## MinIO

사용자가 업로드한 생육 사진 조회 (Cultivation Service가 저장, AI Service가 읽기 전용 조회)

---

# OpenFeign

```
Cultivation

↓

AI
```

---

# RabbitMQ

Publish

```
CultivationFinishedEvent
HarvestCompletedEvent
```

Subscribe

```
Notification Service
```

---

# 상태 변화

초기

```
RUNNING
```

↓

수확 정보 저장

↓

```
FINISHED
```

---

# 예외 상황

- 존재하지 않는 재배
- 이미 종료된 재배
- 권한 없는 재배 접근
- 사진 업로드 실패
- Vision 모델 분석 실패
- Harvest 저장 실패
- RabbitMQ 발행 실패
- Notification 전송 실패

---

# 고려 사항

- AI 생육 분석은 선택 사항이며, 건너뛰고 바로 수확 정보를 입력할 수도 있습니다.
- 생육 점수, 균사 성장률, 갓 크기, 색상, 병충해 여부, 성장 단계는 Vision 모델이 산출한 결과이며 LLM이 추정하지 않습니다.
- LLM은 계산된 지표를 해석하는 설명과 개선 방안 생성에만 사용합니다.
- 사진은 카메라 센서가 아닌 사용자가 직접 촬영하여 업로드합니다.
- 생육 분석 결과는 `ai:{cultivationId}:analysis` Redis 캐시(TTL 6시간, 빠른 재조회용)뿐 아니라 `growth_record` 테이블(PostgreSQL, AI DB)에도 영구 저장됩니다. 업로드된 사진(photo)은 별도로 Cultivation DB에 이력으로 보관됩니다. (자세한 내용은 [growth-analysis.md](./growth-analysis.md) 참고)
- 재배 종료와 수확 정보 저장은 하나의 API(PATCH /finish)로 함께 처리합니다.
- 종료된 재배는 재배 이력(GET /cultivations/history)에서 조회할 수 있습니다.
- Notification 전송 실패는 수확 처리 자체에 영향을 주지 않습니다.
