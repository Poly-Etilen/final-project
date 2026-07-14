# 로그인 시퀀스

## 개요

사용자가 이메일과 비밀번호를 이용하여 로그인하는 과정입니다.

로그인이 성공하면 Access Token과 Refresh Token을 발급합니다.

Access Token은 Client에 전달되며,
Refresh Token은 Redis에 저장하여 관리합니다.

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
auth_user
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

# 사용 Database

## PostgreSQL

```
auth_user
```

---

## Redis

```
refresh:{userId}
```

---

# OpenFeign

사용하지 않습니다.

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