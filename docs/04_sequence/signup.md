# 회원가입 시퀀스

## 개요

사용자가 이메일 회원가입을 진행하는 과정입니다.

회원가입은 Auth Service와 User Service가 협력하여 처리합니다.

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

회원 생성

↓

PostgreSQL(Auth)

↓

User Service 호출

↓

사용자 프로필 생성

↓

PostgreSQL(User)

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

중복 여부 확인

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

auth_user 생성

↓

PostgreSQL

---

## 7. User Service 호출

OpenFeign

↓

User Service

↓

users 생성

---

## 8. 완료

회원가입 성공

---

# 사용 Database

## PostgreSQL

Auth

```
auth_user
```

User

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

```
Auth

↓

User
```

---

# Event

없음

회원가입은 동기 처리합니다.

---

# 예외 상황

- 이메일 중복
- 인증번호 만료
- 인증번호 불일치
- User Service 호출 실패
- DB 저장 실패