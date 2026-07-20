# Auth Database

## 개요

Auth Database는 인증 정보와 회원 프로필 정보를 하나의 `users` 테이블로 관리합니다.

Auth Service와 User Service가 통합되기 전에는 `auth_user`(인증)와 `users`(프로필) 두 테이블로 나뉘어 있었고,
회원가입 시마다 두 서비스 간 OpenFeign 호출이 필요했습니다.

서비스 통합과 함께 두 테이블도 하나로 합쳐, 회원가입/조회/탈퇴가 모두 단일 트랜잭션으로 처리됩니다.

Refresh Token과 이메일 인증번호는 여전히 PostgreSQL이 아닌 Redis에서 관리합니다.

> ℹ️ **변경 이력**: 구글 소셜 로그인이 추가되면서 `provider`(LOCAL/GOOGLE) 컬럼이 생겼습니다.
> `password`는 GOOGLE 계정에는 없을 수 있어 NULL을 허용하도록 바뀌었습니다. 탈퇴 처리 방식도
> `deleted_at`(NULL 여부로 판단하는 소프트 삭제)에서 `status`(ACTIVE/DELETED) 컬럼 기반으로
> 바뀌었습니다. `role`(USER/ADMIN)은 관리자 전용 API(예: `mushroom_reference` 갱신,
> Embedding 관리 API)의 권한 판단에 계속 쓰이므로 그대로 유지합니다.

---

# ERD

```
users
──────────────────────────────────────────────
PK  id
    email
    password
    provider
    role
    status
    email_verified
    nickname
    profile_image_url
    created_at
    updated_at
```

---

# Table

## users

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| email | VARCHAR(255) | X | 로그인 이메일 |
| password | VARCHAR(255) | O | BCrypt 암호화 비밀번호 (provider=GOOGLE이면 NULL) |
| provider | VARCHAR(20) | X | 로그인 방식 (LOCAL, GOOGLE) |
| role | VARCHAR(30) | X | 사용자 권한 (USER, ADMIN) |
| status | VARCHAR(20) | X | 계정 상태 (ACTIVE, DELETED) |
| email_verified | BOOLEAN | X | 이메일 인증 여부 |
| nickname | VARCHAR(50) | X | 닉네임 |
| profile_image_url | VARCHAR(500) | O | 프로필 이미지 |
| created_at | TIMESTAMP | X | 생성일 |
| updated_at | TIMESTAMP | X | 수정일 |

`deleted_at`은 더 이상 사용하지 않습니다. 탈퇴 여부는 `status = 'DELETED'`로 판단하며, 탈퇴
시각이 필요하면 그 시점의 `updated_at`을 참고합니다(탈퇴 처리 시 `status`와 함께 `updated_at`도
갱신되므로).

---

# DDL

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,

    email VARCHAR(255) NOT NULL,

    password VARCHAR(255),

    provider VARCHAR(20) NOT NULL DEFAULT 'LOCAL',

    role VARCHAR(30) NOT NULL DEFAULT 'USER',

    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',

    email_verified BOOLEAN NOT NULL DEFAULT FALSE,

    nickname VARCHAR(50) NOT NULL,

    profile_image_url VARCHAR(500),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uk_users_email_provider UNIQUE (email, provider),

    CONSTRAINT uk_users_nickname UNIQUE (nickname),

    CONSTRAINT chk_users_provider CHECK (provider IN ('LOCAL', 'GOOGLE')),

    CONSTRAINT chk_users_status CHECK (status IN ('ACTIVE', 'DELETED')),

    CONSTRAINT chk_users_password CHECK (provider != 'LOCAL' OR password IS NOT NULL)
);
```

`email`은 더 이상 단독 `UNIQUE`가 아니라 `(email, provider)` 조합으로 유니크합니다. 같은
이메일이라도 LOCAL 계정과 GOOGLE 계정을 별개의 행으로 허용하기 위해서입니다(두 계정을 하나로
합치는 계정 연동 기능은 "추후 개발 예정"). `password`는 provider가 LOCAL일 때만 필수이며,
`chk_users_password` 제약으로 DB 레벨에서도 강제합니다.

---

# Index

```sql
CREATE UNIQUE INDEX uk_users_email_provider
ON users(email, provider);
```

```sql
CREATE UNIQUE INDEX uk_users_nickname
ON users(nickname);
```

이메일 하나로 로그인 시 사용자를 특정할 수 없는 경우(같은 이메일로 LOCAL/GOOGLE 계정이 모두
있는 경우)가 생길 수 있어, 로그인 시에는 이메일과 함께 provider도 함께 조회 조건에 사용합니다.

---

# 제약 조건

## Email

- NULL 불가
- 단독으로는 중복 불가하지 않으며, `(email, provider)` 조합으로 중복 불가

---

## Password

- BCrypt 암호화 저장
- provider가 LOCAL일 때만 필수, GOOGLE이면 NULL

---

## Provider

허용 값

- LOCAL (이메일/비밀번호 회원가입)
- GOOGLE (구글 소셜 로그인)

기본값은 LOCAL입니다.

---

## Role

허용 값

- USER
- ADMIN

---

## Status

허용 값

- ACTIVE (정상)
- DELETED (탈퇴, Soft Delete)

기본값은 ACTIVE입니다.

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

GOOGLE 계정은 구글이 이미 이메일을 검증했으므로, 최초 가입 시점에 바로 `true`로 생성됩니다.

---

## nickname

- NULL 불가
- 최대 50자
- 중복 불가

---

## profile_image_url

- 선택 입력
- 이미지 URL 저장
- GOOGLE 계정은 최초 가입 시 구글 프로필 이미지 URL을 기본값으로 사용할 수 있습니다.

---

# Soft Delete

회원 탈퇴 시 데이터를 즉시 삭제하지 않습니다.

`status` 컬럼을 이용하여 Soft Delete를 수행합니다.

예시

```
status = 'ACTIVE'

→ 정상 사용자
```

```
status = 'DELETED' (updated_at = 2026-08-15 13:30)

→ 탈퇴한 사용자, 탈퇴 시각은 updated_at 참고
```

> ℹ️ **변경 이력**: 이전에는 `deleted_at` 컬럼(NULL 여부로 판단)을 사용했지만, 상태를
> ACTIVE/DELETED로 명시적으로 표현하는 `status` 컬럼으로 바꿨습니다. 탈퇴 시각이 별도
> 컬럼으로 남지 않는 대신, 탈퇴 처리 시점에 `updated_at`도 함께 갱신되므로 이 값으로 대체합니다.

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

- 비밀번호는 BCrypt로 암호화하여 저장하며, provider가 LOCAL일 때만 존재합니다.
- Refresh Token/이메일 인증번호는 PostgreSQL이 아닌 Redis에서 관리합니다.
- 회원 탈퇴는 `status = 'DELETED'`로 표현하는 Soft Delete를 사용합니다.
- 인증 정보와 프로필 정보가 같은 테이블에 있으므로, 회원가입/수정/탈퇴가 서비스 간 호출 없이 하나의 트랜잭션으로 처리됩니다.
- 향후 프로필 이미지 저장소(MinIO 등)와 연동할 수 있도록 URL만 저장합니다.
- LOCAL 계정과 GOOGLE 계정은 이메일이 같아도 서로 다른 행으로 존재할 수 있습니다(`(email, provider)` 조합이 유니크). 같은 사람의 두 계정을 하나로 합치는 계정 연동은 아직 지원하지 않습니다.
- `role`(USER/ADMIN)은 계정 상태(`status`)나 로그인 방식(`provider`)과 독립적인 값입니다. 관리자 전용 API(mushroom_reference 갱신 등) 권한 판단에 사용됩니다.
