# User Database

## 개요

User Service는 사용자 정보를 관리한다.

인증(Auth Service)과 분리되어 있으며 개인정보만 관리한다.

---

# users

```sql
CREATE TABLE users (
                       user_id BIGSERIAL PRIMARY KEY,

                       auth_user_id BIGINT NOT NULL UNIQUE,

                       nickname VARCHAR(50) NOT NULL UNIQUE,

                       profile_image_url VARCHAR(500),

                       created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

                       updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

# 흐름

### 회원가입
```text
Client
      │
      ▼
Gateway
      │
      ▼
Auth Service
      │
      ├── auth_user 생성
      │
      └── OpenFeign
             │
             ▼
       User Service
             │
             ▼
         users 생성
```

### 내 정보 조회
```text
JWT

↓

Gateway

↓

User Service

↓

auth_user_id 추출

↓

users 조회

↓

닉네임 반환
```

