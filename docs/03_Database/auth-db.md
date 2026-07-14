# Auth Database

## 개요

Auth Database는 사용자 인증(Authentication)을 위한 정보를 저장합니다.

인증과 관련된 최소한의 정보만 저장하며,
Refresh Token과 이메일 인증번호는 Redis에서 관리합니다.

---

# ERD

```
auth_user
──────────────────────────────────────────────
PK  id
    email
    password
    role
    email_verified
    created_at
    updated_at
```

---

# Table

## auth_user

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| email | VARCHAR(255) | X | 로그인 이메일 |
| password | VARCHAR(255) | X | BCrypt 암호화 비밀번호 |
| role | VARCHAR(30) | X | 사용자 권한 |
| email_verified | BOOLEAN | X | 이메일 인증 여부 |
| created_at | TIMESTAMP | X | 생성일 |
| updated_at | TIMESTAMP | X | 수정일 |

---

# DDL

```sql
CREATE TABLE auth_user (
    id BIGSERIAL PRIMARY KEY,

    email VARCHAR(255) NOT NULL UNIQUE,

    password VARCHAR(255) NOT NULL,

    role VARCHAR(30) NOT NULL DEFAULT 'USER',

    email_verified BOOLEAN NOT NULL DEFAULT FALSE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

# Index

```sql
CREATE UNIQUE INDEX uk_auth_user_email
ON auth_user(email);
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

Auth Service는 다른 서비스와 데이터베이스를 공유하지 않습니다.

User Service와는 userId를 기준으로만 연관됩니다.

```
Auth DB

auth_user
     │
     │ userId
     ▼
User DB

user
```

---

# 고려 사항

- 비밀번호는 BCrypt로 암호화하여 저장합니다.
- Refresh Token은 PostgreSQL이 아닌 Redis에서 관리합니다.
- 이메일 인증번호는 Redis TTL을 이용하여 자동 만료됩니다.
- 사용자 프로필 정보는 User Service에서 관리합니다.