# 회원가입 시퀀스

## 개요

사용자가 이메일/비밀번호(LOCAL)로 회원가입을 진행하는 과정입니다. Auth Service가 이메일
인증부터 `users` 생성까지 단일 트랜잭션으로 처리하며, 다른 서비스를 호출하지 않습니다.

이 문서는 LOCAL 회원가입만 다룹니다. 구글 소셜 로그인은 별도의 회원가입 절차 없이 최초
로그인 시 자동으로 계정이 생성됩니다. 자세한 내용은 [login.md](./login.md)의 "구글 로그인"
참고.

---

# Sequence

```text
Client
↓
API Gateway
↓
Auth Service
↓
이메일 중복 확인 (PostgreSQL)
↓
인증번호 생성 → Redis 저장
↓
Email 발송
↓
사용자 입력
↓
인증번호 확인 (Redis)
↓
users 생성 (PostgreSQL, 단일 트랜잭션)
↓
회원가입 완료
```

---

# 상세 과정

## 1. 회원가입 요청

```http
POST /api/v1/auth/signup
```

```json
{
    "email": "test@test.com",
    "password": "P@ssw0rd!",
    "nickname": "버섯초보"
}
```

---

## 2. 이메일/닉네임 중복 검사

`users` 테이블에서 `email`, `nickname` UNIQUE 제약 위반 여부를 확인합니다.

---

## 3. 인증번호 생성

랜덤 6자리를 생성해 Redis에 저장합니다.

```
Key: email:test@test.com
Value: 845231
TTL: 5분
```

---

## 4. 이메일 발송

SMTP로 인증번호를 발송합니다.

---

## 5. 인증번호 입력/검증

```http
POST /api/v1/auth/verify-email
```

```json
{ "email": "test@test.com", "code": "845231" }
```

Auth Service가 Redis에 저장된 값과 비교해 검증합니다.

---

## 6. 회원 생성

검증에 성공하면 `users` 행을 생성합니다(`email`, `password`(BCrypt 해시), `email_verified
= true`, `nickname`, `status = 'ACTIVE'`). LOCAL 가입은 `oauth_user` 행을 만들지 않습니다.

---

## 7. 완료

```json
{ "message": "회원가입이 완료되었습니다." }
```

---

# 사용 Database

## PostgreSQL

```
users (Auth DB)
```

## Redis

```
email:{email}
```

---

# OpenFeign

사용하지 않습니다. 다른 서비스를 호출하지 않고 Auth Service 내부에서 모두 처리합니다.

---

# Event

없음. 회원가입은 동기 처리합니다.

---

# 예외 상황

- 이메일 중복 / 닉네임 중복
- 인증번호 만료 / 불일치
- DB 저장 실패
