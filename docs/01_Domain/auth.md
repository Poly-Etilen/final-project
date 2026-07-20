# Auth Service

## 역할

Auth Service는 사용자의 인증(Authentication)/인가(Authorization)와 회원 프로필 관리를 함께 담당하는 서비스입니다.

기존에는 Auth Service(인증)와 User Service(프로필)로 분리되어 있었으나,
두 서비스가 사실상 동일한 사용자 엔티티를 다루고 회원가입/탈퇴 시마다 서비스 간 호출이 필요해
운영 복잡도만 늘어난다고 판단하여 하나의 서비스로 통합했습니다.

JWT 기반 인증, 이메일 인증, 회원 프로필 조회/수정, 회원 탈퇴, 재배 통계 조회까지 모두 이 서비스가 처리합니다.

> ℹ️ **변경 이력**: 구글 소셜 로그인이 추가되었습니다("추후 개발 예정"에서 승격). 이메일/비밀번호
> 회원가입(LOCAL)과 구글 로그인(GOOGLE) 두 가지 방식을 지원하며, `users.provider`로
> 구분합니다. 회원 탈퇴는 `status`(ACTIVE/DORMANT/DELETED) 컬럼으로 표현하되, 정확한 탈퇴
> 시각은 `deleted_at`에 별도로 기록합니다. (자세한 내용은 [auth-db.md](../03_Database/auth-db.md) 참고)

> ℹ️ **변경 이력**: 휴면 계정(DORMANT) 관리가 추가되었습니다. 이메일 인증은 회원가입 시점에
> 이미 필수라 "인증 대기" 상태는 없지만, 장기간 로그인하지 않은 계정은 로그인 시도 시점에
> 휴면으로 전환되고 이메일 재인증을 거쳐야 다시 로그인할 수 있습니다.

---

# 책임

- 회원가입 (이메일/비밀번호, 구글 소셜 로그인)
- 로그인 / 로그아웃
- JWT 발급 및 검증
- Refresh Token 관리
- 이메일 인증
- 사용자 정보 조회/수정
- 프로필 관리
- 회원 탈퇴
- 재배 통계 조회

---

# 주요 기능

## 회원가입

사용자는 이메일을 통해 회원가입을 진행합니다.

이메일 인증이 완료되어야 회원가입이 가능합니다.

프로필(닉네임 등)은 회원가입 시 기본값으로 함께 생성되며, 별도 서비스 호출 없이 하나의 트랜잭션으로 처리됩니다.

---

## 로그인

이메일과 비밀번호를 검증한 후 JWT를 발급합니다.

비밀번호 검증에 성공해도 계정이 휴면 상태이거나 휴면 기준을 넘었으면 바로 로그인시키지 않고,
이메일 재인증을 먼저 요구합니다. 자세한 내용은 아래 "휴면 계정 관리" 참고.

---

## 휴면 계정 관리

이메일 인증은 회원가입 시점에 이미 필수이므로 별도의 "인증 대기" 상태는 두지 않습니다. 대신
장기간(기준일 예시: 90일) 로그인하지 않은 계정은 휴면(`DORMANT`)으로 전환하여 방치된 계정의
보안 위험을 줄입니다.

- 전환 시점: 별도 배치가 아니라, 사용자가 로그인을 시도한 그 순간에 `last_login_at` 기준으로
  판단합니다.
- 전환 시 처리: `status`를 `DORMANT`로 바꾸고, 이메일로 인증번호를 발송합니다. 이 시점에는
  로그인이 완료되지 않습니다(토큰 미발급).
- 재활성화: 사용자가 인증번호를 제출하면 `status`를 다시 `ACTIVE`로 바꾸고, 최초 로그인
  시도(비밀번호 검증까지 마친 상태)를 그대로 이어서 완료해 토큰을 발급합니다.

---

## 구글 소셜 로그인

사용자가 프론트엔드에서 구글 로그인으로 받은 ID Token을 Auth Service에 전달하면, 구글 공개키로
서명을 검증하고 이메일을 추출합니다.

- 기존에 같은 이메일+GOOGLE 조합의 계정이 있으면 바로 로그인 처리(JWT 발급)합니다.
- 없으면 자동으로 회원가입합니다(provider=GOOGLE, password 없음, email_verified=true,
  닉네임/프로필 이미지는 구글 프로필 정보를 기본값으로 사용). 별도 이메일 인증 절차를 거치지
  않습니다(구글이 이미 검증한 이메일이기 때문).
- 이메일/비밀번호(LOCAL)로 가입한 계정과는 별개의 계정으로 취급합니다. 같은 이메일이라도
  provider가 다르면 다른 사용자로 저장됩니다(계정 연동은 추후 개발 예정).

---

## JWT 인증

Access Token을 검증합니다.

---

## Refresh Token

Redis에 저장된 Refresh Token을 이용하여 Access Token을 재발급합니다.

---

## 이메일 인증

회원가입 시 이메일 인증번호를 발송하고 검증합니다.

---

## 내 정보 조회

로그인한 사용자의 정보를 조회합니다.

조회 정보

- 이메일
- 닉네임
- 프로필 이미지
- 가입일

---

## 사용자 정보 수정

사용자는 자신의 정보를 수정할 수 있습니다.

수정 가능 항목

- 닉네임
- 프로필 이미지

---

## 비밀번호 변경

기존 비밀번호를 확인한 후 새로운 비밀번호로 변경합니다.

---

## 회원 탈퇴

회원 탈퇴를 요청합니다.

회원 탈퇴 시

- 사용자 정보 Soft Delete (`status`를 `'DELETED'`로 변경, `deleted_at`에 탈퇴 시각 기록)
- Refresh Token 삭제 (같은 서비스 내부 처리이므로 즉시 수행)
- 재배 데이터 비활성화 (Cultivation Service에 이벤트 발행)

를 수행합니다.

---

## 재배 통계 조회

사용자의 전체 재배 이력을 기반으로 통계를 제공합니다.

예시

- 총 재배 횟수
- 총 수확량
- 평균 재배 기간
- 가장 많이 재배한 버섯

---

# API

## 회원가입

POST /auth/signup

---

## 로그인

POST /auth/login

---

## 휴면 계정 재활성화

POST /auth/login/reactivate

---

## 구글 로그인

POST /auth/google

---

## 로그아웃

POST /auth/logout

---

## Access Token 재발급

POST /auth/refresh

---

## 이메일 인증 요청

POST /auth/email/send

---

## 이메일 인증 확인

POST /auth/email/verify

---

## 내 정보 조회

GET /users/me

---

## 사용자 정보 수정

PATCH /users/me

---

## 회원 탈퇴

DELETE /users/me

---

## 내 재배 통계 조회

GET /users/me/statistics

---

# Database

Auth Service는 하나의 PostgreSQL Database를 사용합니다.

### Table

- users (인증 정보 + 프로필 정보 통합)

Auth+User가 통합되기 전에는 auth_user / users 두 테이블로 나뉘어 있었으나,
서비스 통합과 함께 하나의 users 테이블로 합쳤습니다. 자세한 내용은 auth-db.md 참고.

---

# Redis

### Refresh Token

Key

```
refresh:{userId}
```

Value

```
Refresh Token
```

---

### 이메일 인증

Key

```
email:{email}
```

Value

```
인증번호
```

---

# 다른 서비스와의 통신

## 호출하는 서비스

없음

Auth+User 통합 이전에는 회원가입 시 User Service를 OpenFeign으로 호출했지만,
통합 이후에는 같은 서비스 내부 로직이라 별도 호출이 필요 없습니다.

---

## 호출받는 서비스

- API Gateway
- Cultivation Service (사용자 정보 조회)

---

# Event

## 발행 이벤트

### UserDeletedEvent

회원 탈퇴 시 발행됩니다.

구독 서비스

- Cultivation Service (재배 데이터 비활성화)

Refresh Token 삭제는 같은 서비스 내부 처리이므로 이벤트로 발행하지 않고 즉시 수행합니다.

---

# Sequence

## 로그인

Client

↓

Gateway

↓

Auth Service

↓

status 판단 (DELETED면 거부 / 휴면 기준 초과면 DORMANT 전환 + 인증번호 발송)

↓

JWT 발급 (정상 로그인이거나, 재활성화 인증번호 확인 후)

↓

Client

자세한 분기는 [login.md](../04_sequence/login.md) 참고.

---

## 구글 로그인

Client (구글 로그인 후 ID Token 보유)

↓

Gateway

↓

Auth Service

↓

ID Token 검증 (구글 공개키)

↓

이메일+provider(GOOGLE)로 users 조회

↓

없으면 자동 회원가입 (provider=GOOGLE, email_verified=true)

↓

JWT 발급

↓

Client

---

## 회원 탈퇴

Client

↓

Gateway

↓

Auth Service

↓

users Soft Delete + Refresh Token 삭제 (하나의 트랜잭션/처리 내에서 수행)

↓

RabbitMQ Publish (UserDeletedEvent)

↓

Cultivation Service → 재배 데이터 비활성화

---

# 예외 상황

- 존재하지 않는 이메일
- 비밀번호 불일치
- 만료된 Access Token
- 만료된 Refresh Token
- 이메일 인증 실패
- 중복 닉네임
- 이미 탈퇴한 사용자
- 유효하지 않은 구글 ID Token
- 구글 서버 응답 실패/시간 초과
- 휴면 계정 (이메일 재인증 필요)
- 재활성화 인증번호 불일치 또는 만료

---

# 추후 개발 예정

- LOCAL/GOOGLE 계정 연동 (같은 이메일의 두 계정을 하나로 합치는 기능)
- Kakao 로그인
- Naver 로그인
- 2차 인증(MFA)
- 프로필 이미지 업로드
- 재배 성과 랭킹
- 업적(Achievement) 시스템
