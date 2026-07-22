# Auth Service

## 역할

Auth Service는 사용자의 인증(Authentication)과 회원 프로필 관리를 담당하는 서비스입니다.
이메일/비밀번호(LOCAL) 로그인과 구글 소셜 로그인(GOOGLE)을 모두 지원하며, JWT 발급/검증,
이메일 인증, 회원 탈퇴, 휴면 계정 관리를 수행합니다. 역할 기반 접근 제어(관리자 전용 기능
등)는 아직 구현되어 있지 않습니다.

---

# 책임

- 이메일/비밀번호 회원가입 및 이메일 인증
- 이메일/비밀번호 로그인, 구글 소셜 로그인
- JWT(Access/Refresh Token) 발급 및 검증
- 회원 프로필 조회/수정
- 회원 탈퇴 (Soft Delete)
- 휴면 계정 전환 및 재활성화

---

# 주요 기능

## 이메일 회원가입

이메일/비밀번호로 가입합니다. 이메일 인증(6자리 인증번호, Redis TTL 5분)을 거쳐야
`users` 행이 생성됩니다.

---

## 이메일/비밀번호 로그인

BCrypt로 비밀번호를 검증하고 JWT(Access Token 30분, Refresh Token 14일)를 발급합니다.
`status = 'DELETED'`인 계정은 로그인을 거부합니다.

---

## 구글 소셜 로그인

프론트엔드가 구글 로그인으로 받은 ID Token을 `POST /auth/google`로 전달하면, Auth
Service가 구글 공개키로 검증한 뒤 `oauth_user`에서 `(provider='GOOGLE',
providerUserId)`로 연동된 계정을 조회합니다. 없으면 `users`를 자동 생성(비밀번호 없음)하고
`oauth_user` 연동 행도 함께 만듭니다 — 별도의 "구글 회원가입" API는 없습니다.

같은 이메일로 LOCAL 계정과 GOOGLE 계정을 연동하는 기능(계정 연동)은 스키마상 가능하지만
(`oauth_user`가 1:N이므로 한 `users`에 여러 provider를 연동할 수 있음) 실제 연동 플로우는
추후 개발 예정입니다.

---

## 휴면 계정 전환/재활성화

별도 배치 없이 로그인 시도 시점에 판단합니다. 비밀번호 검증에 성공한 뒤 `last_login_at`
기준으로 휴면 기준일(정책값, 예: 90일)을 넘었으면 `status`를 `DORMANT`로 전환하고 이메일
인증번호를 발송합니다(JWT는 이 시점에 발급하지 않음). `POST /auth/login/reactivate`로
인증번호를 검증하면 `status`를 `ACTIVE`로 되돌리고 JWT를 발급합니다. 최초 가입 후 한 번도
로그인하지 않아 `last_login_at`이 NULL이면 휴면 판단에서 제외합니다.

---

## 회원 탈퇴

`users.status`를 `'DELETED'`로 바꾸고 같은 시점에 `deleted_at`을 기록하는 Soft Delete로
처리합니다. `users` 행은 삭제하지 않습니다. 같은 요청 트랜잭션 내에서 Redis의 Refresh
Token도 즉시 삭제해, 탈퇴 응답 시점부터 재로그인이 불가능합니다. 탈퇴 이후
`UserDeletedEvent`를 발행해 Cultivation Service가 해당 사용자의 재배 데이터를
비활성화하도록 합니다. 계정 복구(탈퇴 취소) 기능은 추후 개발 예정입니다.

---

# API

## 회원가입

POST /auth/signup

---

## 이메일 인증

POST /auth/verify-email

---

## 로그인

POST /auth/login

---

## 구글 로그인

POST /auth/google

---

## 휴면 계정 재활성화

POST /auth/login/reactivate

---

## Refresh Token 재발급

POST /auth/refresh

---

## 로그아웃

POST /auth/logout

---

## 내 정보 조회

GET /users/me

---

## 회원 탈퇴

DELETE /users/me

---

# Database

Auth Service는 하나의 PostgreSQL Database를 사용합니다.

## Table

- users (email/nickname UNIQUE, status ACTIVE/DORMANT/DELETED, deleted_at)
- oauth_user (provider 정규화, UNIQUE(provider, provider_user_id))

자세한 내용은 [auth-db.md](../03_Database/auth-db.md) 참고.

---

# Redis

- Refresh Token (`refresh:{userId}`, TTL 14일)
- 이메일 인증번호 (`email:{email}`, TTL 5분)

---

# 다른 서비스와의 통신

## 호출하는 서비스

### 구글 공개키 엔드포인트 (외부)

구글 ID Token 서명/만료 검증 (내부 서비스 간 OpenFeign 호출 아님)

---

## 호출받는 서비스

### API Gateway

회원가입/로그인/탈퇴/프로필 조회 등 전체 REST API 요청

---

# Event

## Publish

### UserDeletedEvent

회원 탈퇴 시 발행. Cultivation Service가 구독해 재배 데이터를 비활성화합니다.

---

# Sequence

관련 시퀀스는 [signup.md](../04_sequence/signup.md), [login.md](../04_sequence/login.md),
[withdraw.md](../04_sequence/withdraw.md) 참고.

---

# 예외 상황

- 이메일 중복
- 인증번호 만료/불일치
- 존재하지 않는 이메일
- 비밀번호 불일치
- 유효하지 않은/만료된 구글 ID Token
- 이미 탈퇴한 사용자(`status = 'DELETED'`)의 로그인 시도
- 휴면 계정 전환 및 재인증번호 불일치/만료
- Refresh Token 만료/불일치

---

# 추후 개발 예정

- 역할 기반 접근 제어(관리자 전용 기능)
- 계정 연동 (같은 사용자에 LOCAL + GOOGLE 동시 연동 플로우)
- 회원 탈퇴 취소(계정 복구)
