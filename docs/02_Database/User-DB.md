# User Database

## 개요

User Service는 사용자 정보를 관리한다.

인증(Auth Service)과 분리되어 있으며 개인정보만 관리한다.

---

# users

```sql
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,

    email VARCHAR(255) UNIQUE NOT NULL,

    nickname VARCHAR(50) NOT NULL,

    profile_image VARCHAR(500),

    created_at TIMESTAMP,

    updated_at TIMESTAMP
);
```

---

# oauth_account

```sql
CREATE TABLE oauth_account (

    oauth_account_id BIGSERIAL PRIMARY KEY,

    user_id BIGINT,

    provider VARCHAR(20),

    provider_user_id VARCHAR(255),

    created_at TIMESTAMP
);
```

---

# 관계

```
User

1:N

OAuthAccount
```

---

# 비고

비밀번호 및 Refresh Token은 Auth Service에서 관리한다.