```sql
CREATE TABLE auth_user (
    auth_user_id BIGSERIAL PRIMARY KEY,

    email VARCHAR(255) NOT NULL UNIQUE,

    password VARCHAR(255),

    provider VARCHAR(20) NOT NULL DEFAULT 'LOCAL',

    provider_user_id VARCHAR(255),

    email_verified BOOLEAN NOT NULL DEFAULT FALSE,

    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## Auth Service가 관리하는 데이터
### PostgreSQL
* auth_user

### Redis
* Refresh Token
* 이메일 인증 코드

### 회원가입 흐름
```text
회원가입 요청

↓

이메일 입력

↓

Redis
email_verification:{email}
↓

인증번호 저장(TTL 5분)

↓

사용자가 인증번호 입력

↓

Redis 인증번호 비교

↓

일치

↓

Auth Service

↓

auth_user 생성

↓

OpenFeign

↓

User Service

↓

users 생성

↓

회원가입 완료
```

### 로그인 흐름
```text
로그인

↓

Auth Service

↓

auth_user 조회

↓

비밀번호 검증

↓

Access Token 생성

↓

Refresh Token 생성

↓

Redis 저장

↓

Client
```

### OAuth 로그인
```text
Google OAuth

↓

Auth Service

↓

provider = GOOGLE

↓

provider_user_id 저장

↓

auth_user 생성

↓

OpenFeign

↓

User Service

↓

users 생성
```