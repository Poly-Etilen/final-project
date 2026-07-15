# User API

## 개요

User Service에서 제공하는 REST API 명세입니다.

Base URL

```
/api/v1/users
```

인증 방식

```
Bearer JWT
```

---

# 내 정보 조회

## GET /me

### Response

```json
{
    "userId": 15,
    "email": "mushroom@example.com",
    "nickname": "느타리팜",
    "profileImageUrl": "https://minio/profile/15.jpg",
    "createdAt": "2026-07-01T10:00:00"
}
```

email은 Auth Service의 정보를 조합하여 반환합니다.

---

# 사용자 정보 수정

## PATCH /me

### Request

```json
{
    "nickname": "새느타리팜",
    "profileImageUrl": "https://minio/profile/15-new.jpg"
}
```

---

### Response

```json
{
    "message": "정보가 수정되었습니다."
}
```

---

# 회원 탈퇴

## DELETE /me

### Process

User Service

↓

users Soft Delete (deleted_at 저장)

↓

RabbitMQ Publish

↓

UserDeletedEvent

↓

Auth Service (Refresh Token 삭제)
Cultivation Service (재배 데이터 비활성화)

---

### Response

```json
{
    "message": "회원 탈퇴가 완료되었습니다."
}
```

---

# 내 재배 통계 조회

## GET /me/statistics

### Response

```json
{
    "totalCultivationCount": 12,
    "totalHarvestWeight": 38400,
    "averageCultivationDays": 26,
    "mostCultivatedMushroomType": "OYSTER"
}
```

---

# Error Code

| Code | Description |
|------|-------------|
| U001 | 존재하지 않는 사용자 |
| U002 | 중복 닉네임 |
| U003 | 이미 탈퇴한 사용자 |
| U004 | 권한 없음 |

---

# OpenFeign

호출받는 서비스

```
Auth Service (회원가입 시 프로필 생성)
Cultivation Service (사용자 정보 조회)
```

호출하는 서비스

```
없음
```

---

# Event

## 발행 이벤트

### UserDeletedEvent

회원 탈퇴 시 발행됩니다.

구독 서비스

- Auth Service
- Cultivation Service
