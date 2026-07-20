# 로그인 시퀀스

## 개요

사용자가 이메일과 비밀번호를 이용하여 로그인하는 과정입니다.

로그인이 성공하면 Access Token과 Refresh Token을 발급합니다.

Access Token은 Client에 전달되며,
Refresh Token은 Redis에 저장하여 관리합니다.

> ℹ️ **변경 이력**: 이메일/비밀번호(LOCAL) 로그인 외에 구글 소셜 로그인(GOOGLE)이
> 추가되었습니다. 아래는 LOCAL 로그인 흐름이며, 구글 로그인은 "구글 로그인" 섹션을 참고하세요.

---

# Sequence

```
Client

↓

API Gateway

↓

Auth Service

↓

이메일 조회

↓

PostgreSQL

↓

비밀번호 검증

↓

BCrypt

↓

JWT 생성

├── Access Token
└── Refresh Token

↓

Redis 저장

↓

Client

↓

로그인 완료
```

---

# 상세 과정

## 1. 로그인 요청

사용자가

- 이메일
- 비밀번호

를 입력합니다.

↓

Auth Service

---

## 2. 사용자 조회

```
users
```

조회

↓

사용자 존재 여부 확인

---

## 3. 비밀번호 검증

BCrypt 비교

```
PasswordEncoder.matches()
```

↓

성공

↓

JWT 생성

---

## 4. JWT 생성

생성 항목

### Access Token

포함 정보

- userId
- role

유효시간

```
30분
```

---

### Refresh Token

유효시간

```
14일
```

---

## 5. Refresh Token 저장

Redis

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

## 6. 응답

Client에게

```json
{
  "accessToken": "...",
  "refreshToken": "...",
  "expiresIn": 1800
}
```

전달

---

# Access Token 사용

Client

↓

API Gateway

↓

JWT 검증

↓

각 Service 호출

---

# Refresh Token 재발급

Client

↓

Auth Service

↓

Redis 조회

↓

Refresh Token 비교

↓

새로운 Access Token 발급

↓

Client

---

# 로그아웃

Client

↓

Auth Service

↓

Redis

↓

Refresh Token 삭제

↓

로그아웃 완료

---

# 구글 로그인

```
Client (구글 로그인 완료, ID Token 보유)

↓

API Gateway

↓

Auth Service

↓

구글 공개키로 ID Token 서명/만료 검증

↓

이메일/이름/프로필 이미지 추출

↓

users에서 (email, provider='GOOGLE') 조회

├── 존재함 → 바로 다음 단계
└── 존재하지 않음 → users 자동 생성
      (provider='GOOGLE', password=NULL, email_verified=true,
       nickname/profile_image_url은 구글 프로필 기본값)

↓

JWT 생성 (Access Token + Refresh Token)

↓

Redis 저장 (refresh:{userId})

↓

Client
```

이메일/비밀번호 검증(BCrypt) 단계가 없다는 점을 제외하면, JWT 생성/Refresh Token 저장/응답
형식은 LOCAL 로그인과 동일합니다. 최초 로그인 시 자동 회원가입까지 한 번에 처리되므로, 별도의
"구글 회원가입" API는 없습니다.

같은 이메일로 LOCAL 계정이 이미 있어도 조회 조건이 `(email, provider)`이기 때문에 GOOGLE
계정은 별도로 생성됩니다. 두 계정을 하나로 합치는 기능은 아직 없습니다.

---

# 사용 Database

## PostgreSQL

```
users
```

---

## Redis

```
refresh:{userId}
```

---

# OpenFeign

서비스 간(내부 MSA) 호출은 사용하지 않습니다.

구글 로그인 시 ID Token 검증을 위해 구글의 공개키 엔드포인트를 외부 HTTP 호출하지만, 이는
내부 서비스 간 OpenFeign 호출이 아닙니다.

---

# Event

없음

로그인은 동기 처리합니다.

---

# 예외 상황

- 존재하지 않는 이메일
- 비밀번호 불일치
- 이메일 인증 미완료
- Refresh Token 만료
- Refresh Token 불일치
- Redis 장애
- JWT 생성 실패
- 유효하지 않은 구글 ID Token (구글 로그인)
- 구글 인증 서버 응답 실패/시간 초과 (구글 로그인)

---

# Token 정책

## Access Token

용도

```
API 인증
```

유효시간

```
30분
```

---

## Refresh Token

용도

```
재로그인 없이 Access Token 재발급
```

유효시간

```
14일
```

Redis에서 TTL로 관리합니다.

---

# 보안

- Access Token은 JWT 서명을 검증합니다.
- Refresh Token은 Redis에만 저장합니다.
- 비밀번호는 BCrypt로 암호화합니다.
- HTTPS 환경에서만 통신합니다.
- Access Token 만료 시 Refresh Token으로 재발급합니다.