# Auth Service API

## 개요

사용자의 인증을 담당한다.

- 회원가입
- 로그인
- 이메일 인증
- OAuth2
- Access Token
- Refresh Token

---

# 회원가입

## POST /api/auth/signup

### Request

```json
{
  "email":"test@test.com",
  "password":"1234",
  "nickname":"eco"
}
```

### Response

```json
{
  "message":"회원가입이 완료되었습니다."
}
```

---

# 이메일 인증

## POST /api/auth/email/verify

### Request

```json
{
  "email":"test@test.com",
  "code":"123456"
}
```

---

# 로그인

## POST /api/auth/login

### Request

```json
{
  "email":"test@test.com",
  "password":"1234"
}
```

### Response

```json
{
  "accessToken":"...",
  "refreshToken":"..."
}
```

---

# 토큰 재발급

## POST /api/auth/reissue

### Header

Authorization : Bearer RefreshToken

---

# 로그아웃

## POST /api/auth/logout