# 회원 탈퇴 시퀀스

## 개요

사용자가 회원 탈퇴를 요청하는 과정입니다.

User Service는 사용자 정보를 Soft Delete 처리하고 UserDeletedEvent를 발행합니다.

Auth Service와 Cultivation Service는 이 이벤트를 구독하여
각각 Refresh Token 삭제와 재배 데이터 비활성화를 수행합니다.

---

# Sequence

```text
Client

↓

API Gateway

↓

User Service

↓

사용자 조회

↓

PostgreSQL

↓

Soft Delete

↓

RabbitMQ

↓

UserDeletedEvent 발행

├── Auth Service
│     └── Refresh Token 삭제 (Redis)
│
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

User Service는

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
deleted_at
```

컬럼에 현재 시각을 저장합니다.

예시

```
deleted_at = 2026-08-15T13:30:00
```

사용자 정보는 즉시 삭제하지 않습니다.

---

## 4. Event 발행

User Service

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

## 5. Auth Service 처리

RabbitMQ Subscribe

↓

Redis에서 Refresh Token을 삭제합니다.

Key

```
refresh:{userId}
```

삭제 후 해당 사용자는 재로그인이 불가능합니다.

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
users (User DB)
```

---

## Redis

```
refresh:{userId} (Auth DB)
```

---

# OpenFeign

사용하지 않습니다.

탈퇴 후속 처리는 이벤트 기반(RabbitMQ)으로 비동기 처리합니다.

---

# RabbitMQ

Publish

```
UserDeletedEvent
```

Subscribe

- Auth Service
- Cultivation Service

---

# 예외 상황

- 존재하지 않는 사용자
- 이미 탈퇴한 사용자
- RabbitMQ 발행 실패
- Auth Service 처리 실패
- Cultivation Service 처리 실패

---

# 고려 사항

- 회원 탈퇴는 Soft Delete로 처리하며 개인정보를 즉시 파기하지 않습니다.
- User Service는 이벤트 발행 이후 결과를 기다리지 않습니다.
- Auth/Cultivation Service의 후속 처리가 실패해도 탈퇴 자체는 롤백하지 않습니다.
- Refresh Token이 삭제되기 전까지는 이미 발급된 Access Token으로 요청이 가능할 수 있습니다.
- Access Token은 만료 시간(30분)이 지나면 자동으로 무효화됩니다.
