# AI 리포트 생성 시퀀스

## 개요

AI 리포트는 재배 기간 동안 수집된 환경 데이터를 분석하여
사용자에게 주간 또는 월간 재배 리포트를 제공합니다.

Rule Engine Service는 InfluxDB에서 통계 데이터를 조회하여
AI Service에 전달하고,
AI Service는 LLM을 이용하여 자연어 리포트를 생성합니다.

---

# Sequence

```text
Client

↓

API Gateway

↓

AI Service

↓

Rule Engine Service

↓

InfluxDB

↓

환경 데이터 집계

↓

AI Service

↓

LLM

↓

리포트 생성

↓

Redis Cache 저장

↓

Client
```

---

# 상세 과정

## 1. 리포트 요청

사용자가

- 주간 리포트
- 월간 리포트

를 요청합니다.

예시

```http
GET /api/v1/report/weekly/{cultivationId}
```

---

## 2. Redis 조회

AI Service는 먼저 Redis를 조회합니다.

Key

```
report:{cultivationId}:weekly
```

↓

존재하면

↓

즉시 반환

---

## 3. Cache Miss

Redis에 없으면

↓

Rule Engine Service 호출

(OpenFeign)

---

## 4. 환경 데이터 조회

Rule Engine Service

↓

InfluxDB

↓

기간별 집계

예시

```
최근 7일
```

조회 항목

- 평균 온도
- 평균 습도
- 평균 CO₂
- 평균 조도
- 최고 온도
- 최저 온도
- 최고 습도
- 최저 습도

---

## 5. 통계 반환

Rule Engine Service

↓

AI Service

예시

```json
{
  "averageTemperature":22.1,
  "averageHumidity":90.3,
  "averageCo2":810,
  "averageLight":430,
  "maxTemperature":24.5,
  "minTemperature":20.2
}
```

---

## 6. LLM 요청

AI Service

↓

LLM

입력

- 버섯 종류
- 환경 통계
- 목표 환경

---

## 7. 리포트 생성

예시

```
지난 7일 동안 평균 온도는
22.1℃로 적정 범위를 유지했습니다.

습도는 목표보다 약간 낮은 시간이
2회 발생했지만 자동 제어를 통해
빠르게 복구되었습니다.

전체적으로 매우 안정적인 재배 환경을
유지하고 있습니다.
```

---

## 8. Redis 저장

Key

```
report:{cultivationId}:weekly
```

TTL

```
24시간
```

---

## 9. 사용자 반환

Client

↓

AI 리포트 출력

---

# 사용 Database

## Redis

AI 리포트 Cache

---

## InfluxDB

환경 통계 조회

---

# OpenFeign

```
AI Service

↓

Rule Engine Service
```

---

# RabbitMQ

사용하지 않습니다.

리포트 생성은 사용자 요청 기반의
동기 처리입니다.

---

# Cache 정책

Redis Key

```
report:{cultivationId}:weekly
```

```
report:{cultivationId}:monthly
```

TTL

```
24시간
```

---

# 예외 상황

- Redis 장애
- InfluxDB 조회 실패
- LLM 응답 실패
- Rule Engine Service 호출 실패

---

# 고려 사항

- 동일 기간의 리포트는 Redis에서 조회합니다.
- 환경 통계는 Rule Engine Service에서만 계산합니다.
- AI Service는 자연어 리포트 생성만 담당합니다.
- InfluxDB 원본 데이터는 직접 LLM에 전달하지 않고, 집계 데이터를 전달하여 토큰 사용량을 줄입니다.