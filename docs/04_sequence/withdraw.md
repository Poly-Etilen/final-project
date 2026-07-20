# 회원 탈퇴 시퀀스

## 개요

사용자가 회원 탈퇴를 요청하는 과정입니다.

Auth Service(기존 Auth+User 통합)는 사용자 정보를 Soft Delete 처리하고, 같은 트랜잭션 내에서
Refresh Token을 즉시 삭제한 뒤 UserDeletedEvent를 발행합니다.

Cultivation Service는 이 이벤트를 구독하여 재배 데이터 비활성화를 수행합니다.
(기존에는 Auth Service도 이 이벤트를 구독해 Refresh Token을 삭제했으나, 서비스 통합으로
Refresh Token 삭제가 내부 동기 처리로 바뀌면서 더 이상 이벤트 구독이 필요하지 않습니다.)

> ℹ️ **변경 이력**: Soft Delete 표현 방식이 `deleted_at` 컬럼(NULL 여부로 판단)에서
> `status` 컬럼(ACTIVE/DELETED)으로 바뀌었습니다. 탈퇴 시각은 별도 컬럼 대신 탈퇴 처리 시점의
> `updated_at`으로 확인합니다. (자세한 내용은 [auth-db.md](../03_Database/auth-db.md) 참고)

---

# Sequence

```text
Client

↓

API Gateway

↓

Auth Service

↓

사용자 조회

↓

PostgreSQL

↓

Soft Delete

↓

Refresh Token 삭제 (Redis, 내부 동기 처리)

↓

RabbitMQ

↓

UserDeletedEvent 발행

└── Cultivation Service
      └── 재배 데이터 비활성화

↓

Client

↓

탈퇴 완료
```

---

# 상세 과정

## 1. 탈퇴 요청

사용자가 회원 탈퇴를 요청합니다.

예시

```http
DELETE /users/me
```

---

## 2. 사용자 조회

Auth Service는

```
users
```

에서 요청한 사용자를 조회합니다.

↓

이미 탈퇴한 사용자인지 확인합니다.

---

## 3. Soft Delete 처리

users 테이블의

```
status
```

컬럼을 `'DELETED'`로 변경합니다. `updated_at`도 이 시점으로 함께 갱신되어, 별도 컬럼 없이도
탈퇴 시각을 확인할 수 있습니다.

예시

```
status = 'DELETED'
updated_at = 2026-08-15T13:30:00
```

사용자 정보는 즉시 삭제하지 않습니다.

---

## 4. Refresh Token 삭제 (내부 동기 처리)

같은 요청 트랜잭션 내에서 Redis의 Refresh Token을 즉시 삭제합니다.

Key

```
refresh:{userId}
```

삭제 직후부터 해당 사용자는 재로그인이 불가능합니다.
(기존에는 RabbitMQ 이벤트를 거쳐 비동기로 처리했으나, 같은 서비스 내부 로직이 되면서 동기 처리로 단순화되었습니다.)

---

## 5. Event 발행

Auth Service

↓

RabbitMQ Publish

```json
{
    "userId":15,
    "deletedAt":"2026-08-15T13:30:00"
}
```

발행 이벤트

```
UserDeletedEvent
```

---

## 6. Cultivation Service 처리

RabbitMQ Subscribe

↓

user_id 기준으로 보유한 재배 데이터를 비활성화합니다.

- RUNNING 상태인 재배는 더 이상 대시보드에 노출되지 않습니다.
- 재배 데이터 자체는 삭제하지 않습니다.

---

## 7. 응답

Client에게

```json
{
    "message":"Withdrawal Completed"
}
```

전달

---

# 사용 Database

## PostgreSQL

```
users (Auth DB)
```

---

## Redis

```
refresh:{userId} (Auth DB)
```

---

# OpenFeign

사용하지 않습니다.

Cultivation Service로의 후속 처리는 이벤트 기반(RabbitMQ)으로 비동기 처리합니다.

---

# RabbitMQ

Publish

```
UserDeletedEvent
```

Subscribe

- Cultivation Service

---

# 예외 상황

- 존재하지 않는 사용자
- 이미 탈퇴한 사용자
- RabbitMQ 발행 실패
- Cultivation Service 처리 실패

---

# 고려 사항

- 회원 탈퇴는 `status = 'DELETED'`로 표현하는 Soft Delete로 처리하며 개인정보를 즉시 파기하지 않습니다.
- Refresh Token 삭제는 Auth Service 내부에서 동기로 즉시 처리되어, 탈퇴 응답 시점부터 재로그인이 불가능합니다.
- Auth Service는 이벤트 발행 이후 Cultivation Service의 처리 결과를 기다리지 않습니다.
- Cultivation Service의 후속 처리가 실패해도 탈퇴 자체는 롤백하지 않습니다.
- Access Token은 만료 시간(30분)이 지나면 자동으로 무효화됩니다.
