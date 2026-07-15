# Notification Service

## 역할

Notification Service는 시스템에서 발생한 이벤트를 사용자에게 다양한 채널을 통해 실시간으로 전달하는 서비스입니다.

RabbitMQ를 통해 이벤트를 수신하며,
사용자가 설정한 알림 방식에 따라 WebSocket, Telegram, Discord로 알림을 전송합니다.

---

# 책임

- 실시간 알림 전송
- Telegram 알림
- Discord 알림
- WebSocket Push
- 알림 템플릿 관리
- 알림 채널 관리

---

# 주요 기능

## 환경 이상 알림

재배 환경이 목표 범위를 벗어난 경우 사용자에게 알림을 전송합니다.

예시

- 습도가 너무 낮습니다.
- CO₂ 농도가 높습니다.
- 온도가 권장 범위를 초과했습니다.

---

## 자동 제어 알림

Rule Engine이 자동으로 장치를 제어했을 경우 사용자에게 알려줍니다.

예시

- 가습기가 자동으로 실행되었습니다.
- 환풍기가 자동으로 실행되었습니다.
- LED가 자동으로 켜졌습니다.

---

## 센서 오류 알림

센서가 정상적으로 동작하지 않을 경우 사용자에게 알립니다.

예시

- 온도 센서 연결 실패
- 습도 센서 응답 없음

---

## AI 리포트 생성 알림

Scheduler 기반으로 생성되는 주간/월간 AI 리포트가 준비되면 사용자에게 알립니다.

예시

- 이번 주 재배 리포트가 도착했습니다.
- 이번 달 재배 리포트가 도착했습니다.

AI 생육 분석(Vision)은 사용자가 사진을 업로드하면 그 자리에서 결과가 반환되는 동기 방식이므로
별도의 알림을 발행하지 않습니다.

---

## 수확 완료 알림

재배가 종료되고 수확 정보가 저장되면 사용자에게 알립니다.

예시

- 느타리 1호기 재배가 종료되었습니다. 수확량: 3.2kg

---

# API

Notification Service는 외부 API를 제공하지 않습니다.

RabbitMQ 이벤트를 기반으로 동작합니다.

---

# Database

별도의 Database를 사용하지 않습니다.

---

# Redis

사용하지 않습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

### Telegram Bot API

Telegram 메시지 전송

---

### Discord Webhook

Discord 메시지 전송

---

## 호출받는 서비스

### RabbitMQ

이벤트 수신

---

# RabbitMQ

## Subscribe Event

### EnvironmentControlEvent

자동 제어 결과

---

### SensorErrorEvent

센서 오류 / 연결 해제

---

### HarvestCompletedEvent

수확 완료

---

### CultivationFinishedEvent

재배 종료

---

### WeeklyReportCompletedEvent

주간 리포트 생성 완료

---

### MonthlyReportCompletedEvent

월간 리포트 생성 완료

---

# WebSocket

사용자가 웹 페이지를 열고 있는 경우

실시간으로 알림을 Push합니다.

예시

```
습도가 낮아
가습기를 자동 실행했습니다.
```

---

# Telegram

사용자가 Telegram 연동을 활성화한 경우

Telegram Bot을 통해 메시지를 전송합니다.

예시

```
🍄 버섯 재배 알림

습도가 낮아
가습기를 자동으로 실행했습니다.
```

---

# Discord

사용자가 Discord 연동을 활성화한 경우

Webhook을 이용하여 메시지를 전송합니다.

예시

```
🍄 Mushroom Notification

Temperature High

Cooling Fan ON
```

---

# Sequence

Rule Engine

↓

RabbitMQ

↓

Notification Service

↓

알림 채널 확인

├── WebSocket

├── Telegram

└── Discord

↓

사용자

---

# 예외 상황

- Telegram 전송 실패
- Discord Webhook 실패
- WebSocket 연결 종료
- RabbitMQ 연결 실패

---

# 추후 개발 예정

- Email 알림
- Push Notification
- Slack 연동
- 알림 우선순위 설정
- 알림 ON/OFF 설정