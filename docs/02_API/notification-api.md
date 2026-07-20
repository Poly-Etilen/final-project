# Notification API

## 개요

Notification Service에서 제공하는 REST API 명세입니다.

> ℹ️ **변경 이력**: 알림 이력 조회 기능이 추가되면서 Notification Service가 처음으로 REST API를
> 제공하게 되었습니다. 기존에는 RabbitMQ 이벤트만 구독해 WebSocket/Telegram/Discord로
> 발송하고 끝이었지만, 이제 발송한 알림을 `notification` 테이블에 저장하고 조회/읽음 처리
> API를 제공합니다. (자세한 내용은 [notification.md](../01_Domain/notification.md),
> [notification-db.md](../03_Database/notification-db.md) 참고)

Base URL

```
/api/v1/notifications
```

인증 방식

```
Bearer JWT
```

---

# 알림 목록 조회

## GET /

로그인한 사용자의 알림을 최신순으로 조회합니다.

### Query Parameter

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| page | int | X | 페이지 번호 (기본값 0) |
| size | int | X | 페이지 크기 (기본값 20) |
| unreadOnly | boolean | X | true면 읽지 않은 알림만 조회 (기본값 false) |

---

### Response

```json
{
    "notifications": [
        {
            "notificationId": 42,
            "type": "SENSOR_ERROR",
            "message": "온도 센서 연결 실패",
            "isRead": false,
            "createdAt": "2026-08-15T13:30:00"
        },
        {
            "notificationId": 41,
            "type": "ENVIRONMENT_CONTROL",
            "message": "습도가 낮아 가습기를 자동으로 실행했습니다.",
            "isRead": true,
            "createdAt": "2026-08-15T09:12:00"
        }
    ],
    "unreadCount": 1
}
```

---

# 알림 읽음 처리

## PATCH /{notificationId}/read

특정 알림을 읽음 상태로 변경합니다.

### Process

Notification Service

↓

`notification` 조회 (요청자 소유 확인)

↓

`is_read = true` 갱신

---

### Response

```json
{
    "message": "읽음 처리되었습니다."
}
```

---

# 전체 읽음 처리

## PATCH /read-all

로그인한 사용자의 읽지 않은 알림을 모두 읽음 상태로 변경합니다.

### Response

```json
{
    "message": "전체 읽음 처리되었습니다.",
    "updatedCount": 5
}
```

---

# Error Code

| Code | Description |
|------|-------------|
| N001 | 존재하지 않는 알림 |
| N002 | 다른 사용자의 알림에 대한 접근 시도 |

---

# OpenFeign

호출하는 서비스

```
없음
```

호출받는 서비스

```
없음 (RabbitMQ 이벤트 기반으로만 데이터가 생성됩니다)
```

---

# Event

RabbitMQ 구독 이벤트를 통해 알림이 생성됩니다. 자세한 내용은
[notification.md](../01_Domain/notification.md)의 RabbitMQ 섹션 참고.

이 API는 이미 생성된 알림의 조회/읽음 처리만 담당하며, 알림 생성 자체는 이벤트 구독으로
비동기 처리되어 이 API와 무관하게 이루어집니다.
