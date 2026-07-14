# Auth Service

## 역할

Auth Service는 사용자의 인증(Authentication) 및 인가(Authorization)를 담당하는 서비스입니다.

JWT 기반 인증을 제공하며, 로그인 및 토큰 재발급을 처리합니다.

---

# 책임

- 회원가입
- 로그인
- 로그아웃
- JWT 발급
- Access Token 검증
- Refresh Token 관리
- 이메일 인증

---

# 주요 기능

## 회원가입

사용자는 이메일을 통해 회원가입을 진행합니다.

이메일 인증이 완료되어야 회원가입이 가능합니다.

---

## 로그인

이메일과 비밀번호를 검증한 후 JWT를 발급합니다.

---

## JWT 인증

Access Token을 검증합니다.

---

## Refresh Token

Redis에 저장된 Refresh Token을 이용하여 Access Token을 재발급합니다.

---

## 이메일 인증

회원가입 시 이메일 인증번호를 발송하고 검증합니다.

---

# API

## 회원가입

POST /auth/signup

---

## 로그인

POST /auth/login

---

## 로그아웃

POST /auth/logout

---

## Access Token 재발급

POST /auth/refresh

---

## 이메일 인증 요청

POST /auth/email/send

---

## 이메일 인증 확인

POST /auth/email/verify

---

# Database

Auth Service는 별도의 PostgreSQL Database를 사용합니다.

### Table

- auth_user

---

# Redis

### Refresh Token

Key

```
refresh:{userId}
```

Value

```
Refresh Token
```

---

### 이메일 인증

Key

```
email:{email}
```

Value

```
인증번호
```

---

# 다른 서비스와의 통신

## 호출하는 서비스

없음

---

## 호출받는 서비스

API Gateway

---

# Event

현재 없음

추후 로그인 이벤트 발행 가능

---

# Sequence

로그인

Client

↓

Gateway

↓

Auth Service

↓

JWT 발급

↓

Client

---

# 예외 상황

- 존재하지 않는 이메일
- 비밀번호 불일치
- 만료된 Access Token
- 만료된 Refresh Token
- 이메일 인증 실패

---

# 추후 개발 예정

- Google OAuth2 로그인
- Kakao 로그인
- Naver 로그인
- 2차 인증(MFA)