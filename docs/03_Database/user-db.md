# User Database

## 개요

User Database는 사용자의 프로필 및 서비스 이용 정보를 저장합니다.

인증(Authentication)과 관련된 정보는 Auth Service에서 관리하며,
User Database는 사용자 프로필만 관리합니다.

---

# ERD

```
user
──────────────────────────────────────────────────────────────
PK  id
FK  auth_user_id
    nickname
    profile_image_url
    created_at
    updated_at
    deleted_at
```

---

# Table

## user

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| auth_user_id | BIGINT | X | Auth Service 사용자 ID |
| nickname | VARCHAR(50) | X | 닉네임 |
| profile_image_url | VARCHAR(500) | O | 프로필 이미지 |
| created_at | TIMESTAMP | X | 생성일 |
| updated_at | TIMESTAMP | X | 수정일 |
| deleted_at | TIMESTAMP | O | 탈퇴일(Soft Delete) |

---

# DDL

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,

    auth_user_id BIGINT NOT NULL,

    nickname VARCHAR(50) NOT NULL,

    profile_image_url VARCHAR(500),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    deleted_at TIMESTAMP
);
```

---

# Index

닉네임 검색

```sql
CREATE INDEX idx_user_nickname
ON users(nickname);
```

Auth 사용자 조회

```sql
CREATE UNIQUE INDEX uk_user_auth
ON users(auth_user_id);
```

---

# 제약 조건

## auth_user_id

- 중복 불가
- Auth Service 사용자와 1:1 관계

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

# 관계

```
Auth DB

auth_user
      │
      │ auth_user_id
      ▼

User DB

users
```

User Service는 Auth Service와 auth_user_id를 통해 연결됩니다.

---

# 다른 서비스와의 연관

## Cultivation Service

재배 생성 시

```
userId
```

를 이용하여 재배 소유자를 식별합니다.

---

## AI Service

직접 연결되지 않습니다.

---

## Notification Service

직접 연결되지 않습니다.

---

# 고려 사항

- 사용자 인증 정보는 저장하지 않습니다.
- 이메일은 Auth Service에서 관리합니다.
- 비밀번호는 Auth Service에서 관리합니다.
- 회원 탈퇴는 Soft Delete를 사용합니다.
- 향후 프로필 이미지 저장소(S3, MinIO 등)와 연동할 수 있도록 URL만 저장합니다.