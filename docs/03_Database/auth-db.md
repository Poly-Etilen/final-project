# Auth Database

## 개요

Auth Database는 인증 정보와 회원 프로필 정보를 하나의 `users` 테이블로 관리합니다.

Auth Service와 User Service가 통합되기 전에는 `auth_user`(인증)와 `users`(프로필) 두 테이블로 나뉘어 있었고,
회원가입 시마다 두 서비스 간 OpenFeign 호출이 필요했습니다.

서비스 통합과 함께 두 테이블도 하나로 합쳐, 회원가입/조회/탈퇴가 모두 단일 트랜잭션으로 처리됩니다.

Refresh Token과 이메일 인증번호는 여전히 PostgreSQL이 아닌 Redis에서 관리합니다.

> ℹ️ **변경 이력**: 구글 소셜 로그인이 추가되면서 `provider`(LOCAL/GOOGLE) 컬럼이 생겼습니다.
> `password`는 GOOGLE 계정에는 없을 수 있어 NULL을 허용하도록 바뀌었습니다. 탈퇴 여부를 명시적으로
> 표현하기 위해 `status`(ACTIVE/DELETED) 컬럼도 함께 도입했습니다. `role`(USER/ADMIN)은 관리자
> 전용 API(예: `mushroom_reference` 갱신, Embedding 관리 API)의 권한 판단에 계속 쓰이므로 그대로
> 유지합니다.

> ℹ️ **변경 이력**: `status` 도입 초기에는 `deleted_at` 컬럼을 제거했으나, 다시 복원했습니다.
> 탈퇴 여부 자체는 `status = 'DELETED'`로 판단할 수 있지만, "탈퇴 취소(계정 복구)" 같은 향후
> 기능은 정확한 탈퇴 시점이 있어야 유예기간을 계산할 수 있습니다. `updated_at`은 탈퇴 이외의
> 이유로도 바뀔 수 있어 대체 지표로 쓰기 부적절합니다. 그래서 `status`(상태 구분)와 `deleted_at`
> (정확한 탈퇴 시각)을 함께 두고, `CHECK ((status = 'DELETED') = (deleted_at IS NOT NULL))`
> 제약으로 두 컬럼이 항상 일치하도록 강제합니다.

> ℹ️ **변경 이력**: 휴면 계정(`DORMANT`) 정책이 추가되었습니다. 이메일 인증은 회원가입 시점에
> 이미 필수이므로 별도의 "인증 대기" 상태는 두지 않지만, 장기간 로그인하지 않은 계정은 휴면으로
> 전환됩니다. 별도 배치 작업 없이 로그인 시도 시점에 `last_login_at` 기준으로 판단하며, 휴면
> 상태는 이메일 재인증을 거쳐야 다시 `ACTIVE`로 전환됩니다. 자세한 흐름은
> [login.md](../04_sequence/login.md)의 "휴면 계정 판단 및 재활성화" 참고.

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
    last_login_at
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
| password | VARCHAR(255) | O | BCrypt 암호화 비밀번호 (provider=GOOGLE이면 NULL) |
| provider | VARCHAR(20) | X | 로그인 방식 (LOCAL, GOOGLE) |
| role | VARCHAR(30) | X | 사용자 권한 (USER, ADMIN) |
| status | VARCHAR(20) | X | 계정 상태 (ACTIVE, DORMANT, DELETED) |
| email_verified | BOOLEAN | X | 이메일 인증 여부 |
| nickname | VARCHAR(50) | X | 닉네임 |
| profile_image_url | VARCHAR(500) | O | 프로필 이미지 |
| last_login_at | TIMESTAMP | O | 마지막 로그인 시각 (휴면 판단 기준, 최초 가입 후 로그인 전이면 NULL) |
| created_at | TIMESTAMP | X | 생성일 |
| updated_at | TIMESTAMP | X | 수정일 |
| deleted_at | TIMESTAMP | O | 탈퇴일 (Soft Delete, 정상 사용자는 NULL) |

탈퇴 여부는 `status = 'DELETED'`로 조회/필터링하고, 정확한 탈퇴 시점은 `deleted_at`으로
확인합니다. `updated_at`은 탈퇴 이외의 이유로도 바뀔 수 있어 탈퇴 시점 판단에는 쓰지 않습니다.
`deleted_at`은 향후 탈퇴 취소(계정 복구) 기능에서 유예기간 계산에 사용할 수 있습니다(추후 개발
예정).

휴면 여부는 `status = 'DORMANT'`로 판단합니다. 별도의 `dormant_at` 컬럼은 두지 않는데, 휴면
전환은 배치가 아니라 로그인 시도 시점에 `last_login_at`을 기준으로 그 자리에서 판단하기
때문에 `last_login_at` 자체가 곧 휴면 전환 근거가 됩니다.

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

    last_login_at TIMESTAMP,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    deleted_at TIMESTAMP,

    CONSTRAINT uk_users_email_provider UNIQUE (email, provider),

    CONSTRAINT uk_users_nickname UNIQUE (nickname),

    CONSTRAINT chk_users_provider CHECK (provider IN ('LOCAL', 'GOOGLE')),

    CONSTRAINT chk_users_status CHECK (status IN ('ACTIVE', 'DORMANT', 'DELETED')),

    CONSTRAINT chk_users_password CHECK (provider != 'LOCAL' OR password IS NOT NULL),

    CONSTRAINT chk_users_status_deleted_at CHECK ((status = 'DELETED') = (deleted_at IS NOT NULL))
);
```

`email`은 더 이상 단독 `UNIQUE`가 아니라 `(email, provider)` 조합으로 유니크합니다. 같은
이메일이라도 LOCAL 계정과 GOOGLE 계정을 별개의 행으로 허용하기 위해서입니다(두 계정을 하나로
합치는 계정 연동 기능은 "추후 개발 예정"). `password`는 provider가 LOCAL일 때만 필수이며,
`chk_users_password` 제약으로 DB 레벨에서도 강제합니다. `chk_users_status_deleted_at`은
`status`와 `deleted_at`이 서로 어긋나지 않도록(탈퇴 상태인데 탈퇴 시각이 없거나, 탈퇴 시각은
있는데 상태가 탈퇴가 아닌 경우를 방지) 강제합니다.

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
- DORMANT (휴면, 장기 미로그인)
- DELETED (탈퇴, Soft Delete)

기본값은 ACTIVE입니다.

DORMANT는 별도 배치 없이 로그인 시도 시점에 `last_login_at` 기준으로 그 자리에서 판단합니다.
휴면 계정은 이메일 재인증을 거쳐야 ACTIVE로 돌아갑니다. 자세한 흐름은
[login.md](../04_sequence/login.md) 참고.

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

## last_login_at

- 로그인 성공 시마다 현재 시각으로 갱신합니다.
- 회원가입만 하고 한 번도 로그인하지 않은 경우 NULL입니다.
- 휴면 전환 여부를 판단하는 유일한 기준입니다.

---

# Soft Delete

회원 탈퇴 시 데이터를 즉시 삭제하지 않습니다.

`status`를 `'DELETED'`로 바꾸고, 동시에 `deleted_at`에 탈퇴 처리 시각을 기록합니다.

예시

```
status = 'ACTIVE'
deleted_at = NULL

→ 정상 사용자
```

```
status = 'DELETED'
deleted_at = 2026-08-15T13:30:00

→ 탈퇴한 사용자, 탈퇴 시각은 deleted_at 참고 (updated_at은 다른 이유로도 바뀔 수 있어 사용하지 않음)
```

탈퇴 여부를 조회/필터링할 때는 `status`를, 정확한 탈퇴 시점이 필요할 때는 `deleted_at`을
사용합니다. `deleted_at`은 향후 탈퇴 취소(계정 복구) 기능의 유예기간 계산에 쓰일 수 있습니다
(추후 개발 예정).

---

# 휴면 계정 (Dormant)

이메일 인증은 회원가입 시점에 이미 필수이므로 "인증 대기" 상태는 따로 두지 않습니다. 대신 장기간
로그인하지 않은 계정을 휴면(`DORMANT`)으로 전환하여, 오래 방치된 계정의 보안 위험을 줄입니다.

전환 시점은 별도 배치가 아니라 **로그인 시도 시점**입니다. 사용자가 로그인을 시도하면 비밀번호
검증 이후 `last_login_at`과 현재 시각의 차이를 휴면 기준일과 비교하고, 기준을 넘었다면 그 자리에서
`status`를 `'DORMANT'`로 전환한 뒤 정상 로그인 대신 이메일 재인증을 요구합니다.

예시

```
status = 'ACTIVE'
last_login_at = 2025-07-01T09:00:00

→ (로그인 시도 시점 기준일 초과) status = 'DORMANT'로 전환, 이메일 인증번호 발송
```

재인증(회원가입 때와 동일한 6자리 인증번호, Redis TTL 5분)을 완료하면 `status`가 다시
`'ACTIVE'`로 바뀌고, 처음 시도했던 로그인이 그대로 완료되어 JWT가 발급됩니다. 비밀번호는 최초
로그인 시도에서 이미 검증되었으므로 재인증 단계에서 다시 입력받지 않습니다.

기준일(며칠간 미로그인 시 휴면 전환할지)은 정책값으로, 문서상 예시는 90일이며 운영 중 조정될 수
있습니다. 자세한 흐름은 [login.md](../04_sequence/login.md)의 "휴면 계정 판단 및 재활성화"
참고.

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
- 회원 탈퇴는 `status = 'DELETED'` + `deleted_at`(정확한 탈퇴 시각)으로 표현하는 Soft Delete를 사용합니다. 두 컬럼은 `chk_users_status_deleted_at` 제약으로 항상 일치합니다.
- 장기간 미로그인 계정은 `status = 'DORMANT'`로 표현하며, 로그인 시도 시점에 `last_login_at` 기준으로 판단합니다(별도 배치 없음). 재활성화는 이메일 재인증으로만 가능합니다.
- `status`는 ACTIVE/DORMANT/DELETED 세 값을 가지므로 `deleted_at` 하나로는 상태를 전부 표현할 수 없어, `status`(상태 구분)와 `deleted_at`(탈퇴 시각)을 함께 둡니다.
- 인증 정보와 프로필 정보가 같은 테이블에 있으므로, 회원가입/수정/탈퇴가 서비스 간 호출 없이 하나의 트랜잭션으로 처리됩니다.
- 향후 프로필 이미지 저장소(MinIO 등)와 연동할 수 있도록 URL만 저장합니다.
- LOCAL 계정과 GOOGLE 계정은 이메일이 같아도 서로 다른 행으로 존재할 수 있습니다(`(email, provider)` 조합이 유니크). 같은 사람의 두 계정을 하나로 합치는 계정 연동은 아직 지원하지 않습니다.
- `role`(USER/ADMIN)은 계정 상태(`status`)나 로그인 방식(`provider`)과 독립적인 값입니다. 관리자 전용 API(mushroom_reference 갱신 등) 권한 판단에 사용됩니다.
