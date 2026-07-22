# Auth Database

## 개요

Auth Database는 인증/인가와 회원 프로필 정보를 관리하는 Auth Service 소유의 PostgreSQL
Database입니다. 이메일/비밀번호(LOCAL) 로그인과 구글 소셜 로그인(GOOGLE)을 모두 지원하며,
회원가입·로그인·탈퇴·휴면 전환에 필요한 데이터를 `users`/`oauth_user` 두 테이블로 관리합니다.

인증 수단(LOCAL/GOOGLE)은 `users`에 직접 컬럼으로 두지 않고 `oauth_user` 하위 테이블로
정규화했습니다. 한 사용자가 여러 소셜 계정을 연동하는 것을 스키마 차원에서 지원하기
위함이며(연동 기능 자체는 추후 개발 예정), LOCAL 계정은 `users`만 있고 `oauth_user` 행이
없는 상태로 표현합니다.

---

# ERD

```
users
──────────────────────────────────────────────
PK  id
    email            (UNIQUE)
    password         (nullable)
    status
    email_verified
    nickname         (UNIQUE)
    profile_image_url
    created_at
    updated_at
    last_login_at
    deleted_at       (nullable)

          │ 1
          │
          │
          ▼
oauth_user
──────────────────────────────────────────────
PK  id
FK  user_id
    provider
    provider_user_id
    created_at

    UNIQUE(provider, provider_user_id)
```

---

# Table

## users

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| email | VARCHAR(255) | X | 이메일, UNIQUE |
| password | VARCHAR(255) | O | BCrypt 해시. GOOGLE 전용 계정(연동된 `oauth_user`만 있고 비밀번호를 설정한 적 없는 경우)은 NULL |
| status | VARCHAR(20) | X | 계정 상태 (ACTIVE/DORMANT/DELETED), 기본값 ACTIVE |
| email_verified | BOOLEAN | X | 이메일 인증 완료 여부, 기본값 FALSE |
| nickname | VARCHAR(20) | X | 닉네임, UNIQUE |
| profile_image_url | VARCHAR(500) | O | 프로필 이미지 URL |
| created_at | DATETIME | X | 가입 일시 |
| updated_at | DATETIME | O | 정보 수정 일시 |
| last_login_at | DATETIME | O | 마지막 로그인 일시. 휴면 전환 판단 기준 |
| deleted_at | DATETIME | O | 탈퇴 처리 일시. `status = 'DELETED'`일 때만 값이 있음 |

LOCAL 가입자는 `password`가 반드시 있고 `oauth_user` 행이 없습니다. GOOGLE 최초 로그인 시
자동 생성되는 계정은 `password`가 NULL이고 `oauth_user`에 연동 행이 하나 생깁니다. 같은
`users` 행에 LOCAL 인증과 GOOGLE 연동이 동시에 존재하는 것(비밀번호도 있고 `oauth_user`도
있는 상태)도 허용됩니다 — 다만 "계정 연동" 플로우 자체는 추후 개발 예정이라, 현재는 이 상태가
자연스럽게 만들어지지는 않습니다.

회원 탈퇴는 Soft Delete로 처리합니다. `status`를 `'DELETED'`로 바꾸고 같은 시점에
`deleted_at`을 채우며, `users` 행 자체는 삭제하지 않습니다. 개인정보를 즉시 파기하지
않는다는 뜻이며, 계정 복구(탈퇴 취소) 기능은 추후 개발 예정입니다.

---

## oauth_user

| 컬럼명 | 타입 | NULL | 설명 |
|---------|------|------|------|
| id | BIGSERIAL | X | PK |
| user_id | BIGINT | X | FK, `users.id` |
| provider | VARCHAR(20) | X | 소셜 로그인 제공자 (예: GOOGLE) |
| provider_user_id | VARCHAR(255) | X | 제공자가 발급한 사용자 식별자 (구글의 `sub` 클레임) |
| created_at | DATETIME | X | 연동 일시 |

`UNIQUE(provider, provider_user_id)`로 같은 구글 계정이 두 명의 `users`에 중복 연동되는
것을 막습니다. `user_id`는 같은 DB 안의 실제 FK입니다. 한 사용자가 여러 provider를
연동하면 이 테이블에 여러 행이 쌓입니다(1:N).

---

# DDL

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,

    email VARCHAR(255) NOT NULL,

    password VARCHAR(255),

    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',

    email_verified BOOLEAN NOT NULL DEFAULT FALSE,

    nickname VARCHAR(20) NOT NULL,

    profile_image_url VARCHAR(500),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP,

    last_login_at TIMESTAMP,

    deleted_at TIMESTAMP,

    CONSTRAINT uk_users_email UNIQUE (email),

    CONSTRAINT uk_users_nickname UNIQUE (nickname),

    CONSTRAINT chk_users_status CHECK (status IN ('ACTIVE', 'DORMANT', 'DELETED')),

    CONSTRAINT chk_users_deleted_consistency
        CHECK ((status = 'DELETED') = (deleted_at IS NOT NULL))
);
```

```sql
CREATE TABLE oauth_user (
    id BIGSERIAL PRIMARY KEY,

    user_id BIGINT NOT NULL,

    provider VARCHAR(20) NOT NULL,

    provider_user_id VARCHAR(255) NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_oauth_user_users
        FOREIGN KEY (user_id) REFERENCES users(id),

    CONSTRAINT uk_oauth_user_provider UNIQUE (provider, provider_user_id)
);
```

---

# Index

```sql
CREATE INDEX idx_oauth_user_user
ON oauth_user(user_id);
```

한 사용자에게 연동된 provider 목록 조회에 사용합니다.

`email`/`nickname`은 UNIQUE 제약이 곧 인덱스 역할을 겸하므로 별도 인덱스를 추가하지
않습니다. `provider_user_id` 단독 조회는 없고 항상 `(provider, provider_user_id)` 쌍으로
조회하므로, UNIQUE 복합 인덱스 하나로 충분합니다.

---

# 관계

```
oauth_user (Auth Service)

    │ user_id (같은 DB 내 실제 FK)
    ▼
users
```

Auth Service는 다른 서비스와 데이터베이스를 공유하지 않습니다. 다른 서비스는 `userId`를
소프트 참조(FK 없는 BIGINT)로만 사용합니다.

---

# 고려 사항

- 휴면 전환은 별도 배치 없이 로그인 시도 시점에 판단합니다. 비밀번호 검증에 성공한 뒤
  `last_login_at` 기준으로 휴면 기준일(정책값, 예시 90일)을 넘었으면 `status`를 `DORMANT`로
  전환하고 이메일 재인증을 요구합니다. 최초 가입 후 한 번도 로그인하지 않아
  `last_login_at`이 NULL이면 휴면 판단에서 제외합니다.
- 탈퇴한 계정(`status = 'DELETED'`)은 로그인 자체가 거부됩니다.
- `password`가 NULL인 계정(GOOGLE 전용)은 이메일/비밀번호 로그인 경로를 사용할 수 없으며,
  반드시 `oauth_user`를 통한 로그인만 가능합니다. 이 정합성은 애플리케이션 레벨에서
  보장하며, DB 제약으로는 강제하지 않습니다(비밀번호 재설정 등으로 나중에 채워질 수
  있으므로).
- 역할 기반 접근 제어(관리자 전용 기능 등)는 아직 구현되어 있지 않아 `role` 컬럼을 두지
  않았습니다. 필요해지면 별도 테이블(예: `user_role`)로 추가하는 것을 권장합니다.
