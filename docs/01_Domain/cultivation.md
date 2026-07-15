# Cultivation Service

## 역할

Cultivation Service는 사용자의 버섯 재배 정보를 관리하는 핵심 서비스입니다.

사용자는 새로운 재배를 생성하고, 공공데이터 기반으로 추천된 재배 환경을 수정하여 저장할 수 있으며,
재배 진행 상황과 수확 결과를 관리할 수 있습니다.

---

# 책임

- 재배 생성
- 재배 조회
- 재배 수정
- 재배 삭제
- 환경 설정 저장
- 재배 종료
- 수확 정보 저장
- 재배 이력 관리
- 생육 사진 업로드
- 생육 사진 이력 관리

---

# 주요 기능

## 재배 생성

새로운 버섯 재배를 생성합니다.

사용자는

- 재배 이름
- 버섯 종류

를 입력합니다.

이후 자체 보유한 `mushroom_reference` 참조 테이블(공공데이터 기반 5종 버섯의 최적 환경 범위)을
버섯 종류로 조회하여 추천값을 반환합니다. AI Service를 호출하지 않습니다.

> ℹ️ **변경 이력**: 원래는 AI Service(Embedding/Vector Search/LLM)를 호출해 추천값을 생성했지만,
> 버섯 종류가 공공데이터 기준 5가지로 고정되어 있어 항상 동일한 값이 나오는 조회에는 AI가
> 불필요하다고 판단, Cultivation Service가 직접 보유한 참조 테이블 조회로 단순화했습니다.

사용자가 추천 환경을 수정한 후 저장하면
최종 환경 설정이 Database에 저장됩니다. 이 저장값(environment_setting)은 mushroom_reference의
"최적 범위"와는 별개로, 사용자가 직접 정하는 자동 제어용 "위험 한계값"입니다.

API로 주고받는 목표값은 단일값(예: 온도 22℃)이지만, Database에는 허용 오차를 적용한
범위(예: 20.5~23.5℃)로 저장됩니다. Rule Engine Service가 값이 범위를 벗어날 때만
장치를 제어하도록 하여 불필요한 On/Off를 줄이기 위함입니다. (자세한 변환 기준은 cultivation-db.md 참고)

---

## 재배 목록 조회

사용자가 생성한 모든 재배 목록을 조회합니다.

조회 정보

- 재배 이름
- 버섯 종류
- 현재 상태
- 생성일

---

## 재배 상세 조회

재배의 상세 정보를 조회합니다.

조회 정보

- 환경 설정
- 센서 상태
- 생육 상태
- 생성일
- 수정일

---

## 재배 환경 수정

사용자는 현재 재배 환경을 수정할 수 있습니다.

수정 항목

- 목표 온도
- 목표 습도
- 목표 CO₂
- 목표 조도

---

## 재배 종료

재배를 종료합니다.

종료 시

- 종료일 저장
- 상태 변경
- 수확 정보 입력

---

## 수확 기록 저장

재배 종료 후

- 수확량
- 메모

를 저장합니다.

---

## 재배 이력 조회

완료된 재배 목록을 조회합니다.

---

## 생육 사진 업로드

사용자가 직접 촬영한 사진을 업로드하여 생육 상태를 분석합니다.

카메라 센서가 아닌, 사용자가 앱/웹에서 직접 촬영한 사진을 업로드하는 방식입니다.

사진은 MinIO에 저장되고, Cultivation Service는 AI Service를 호출하여
학습된 Vision 모델로 사진을 분석한 결과(균사 성장률, 갓 크기, 색상, 병충해 여부)를 함께 반환합니다.

분석 결과는 즉시 반환되며, 최근 분석 결과는 캐시되어 재조회할 수 있습니다.

---

## 생육 사진 이력 조회

재배별로 업로드된 사진 목록을 조회합니다.

---

# API

## 재배 생성

POST /cultivations

---

## 재배 목록 조회

GET /cultivations

---

## 재배 상세 조회

GET /cultivations/{cultivationId}

---

## 재배 수정

PATCH /cultivations/{cultivationId}

---

## 재배 삭제

DELETE /cultivations/{cultivationId}

---

## 환경 설정 저장

PATCH /cultivations/{cultivationId}/environment

---

## 재배 종료

PATCH /cultivations/{cultivationId}/finish

---

## 수확 정보 저장

PATCH /cultivations/{cultivationId}/harvest

---

## 재배 이력 조회

GET /cultivations/history

---

## 생육 사진 업로드 (+분석)

POST /cultivations/{cultivationId}/photos

사진을 업로드하면 동시에 AI Vision 분석을 수행하여 결과를 함께 반환합니다.

---

## 생육 사진 이력 조회

GET /cultivations/{cultivationId}/photos

---

## 최근 생육 분석 결과 조회

GET /cultivations/{cultivationId}/analysis

새로 분석을 수행하지 않고, 가장 최근 분석 결과(캐시)를 반환합니다.

---

# Database

Cultivation Service는 별도의 PostgreSQL Database를 사용합니다.

### Table

- mushroom_reference (공공데이터 기반 5종 최적 환경 범위, 전역 참조 데이터)
- cultivation
- environment_setting
- harvest
- photo

---

# Redis

현재 사용하지 않습니다.

---

# MinIO

사용자가 업로드한 생육 사진 원본을 저장합니다.

Bucket

```
mushroom-photos
```

Key 구조

```
{cultivationId}/{uploadedAt}.jpg
```

---

# 다른 서비스와의 통신

## 호출하는 서비스

### AI Service

생육 사진 Vision 분석 요청

AI 리포트 조회

환경 추천은 더 이상 AI Service를 호출하지 않고, Cultivation Service가 mushroom_reference를 직접 조회합니다.

---

### Sensor Service

현재 센서 상태 조회

환경 통계 조회

실시간 센서 데이터 조회

---

## 호출받는 서비스

API Gateway

### Rule Engine Service

규칙 평가 시 목표 환경 범위(environment_setting의 min~max) 조회 (Rule Engine Service의 Redis 캐시가 없을 때만 호출되는 fallback)

---

# Event

## 발행 이벤트

### CultivationCreatedEvent

재배 생성 완료

---

### CultivationFinishedEvent

재배 종료

---

### HarvestCompletedEvent

수확 완료

---

### EnvironmentRangeUpdatedEvent

environment_setting을 생성/수정(저장)할 때 발행합니다. 단일 목표값을 범위(min~max)로 변환한 값을 담습니다.

```json
{
    "cultivationId": 3,
    "tempMin": 20.5,
    "tempMax": 23.5,
    "humidityMin": 85,
    "humidityMax": 95,
    "co2Min": 750,
    "co2Max": 850,
    "lightMin": 320,
    "lightMax": 380,
    "updatedAt": "2026-08-15T09:00:00"
}
```

구독 서비스: Rule Engine Service (Redis 캐시 write-through 갱신용)

---

# Sequence

## 재배 생성

Client

↓

Gateway

↓

Cultivation Service

↓

mushroom_reference 조회 (버섯 종류 기준)

↓

환경 추천 반환

↓

Client

↓

사용자 수정

↓

Cultivation Service

↓

PostgreSQL 저장

---

## 재배 종료

Client

↓

Gateway

↓

Cultivation Service

↓

Harvest 저장

↓

재배 종료

↓

Event 발행

---

# 예외 상황

- 존재하지 않는 재배
- 이미 종료된 재배
- 권한 없는 재배 접근
- 지원하지 않는 버섯 종류 (mushroom_reference에 없음)
- 환경 설정 저장 실패
- 사진 업로드 실패
- 지원하지 않는 파일 형식
- Vision 분석 실패

---

# 추후 개발 예정

- 재배 템플릿 저장
- 즐겨찾는 환경 저장
- 자동 재배 스케줄 설정
- 재배 목표 설정