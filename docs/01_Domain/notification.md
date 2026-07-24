# Notification Service

## 역할

Notification Service는 시스템에서 발생한 이벤트를 사용자가 등록한 Telegram/Discord
채널로 전달하는 서비스입니다. RabbitMQ로 이벤트를 수신하며, 사용자가 개별적으로
등록한 채널(`notification_endpoint`)과 구독(`notification_subscription`)을 기준으로
알림을 발송하고 발송 이력을 관리합니다.

---

# 책임

- Telegram/Discord 알림 발송
- 알림 발송 이력 관리
- 사용자 단위 알림 채널(endpoint) 등록/조회/삭제
- 사용자별 구독(subscription) 등록/조회/취소
- 구독 가능한 이벤트 카탈로그(subscription type) 제공

---

# 주요 기능

## 알림 채널 등록/조회/삭제

사용자가 자신의 Telegram Chat ID 또는 Discord Webhook URL을 `notification_endpoint`에
등록해 알림을 받을 채널을 설정합니다. 채널은 **사용자 단위**로 등록됩니다 — 특정
재배와 무관하게, 사용자 프로필처럼 관리되며 한 사용자가 같은 채널 유형을 여러 개
등록할 수 있습니다(예: Telegram 채널 2개).

---

## 구독 설정

사용자는 등록해 둔 채널로 어떤 이벤트를, 어떤 대상(예: 특정 재배)에 대해 받을지
`notification_subscription`으로 개별 설정합니다. 구독 가능한 항목은
`notification_subscription_type`에 미리 정의된 카탈로그(예: "수확 완료
알림", 대상 유형 CULTIVATION)에서 고릅니다.

같은 재배에 여러 사용자(OWNER/MEMBER)가 접근하더라도, 각자 자신의 채널로 각자 원하는
이벤트만 독립적으로 구독합니다 — 기존처럼 재배 단위로 채널과 구독을 공유하지
않습니다.

---

## 이벤트 수신 시 알림 발송

Rule Engine/Cultivation/AI Service가 발행하는 이벤트를 RabbitMQ로 구독해 처리합니다.

1. 이벤트 수신 → `notification_event_type` 매칭
2. 이벤트에 실린 대상 ID(예: `cultivationId`)를 `target_id`로 사용해, 매칭되는
   `notification_subscription_type`을 거쳐 `notification_subscription`을
   조회(`enabled = true`, `target_id` 일치)
3. 조회된 구독마다 연결된 `notification_endpoint`의 채널 유형에 맞는
   `notification_template`(이벤트 × 채널)을 찾아 메시지를 렌더링
4. `notification` 행 1건 생성(원본 이벤트 payload + 공통 렌더링 메시지) + 조회된
   구독 수만큼 `notification_delivery` 행 생성
5. 각 `notification_delivery`에 대해 실제 Telegram/Discord 발송

구독이 하나도 없는 이벤트는 `notification`만 남고 `notification_delivery`는 생성되지
않습니다(에러 아님).

> 재배 생성/멤버 참여 시 주요 이벤트를 자동으로 구독시켜 줄지는 아직 정해지지
> 않았습니다. 현재는 사용자가 채널 등록 후 직접 구독을 설정해야 알림을 받습니다.

---

## 알림 발송 이력 조회

사용자가 자신의 구독을 기준으로 발송된 알림 이력(문구, 발송 일시, 성공 여부)을
조회합니다.

---

# API

## 알림 채널 등록/조회/삭제

POST /notifications/endpoints

GET /notifications/endpoints

DELETE /notifications/endpoints/{endpointId}

인증된 사용자 본인(userId) 소유의 채널만 대상이며, `cultivationId` 파라미터는 더 이상
없습니다.

---

## 구독 등록/조회/취소

POST /notifications/subscriptions

```json
{ "subscriptionTypeId": 3, "endpointId": 12, "targetId": 3 }
```

GET /notifications/subscriptions

DELETE /notifications/subscriptions/{subscriptionId}

---

## 구독 가능 카탈로그 조회

GET /notifications/subscription-types

프론트가 "무엇을 구독할 수 있는지"(이벤트 종류, 대상 유형, 채널별 가능 여부)를 보여줄
때 사용합니다.

---

## 알림 발송 이력 조회

GET /notifications

---

# Database

Notification Service는 하나의 PostgreSQL Database를 사용합니다.

## Table

- channel_type (참조, 시드: TELEGRAM/DISCORD)
- subscription_target_type (참조, 시드: CULTIVATION)
- notification_event_type (참조, 시드: 이벤트 6종)
- notification_subscription_type (구독 가능 카탈로그)
- subscription_channel (구독 종류별 사용 가능 채널)
- notification_template (이벤트 × 채널 메시지 템플릿, 버전 관리)
- notification_endpoint (사용자 단위 채널 등록)
- notification_subscription (사용자별 구독)
- notification (수신 이벤트 + 렌더링 메시지)
- notification_delivery (구독별 발송 이력)

자세한 내용은 [notification-db.md](../03_Database/notification-db.md) 참고.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Telegram Bot API / Discord Webhook

메시지 전송 (외부 서비스)

---

## 호출받는 서비스

### AI Service

- Telegram/Discord 챗봇 웹훅 발신자의 `destination`으로 `notification_endpoint`를
  조회해 `user_id`를 확인합니다(챗봇 대화 컨텍스트 식별용). 기존에는 endpoint가
  재배 단위여서 바로 `cultivationId`를 얻었지만, 사용자 단위로 바뀌며 이후
  `notification_subscription`을 통해 해당 사용자가 어떤 재배를 구독 중인지 별도로
  확인해야 합니다.

### API Gateway

- 알림 채널 등록/조회/삭제, 구독 등록/조회/취소, 구독 카탈로그 조회, 발송 이력 조회
  REST API 요청

---

# Event

## Subscribe

- EnvironmentControlEvent
- SensorErrorEvent
- HarvestCompletedEvent
- CultivationFinishedEvent
- CultivationModeChangedEvent
- DailyFeedbackCompletedEvent

---

# Sequence

관련 시퀀스는 [environment-control.md](../04_sequence/environment-control.md),
[harvest.md](../04_sequence/harvest.md), [growth-analysis.md](../04_sequence/growth-analysis.md),
[sensor-error.md](../04_sequence/sensor-error.md),
[daily-feedback.md](../04_sequence/daily-feedback.md) 참고.

---

# 예외 상황

- Telegram/Discord 발송 실패
- 특정 대상(target_id)에 대해 등록된 구독이 없는 경우 (발송 자체를 시도하지 않음,
  에러 아님)
- 존재하지 않는 채널(endpoint)/구독 삭제/조회 시도
- 다른 사용자의 채널/구독에 대한 조회/삭제 시도
- 같은 채널로 같은 대상에 같은 구독 종류를 중복 등록 시도(`notification_subscription`
  UNIQUE 제약 위반)
- 이벤트에 대응하는 `notification_template`(이벤트×채널)이 없는 경우 (발송 불가,
  운영 설정 누락으로 처리)

---

# 추후 개발 예정

- Email/Push 알림, Slack 연동
- 재시도 정책(최대 횟수/주기) 확정
- 재배 생성/멤버 참여 시 주요 이벤트 자동 구독 여부는 추후 결정, 현재는 사용자가
  직접 구독 설정 필요
