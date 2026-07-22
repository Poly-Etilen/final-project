# AI 리포트 생성 시퀀스

## 개요

AI 리포트는 재배 기간 동안 수집된 환경 데이터를 분석해 사용자에게 주간 재배 리포트를
제공합니다.

Sensor Service의 Weekly Scheduler가 InfluxDB에서 통계 데이터를 집계해 AI Service에
전달(push)하면, AI Service는 LLM을 이용해 자연어 리포트를 생성하고 미리 만들어
둡니다. 사용자는 리포트를 "생성 요청"하지 않고 이미 만들어진 리포트를 "조회"만
합니다. 재배 기간이 한 달을 넘지 않는 도메인 특성상 월간 리포트는 제공하지 않습니다.

---

# Sequence

```text
Weekly Scheduler (Sensor Service)
↓
InfluxDB 조회 (기간별 집계)
↓
AI Service (OpenFeign, push)
↓
LLM
↓
리포트 생성
↓
Redis 저장 (report:{cultivationId}:weekly)
↓
RabbitMQ Publish (WeeklyReportCompletedEvent) → Notification Service

(별도 흐름) Client → API Gateway → AI Service → Redis 조회 → Client
```

---

# 상세 과정

## 1. Weekly Scheduler 실행

Sensor Service의 Weekly Scheduler가 매주 정해진 시각에 실행되어, `RUNNING` 상태인
재배를 순회합니다.

---

## 2. 환경 데이터 조회

Sensor Service → InfluxDB → 최근 7일 기간별 집계

조회 항목: 평균 온도/습도/CO₂/조도, 최고/최저 온도, 최고/최저 습도

---

## 3. AI Service로 전달 (push)

Sensor Service → OpenFeign → AI Service

```json
{
    "cultivationId": 3,
    "averageTemperature": 22.1,
    "averageHumidity": 90.3,
    "averageCo2": 810,
    "averageLight": 430,
    "maxTemperature": 24.5,
    "minTemperature": 20.2
}
```

사용자 요청을 기다리지 않고 Sensor Service가 먼저 호출을 시작합니다.

---

## 4. LLM 요청

입력: 버섯 종류, 환경 통계, 목표 환경

---

## 5. 리포트 생성

```
지난 7일 동안 평균 온도는 22.1℃로 적정 범위를 유지했습니다.
습도는 목표보다 약간 낮은 시간이 2회 발생했지만 자동 제어를 통해 빠르게 복구되었습니다.
전체적으로 매우 안정적인 재배 환경을 유지하고 있습니다.
```

---

## 6. Redis 저장

```
Key: report:{cultivationId}:weekly
TTL: 24시간
```

다음 주 Weekly Scheduler 실행 시 새 리포트로 덮어씁니다. TTL은 이벤트/스케줄러 유실
등으로 리포트가 오래 방치되는 것을 막기 위한 안전장치입니다.

---

## 7. 완료 이벤트 발행

AI Service → RabbitMQ Publish(`WeeklyReportCompletedEvent`) → Notification Service
("이번 주 재배 리포트가 도착했습니다" 알림)

---

## 8. 사용자 조회 (별도 흐름)

위 1~7번과는 독립적으로, 사용자는 언제든 이미 생성된 리포트를 조회할 수 있습니다.

```http
GET /api/v1/ai/report?cultivationId=3
```

AI Service가 Redis(`report:{cultivationId}:weekly`)를 조회해 반환합니다. 아직 첫
주간 리포트가 생성되지 않았다면(재배 시작 후 첫 주가 지나지 않음) 에러를 반환합니다.

---

# 사용 Database

## Redis

AI 리포트 저장 (Weekly Scheduler가 push로 채워두는 저장소 역할까지 겸함)

## InfluxDB

환경 통계 조회

---

# OpenFeign

```
Sensor Service → AI Service
```

주간 통계 전달은 AI Service가 아니라 Sensor Service가 호출을 시작합니다(push).

---

# RabbitMQ

Publish

```
WeeklyReportCompletedEvent (AI Service, 리포트 생성 완료 시)
```

Subscribe

```
Notification Service
```

---

# Cache 정책

```
Key: report:{cultivationId}:weekly
TTL: 24시간
```

매주 Weekly Scheduler 실행 시 새 값으로 덮어씁니다.

---

# 예외 상황

- Weekly Scheduler 실행 실패 (해당 주 리포트가 갱신되지 않고, 이전 리포트가 Redis에
  남아있음)
- InfluxDB 조회 실패 / LLM 응답 실패
- AI Service 호출 실패 (Sensor Service → AI Service push 실패)
- `WeeklyReportCompletedEvent` 발행 실패 (리포트 자체는 Redis에 저장되어 조회는
  가능하지만 알림이 가지 않음)
- Redis 장애 (조회 시)
- 아직 생성된 리포트가 없는 상태에서 조회 시도

---

# 고려 사항

- 리포트 생성은 사용자 요청이 아니라 Weekly Scheduler가 트리거합니다. 사용자는
  조회만 합니다.
- 환경 통계는 Sensor Service에서만 계산합니다. AI Service는 자연어 리포트 생성만
  담당합니다.
- InfluxDB 원본 데이터는 직접 LLM에 전달하지 않고, 집계 데이터를 전달해 토큰 사용량을
  줄입니다.
- 재배 기간이 한 달을 넘지 않는다는 도메인 특성상 월간 리포트는 만들지 않습니다.
- Redis TTL(24시간)이 다음 Weekly Scheduler 실행(7일 후)보다 훨씬 짧기 때문에,
  정상적으로는 다음 주 리포트로 갱신되기 전에 리포트가 만료된 채로 짧게 "리포트 없음"
  상태를 거칠 수 있습니다. 이 간극을 없애려면 TTL을 늘리거나 영구 저장소로 옮기는
  것을 추후 검토할 수 있습니다(현재는 24시간 TTL을 유지).
