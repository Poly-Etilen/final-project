# Notification Service

## 역할

Notification Service는 시스템에서 발생한 이벤트를 Telegram/Discord 채널로 전달하는
서비스입니다. RabbitMQ로 이벤트를 수신하며, 재배별로 등록된 알림 채널로 알림을 발송하고
발송 이력을 관리합니다.

---

# 책임

- Telegram/Discord 알림 발송
- 알림 발송 이력 관리
- 재배별 알림 채널(Telegram/Discord) 등록/조회/삭제

---

# 주요 기능

## 환경 이상/자동 제어/센서 오류/리포트/피드백/수확/종료 알림

Rule Engine/Sensor/Cultivation/AI Service가 발행하는 이벤트를 구독해 `notification_event`로
기록하고, 이벤트에 실린 `cultivationId`로 `notification_endpoint`를 조회해 등록된 채널로
발송합니다.

---

## 알림 채널 등록/조회/삭제

특정 재배에 Telegram Chat ID 또는 Discord Webhook URL을 등록해 알림을 받을 채널을
설정합니다. 채널은 **재배 단위**로 등록됩니다 — 같은 재배에 접근하는 사용자라면 등록된
채널을 함께 씁니다. 채널을 등록하지 않은 재배는 해당 채널로 알림을 받지 않습니다.

---

## 알림 발송 이력 조회

특정 재배로 발송된 알림 이벤트와 채널별 발송 내역(문구, 발송 일시)을 조회합니다.

---

# API

## 알림 채널 등록/조회/삭제

POST /notifications/endpoints

GET /notifications/endpoints

DELETE /notifications/endpoints/{endpointId}

---

## 알림 발송 이력 조회

GET /notifications

---

# Database

Notification Service는 하나의 PostgreSQL Database를 사용합니다.

## Table

- notification_event
- notification_delivery
- notification_endpoint (재배 단위)

자세한 내용은 [notification-db.md](../03_Database/notification-db.md) 참고.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Telegram Bot API / Discord Webhook

메시지 전송 (외부 서비스)

---

## 호출받는 서비스

### AI Service

- Telegram/Discord 챗봇 웹훅 발신자 → cultivationId 조회 (`notification_endpoint` 재사용)

### API Gateway

- 알림 채널 등록/조회/삭제, 발송 이력 조회 REST API 요청

---

# Event

## Subscribe

- EnvironmentControlEvent
- SensorErrorEvent
- HarvestCompletedEvent
- CultivationFinishedEvent
- WeeklyReportCompletedEvent
- DailyFeedbackCompletedEvent

---

# Sequence

관련 시퀀스는 [environment-control.md](../04_sequence/environment-control.md),
[harvest.md](../04_sequence/harvest.md), [sensor-error.md](../04_sequence/sensor-error.md) 참고.

---

# 예외 상황

- Telegram/Discord 발송 실패
- 등록된 알림 채널이 없는 재배 (발송 자체를 시도하지 않음, 에러 아님)
- 존재하지 않는 알림 채널(endpoint) 삭제/조회 시도
- 다른 재배의 알림 채널에 대한 조회/삭제 시도

---

# 추후 개발 예정

- Email/Push 알림, Slack 연동
- 재시도 정책(최대 횟수/주기) 확정
