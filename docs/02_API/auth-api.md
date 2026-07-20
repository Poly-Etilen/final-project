# Auth API

## 개요

Auth Service에서 제공하는 REST API 명세입니다.

Auth Service는 인증(로그인/토큰/이메일 인증)과 회원 프로필(정보 조회/수정/탈퇴/통계)을 함께 제공합니다.
(기존 Auth Service + User Service 통합)

Base URL

```
/api/v1/auth   (인증)
/api/v1/users  (프로필)
```

인증 방식

```
회원가입, 로그인, 휴면 계정 재활성화, 구글 로그인, 이메일 인증, 토큰 재발급은 인증 불필요
그 외 요청은 Bearer JWT
```

> ℹ️ **변경 이력**: 구글 소셜 로그인(`POST /auth/google`)이 추가되었습니다. 이메일/비밀번호
> 회원가입은 `provider='LOCAL'`로, 구글 로그인은 `provider='GOOGLE'`로 구분해 저장합니다.
> 회원 탈퇴 처리는 `status='DELETED'`로 표현하며, 정확한 탈퇴 시각은 `deleted_at`에
> 함께 기록합니다.

> ℹ️ **변경 이력**: 휴면 계정(`DORMANT`) 정책이 추가되었습니다. `POST /auth/login`에서
> 비밀번호 검증에 성공해도 `last_login_at` 기준 장기 미로그인이면 즉시 로그인시키지 않고
> 이메일로 인증번호를 발송한 뒤, 새로 추가된 `POST /auth/login/reactivate`로 인증번호를
> 확인해야 로그인이 완료됩니다. 자세한 내용은 [auth-db.md](../03_Database/auth-db.md)의
> "휴면 계정 (Dormant)" 참고.

---

# 이메일 인증 요청

## POST /auth/email/send

회원가입을 위한 이메일 인증번호를 발송합니다.

### Request

```json
{
    "email": "mushroom@example.com"
}
```

---

### Process

Client

↓

Auth Service

↓

이메일 중복 확인 (PostgreSQL)

↓

인증번호 생성 (Redis 저장, TTL 5분)

↓

SMTP 발송

---

### Response

```json
{
    "message": "인증번호가 발송되었습니다."
}
```

---

# 이메일 인증 확인

## POST /auth/email/verify

### Request

```json
{
    "email": "mushroom@example.com",
    "code": "845231"
}
```

---

### Response

```json
{
    "message": "이메일 인증이 완료되었습니다."
}
```

---

# 회원가입

## POST /auth/signup

이메일 인증이 완료된 사용자만 회원가입할 수 있습니다.

### Request

```json
{
    "email": "mushroom@example.com",
    "password": "P@ssw0rd123",
    "nickname": "느타리팜"
}
```

---

### Process

Client

↓

Auth Service

↓

이메일 인증 여부 확인 (Redis)

↓

users 생성 (provider='LOCAL', 인증 정보 + 프로필 정보를 하나의 트랜잭션으로 저장)

Auth+User가 통합되기 전에는 이 시점에 User Service로 별도 OpenFeign 호출이 필요했지만,
지금은 같은 서비스 내부 로직이라 호출이 없습니다.

---

### Response

```json
{
    "message": "회원가입이 완료되었습니다."
}
```

---

# 로그인

## POST /auth/login

### Request

```json
{
    "email": "mushroom@example.com",
    "password": "P@ssw0rd123"
}
```

---

### Process

Client

↓

Auth Service

↓

email로 users 조회

↓

status = 'DELETED' → A010 반환 (로그인 거부)

↓

비밀번호 검증 (BCrypt)

↓

status 판단

```
status = 'ACTIVE' AND (지금 - last_login_at) <= 휴면 기준일(90일)
→ 정상 로그인. last_login_at 갱신, JWT 발급

status = 'DORMANT' 이거나
status = 'ACTIVE' AND (지금 - last_login_at) > 휴면 기준일
→ status를 'DORMANT'로 전환, 인증번호 발송(Redis, TTL 5분), JWT는 발급하지 않음
```

---

### Response (정상 로그인)

```json
{
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 1800
}
```

Access Token 유효시간은 30분, Refresh Token 유효시간은 14일이며 Redis에 저장됩니다.

---

### Response (휴면 전환됨)

```json
{
    "message": "휴면 계정입니다. 이메일로 인증번호를 발송했습니다.",
    "reactivationRequired": true
}
```

토큰은 발급되지 않습니다. 클라이언트는 `POST /auth/login/reactivate`로 인증번호를 제출해야
로그인이 완료됩니다.

---

# 휴면 계정 재활성화

## POST /auth/login/reactivate

`POST /auth/login`에서 `reactivationRequired: true`를 받은 뒤, 이메일로 받은 인증번호를
제출하여 로그인을 완료합니다.

### Request

```json
{
    "email": "mushroom@example.com",
    "code": "845231"
}
```

---

### Process

Client

↓

Auth Service

↓

Redis에서 인증번호 검증 (email:{email})

↓

status를 'ACTIVE'로 전환, last_login_at 갱신

↓

JWT 발급 (Access Token + Refresh Token)

비밀번호는 최초 `POST /auth/login` 시도에서 이미 검증되었으므로 여기서는 다시 요구하지
않습니다.

---

### Response

```json
{
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 1800
}
```

응답 형식은 `POST /auth/login`의 정상 로그인 응답과 동일합니다.

---

# 구글 로그인

## POST /auth/google

프론트엔드에서 구글 로그인(Google Identity Services)으로 발급받은 ID Token을 전달받아
검증하고 로그인시킵니다. 최초 로그인이면 자동으로 회원가입까지 처리합니다.

### Request

```json
{
    "idToken": "eyJhbGciOiJSUzI1NiIs..."
}
```

---

### Process

Client

↓

Auth Service

↓

구글 공개키로 ID Token 서명 검증 + 만료 확인

↓

이메일/이름/프로필 이미지 추출

↓

users에서 (email, provider='GOOGLE')로 조회

↓

존재하면 → 바로 JWT 발급
존재하지 않으면 → users 자동 생성(provider='GOOGLE', password=NULL,
email_verified=true, nickname=구글 프로필 이름 기반, profile_image_url=구글 프로필 이미지)
후 JWT 발급

---

### Response

```json
{
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 1800,
    "isNewUser": false
}
```

`isNewUser`는 이번 요청으로 신규 가입되었는지 여부입니다. 나머지 응답 형식은 `POST
/auth/login`과 동일합니다.

같은 이메일로 LOCAL 계정이 이미 있어도 별도의 GOOGLE 계정으로 새로 생성됩니다(계정 연동은
지원하지 않음).

---

# 로그아웃

## POST /auth/logout

### Header

```
Authorization: Bearer {accessToken}
```

---

### Response

```json
{
    "message": "로그아웃되었습니다."
}
```

---

# Access Token 재발급

## POST /auth/refresh

### Request

```json
{
    "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

---

### Response

```json
{
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 1800
}
```

---

# 내 정보 조회

## GET /users/me

### Response

```json
{
    "userId": 15,
    "email": "mushroom@example.com",
    "provider": "LOCAL",
    "nickname": "느타리팜",
    "profileImageUrl": "https://minio/profile/15.jpg",
    "createdAt": "2026-07-01T10:00:00"
}
```

---

# 사용자 정보 수정

## PATCH /users/me

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

## DELETE /users/me

### Process

Auth Service

↓

users Soft Delete (status='DELETED', deleted_at=현재 시각) + Refresh Token 삭제 (같은 서비스 내부 처리, 즉시 수행)

↓

RabbitMQ Publish

↓

UserDeletedEvent

↓

Cultivation Service (재배 데이터 비활성화)

Auth+User가 분리되어 있던 시절에는 Refresh Token 삭제도 이벤트를 구독해서 처리했지만,
통합 이후에는 같은 서비스 내부라 즉시 처리하고 이벤트는 Cultivation Service를 위해서만 발행합니다.

---

### Response

```json
{
    "message": "회원 탈퇴가 완료되었습니다."
}
```

---

# 내 재배 통계 조회

## GET /users/me/statistics

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
| A001 | 존재하지 않는 이메일 |
| A002 | 비밀번호 불일치 |
| A003 | 이메일 중복 |
| A004 | 이메일 인증 미완료 |
| A005 | 인증번호 불일치 또는 만료 |
| A006 | 만료된 Access Token |
| A007 | 만료된 Refresh Token 또는 불일치 |
| A008 | 존재하지 않는 사용자 |
| A009 | 중복 닉네임 |
| A010 | 이미 탈퇴한 사용자 |
| A011 | 유효하지 않은 구글 ID Token (서명 검증 실패, 만료 등) |
| A012 | 구글 인증 서버 응답 실패/시간 초과 |
| A013 | 휴면 계정 (이메일 재인증 필요, `POST /auth/login/reactivate`로 재시도) |

---

# OpenFeign

호출하는 서비스

```
없음
```

호출받는 서비스

```
Cultivation Service (사용자 정보 조회)
```

---

# Event

## 발행 이벤트

### UserDeletedEvent

회원 탈퇴 시 발행됩니다.

구독 서비스

```
Cultivation Service (재배 데이터 비활성화)
```
