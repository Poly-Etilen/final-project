# 회원가입 시퀀스

## 개요

사용자가 이메일 회원가입을 진행하는 과정입니다.

Auth Service(기존 Auth+User 통합)가 이메일 인증부터 사용자 생성까지 단일 트랜잭션으로 처리합니다.
(기존에는 Auth Service가 auth_user 생성 후 User Service를 OpenFeign으로 호출해 프로필을 별도 생성했으나,
서비스 통합으로 서비스 간 호출 없이 하나의 트랜잭션으로 처리합니다.)

> ℹ️ **변경 이력**: 이 문서는 이메일/비밀번호(LOCAL) 회원가입만 다룹니다. 구글 소셜 로그인은
> 별도의 회원가입 절차 없이 최초 로그인 시 자동으로 계정이 생성됩니다. 자세한 내용은
> [login.md](./login.md)의 "구글 로그인" 참고.

---

# Sequence

```
Client

↓

API Gateway

↓

Auth Service

↓

이메일 중복 확인

↓

Redis

↓

인증번호 생성

↓

Email 발송

↓

사용자 입력

↓

인증번호 확인

↓

Redis

↓

회원 생성 (users, 단일 트랜잭션)

↓

PostgreSQL(Auth)

↓

회원가입 완료
```

---

# 상세 과정

## 1. 이메일 입력

사용자가 이메일과 비밀번호를 입력합니다.

↓

Auth Service

---

## 2. 이메일 중복 검사

PostgreSQL(Auth)

↓

`users` 테이블에서 중복 여부 확인

---

## 3. 인증번호 생성

랜덤 6자리 생성

↓

Redis 저장

```
email:test@test.com

845231
```

TTL

```
5분
```

---

## 4. 이메일 발송

SMTP

↓

사용자

---

## 5. 인증번호 입력

사용자

↓

Auth Service

↓

Redis 검증

---

## 6. 회원 생성

예시

```http
POST /auth/signup
```

```json
{
    "email":"test@test.com",
    "password":"P@ssw0rd!",
    "nickname":"버섯초보"
}
```

↓

`users` 생성 (email, password, provider='LOCAL', role, email_verified, nickname 등 전체 프로필 컬럼 포함)

↓

PostgreSQL 저장 (단일 트랜잭션, 서비스 간 호출 없음)

---

## 7. 완료

회원가입 성공

---

# 사용 Database

## PostgreSQL

Auth

```
users
```

---

## Redis

```
email:{email}
```

---

# OpenFeign

사용하지 않습니다.

기존에는 Auth → User Service 호출이 있었으나, 서비스 통합으로 내부 처리로 단순화되었습니다.

---

# Event

없음

회원가입은 동기 처리합니다.

---

# 예외 상황

- 이메일 중복
- 인증번호 만료
- 인증번호 불일치
- DB 저장 실패
