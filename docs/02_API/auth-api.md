# Auth API

## 개요

Auth Service에서 제공하는 REST API 명세입니다.

Base URL

```
/api/v1/auth
```

인증 방식

```
회원가입, 로그인, 이메일 인증, 토큰 재발급은 인증 불필요
그 외 요청은 Bearer JWT
```

---

# 이메일 인증 요청

## POST /email/send

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

## POST /email/verify

발송된 인증번호를 확인합니다.

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

인증 성공 여부는 Redis에 임시로 기록되며, 실제 회원 정보는 아직 생성되지 않습니다.

---

# 회원가입

## POST /signup

이메일 인증이 완료된 사용자만 회원가입할 수 있습니다.

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

이메일 인증 여부 확인 (Redis)

↓

auth_user 생성 (PostgreSQL)

↓

OpenFeign

↓

User Service

↓

users 프로필 생성 (기본 닉네임 자동 생성)

---

### Response

```json
{
    "message": "회원가입이 완료되었습니다."
}
```

---

# 로그인

## POST /login

### Request

```json
{
    "email": "mushroom@example.com",
    "password": "P@ssw0rd123"
}
```

---

### Response

```json
{
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 1800
}
```

Access Token 유효시간은 30분, Refresh Token 유효시간은 14일이며 Redis에 저장됩니다.

---

# 로그아웃

## POST /logout

### Header

```
Authorization: Bearer {accessToken}
```

---

### Process

Auth Service

↓

Redis에서 Refresh Token 삭제

Key

```
refresh:{userId}
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

## POST /refresh

### Request

```json
{
    "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

---

### Process

Auth Service

↓

Redis 조회 (refresh:{userId})

↓

Refresh Token 일치 확인

↓

새로운 Access Token 발급

---

### Response

```json
{
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 1800
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
| A008 | User Service 프로필 생성 실패 |

---

# OpenFeign

```
Auth Service

↓

User Service (회원가입 시 프로필 생성)
```

---

# Event

현재 발행하는 이벤트는 없습니다.

회원가입은 OpenFeign을 통한 동기 처리로 이루어집니다.
