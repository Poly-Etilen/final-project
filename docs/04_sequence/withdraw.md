# 회원 탈퇴 시퀀스

## 개요

사용자가 회원 탈퇴를 요청하는 과정입니다. Auth Service는 `users`를 Soft Delete
처리하고, 같은 트랜잭션 내에서 Refresh Token을 즉시 삭제한 뒤 `UserDeletedEvent`를
발행합니다. Cultivation Service는 이 이벤트를 구독해 재배 데이터를 비활성화합니다.

---

# Sequence

```text
Client
↓
API Gateway
↓
Auth Service
↓
사용자 조회 (PostgreSQL)
↓
Soft Delete (status = 'DELETED', deleted_at 기록)
↓
Refresh Token 삭제 (Redis, 내부 동기 처리)
↓
RabbitMQ Publish (UserDeletedEvent)
└── Cultivation Service → 재배 데이터 비활성화
↓
Client
↓
탈퇴 완료
```

---

# 상세 과정

## 1. 탈퇴 요청

```http
DELETE /api/v1/auth/users/me
```

---

## 2. 사용자 조회

`users`에서 요청한 사용자를 조회하고, 이미 탈퇴한 사용자인지 확인합니다.

---

## 3. Soft Delete 처리

`status`를 `'DELETED'`로 변경하고, 같은 시점에 `deleted_at`에 탈퇴 처리 시각을
기록합니다.

```
status = 'DELETED'
deleted_at = 2026-08-15T13:30:00
```

`users` 행 자체는 삭제하지 않습니다.

---

## 4. Refresh Token 삭제 (내부 동기 처리)

같은 요청 트랜잭션 내에서 Redis의 `refresh:{userId}`를 즉시 삭제합니다. 삭제 직후부터
해당 사용자는 재로그인이 불가능합니다.

---

## 5. Event 발행

Auth Service → RabbitMQ Publish

```json
{
    "userId": 15,
    "deletedAt": "2026-08-15T13:30:00"
}
```

발행 이벤트: `UserDeletedEvent`

---

## 6. Cultivation Service 처리

RabbitMQ Subscribe → `user_id` 기준으로 보유한 재배 데이터를 비활성화합니다.

- `RUNNING` 상태인 재배는 더 이상 대시보드에 노출되지 않습니다.
- 재배 데이터 자체는 삭제하지 않습니다.

---

## 7. 응답

```json
{ "message": "탈퇴가 완료되었습니다." }
```

---

# 사용 Database

## PostgreSQL

```
users (Auth DB)
```

## Redis

```
refresh:{userId} (Auth DB)
```

---

# OpenFeign

사용하지 않습니다. Cultivation Service로의 후속 처리는 이벤트 기반(RabbitMQ)으로
비동기 처리합니다.

---

# RabbitMQ

Publish

```
UserDeletedEvent
```

Subscribe

```
Cultivation Service
```

---

# 예외 상황

- 존재하지 않는 사용자
- 이미 탈퇴한 사용자
- RabbitMQ 발행 실패
- Cultivation Service 처리 실패

---

# 고려 사항

- 회원 탈퇴는 `status = 'DELETED'` + `deleted_at`(정확한 탈퇴 시각)으로 표현하는 Soft
  Delete로 처리하며, 개인정보를 즉시 파기하지 않습니다. 별도의 `withdrawn_user` 테이블은
  두지 않습니다.
- Refresh Token 삭제는 Auth Service 내부에서 동기로 즉시 처리되어, 탈퇴 응답 시점부터
  재로그인이 불가능합니다.
- Auth Service는 이벤트 발행 이후 Cultivation Service의 처리 결과를 기다리지 않습니다.
- Cultivation Service의 후속 처리가 실패해도 탈퇴 자체는 롤백하지 않습니다.
- 계정 복구(탈퇴 취소) 기능은 추후 개발 예정입니다.
