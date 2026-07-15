# Auth Database

## 개요

Auth Database는 인증 정보와 회원 프로필 정보를 하나의 `users` 테이블로 관리합니다.

Auth Service와 User Service가 통합되기 전에는 `auth_user`(인증)와 `users`(프로필) 두 테이블로 나뉘어 있었고,
회원가입 시마다 두 서비스 간 OpenFeign 호출이 필요했습니다.

서비스 통합과 함께 두 테이블도 하나로 합쳐, 회원가입/조회/탈퇴가 모두 단일 트랜잭션으로 처리됩니다.

Refresh Token과 이메일 인증번호는 여전히 PostgreSQL이 아닌 Redis에서 관리합니다.

---

# ERD

```
users
──────────────────────────────────────────────
PK  id
    email
    password
    role
    email_verified
    nickname
    profile_image_url
    created_at
    updated_at
    deleted_at
```

---

# Table

## users

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| email | VARCHAR(255) | X | 로그인 이메일 |
| password | VARCHAR(255) | X | BCrypt 암호화 비밀번호 |
| role | VARCHAR(30) | X | 사용자 권한 |
| email_verified | BOOLEAN | X | 이메일 인증 여부 |
| nickname | VARCHAR(50) | X | 닉네임 |
| profile_image_url | VARCHAR(500) | O | 프로필 이미지 |
| created_at | TIMESTAMP | X | 생성일 |
| updated_at | TIMESTAMP | X | 수정일 |
| deleted_at | TIMESTAMP | O | 탈퇴일 (Soft Delete) |

---

# DDL

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,

    email VARCHAR(255) NOT NULL UNIQUE,

    password VARCHAR(255) NOT NULL,

    role VARCHAR(30) NOT NULL DEFAULT 'USER',

    email_verified BOOLEAN NOT NULL DEFAULT FALSE,

    nickname VARCHAR(50) NOT NULL,

    profile_image_url VARCHAR(500),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    deleted_at TIMESTAMP
);
```

---

# Index

```sql
CREATE UNIQUE INDEX uk_users_email
ON users(email);
```

```sql
CREATE INDEX idx_users_nickname
ON users(nickname);
```

---

# 제약 조건

## Email

- 중복 불가
- NULL 불가

---

## Password

- BCrypt 암호화 저장

---

## Role

허용 값

- USER
- ADMIN

---

## Email Verified

초기값

```
false
```

이메일 인증 완료 시

```
true
```

---

## nickname

- NULL 불가
- 최대 50자

---

## profile_image_url

- 선택 입력
- 이미지 URL 저장

---

# Soft Delete

회원 탈퇴 시 데이터를 즉시 삭제하지 않습니다.

deleted_at 컬럼을 이용하여 Soft Delete를 수행합니다.

예시

```
deleted_at = NULL

→ 정상 사용자
```

```
deleted_at = 2026-08-15 13:30

→ 탈퇴한 사용자
```

---

# Redis

## Refresh Token

Key

```
refresh:{userId}
```

Value

```
JWT Refresh Token
```

TTL

```
14일
```

---

## Email Verification

Key

```
email:{email}
```

Value

```
123456
```

TTL

```
5분
```

---

# 관계

```
users (Auth Service)

    │ userId

    ▼

Cultivation Service (userId로 재배 소유자 식별)
```

Auth Service는 다른 서비스와 데이터베이스를 공유하지 않으며, userId를 통해서만 참조됩니다.

---

# 다른 서비스와의 연관

## Cultivation Service

재배 생성 시 `userId`를 이용하여 재배 소유자를 식별합니다.

---

## AI Service / Notification Service

직접 연결되지 않습니다.

---

# 고려 사항

- 비밀번호는 BCrypt로 암호화하여 저장합니다.
- Refresh Token/이메일 인증번호는 PostgreSQL이 아닌 Redis에서 관리합니다.
- 회원 탈퇴는 Soft Delete를 사용합니다.
- 인증 정보와 프로필 정보가 같은 테이블에 있으므로, 회원가입/수정/탈퇴가 서비스 간 호출 없이 하나의 트랜잭션으로 처리됩니다.
- 향후 프로필 이미지 저장소(MinIO 등)와 연동할 수 있도록 URL만 저장합니다.
