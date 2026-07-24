# Auth Service

## 역할

Auth Service는 사용자의 인증(Authentication)과 회원 프로필 관리를 담당하는 서비스입니다.
이메일/비밀번호(LOCAL) 로그인과 구글 소셜 로그인(GOOGLE)을 모두 지원하며, JWT 발급/검증,
이메일 인증, 회원 탈퇴, 휴면 계정 관리를 수행합니다. `users.role`(USER/ADMIN)로 시스템
관리자 여부를 구분하며, 이 값은 JWT 클레임에 담겨 다른 서비스가 관리자 전용 기능(예:
Cultivation Service의 문의 답변/재배 삭제)을 인가할 때 사용합니다.

---

# 책임

- 이메일/비밀번호 회원가입 및 이메일 인증
- 이메일/비밀번호 로그인, 구글 소셜 로그인
- JWT(Access/Refresh Token) 발급 및 검증 (role 클레임 포함)
- 회원 프로필 조회/수정
- 프로필 이미지 업로드/삭제 (신규)
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

## 프로필 이미지 업로드/삭제 (신규)

사용자가 프로필 이미지를 업로드하면 Photo Storage(MinIO 또는 Local)에 저장하고
`profile_image` 행(`object_key`, `storage_type`)을 남깁니다. Auth Service가 Photo
Storage를 사용하는 것은 이번이 처음입니다 — 기존에는 Cultivation Service만 생육
사진(`cultivation_photo`)을 위해 사용했습니다. 저장 패턴은 동일하게 전체 URL이 아닌
`object_key` + `storage_type`만 저장하며, 실제 접근 경로는 조회 시점에 계산합니다.

`profile_image.user_id`는 `UNIQUE` 제약으로 사용자당 한 장만 허용합니다. 이미
이미지가 있는 상태에서 다시 업로드하면 기존 행을 교체합니다(이력을 남기지 않음).
삭제 API를 호출하면 Photo Storage의 실제 파일과 `profile_image` 행을 함께 제거합니다.
기존 `users.profile_image_url`(URL 문자열 직접 저장) 컬럼은 제거되었습니다.

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

## 프로필 이미지 (신규)

POST /users/me/profile-image

DELETE /users/me/profile-image

---

## 회원 탈퇴

DELETE /users/me

---

# Database

Auth Service는 하나의 PostgreSQL Database를 사용합니다.

## Table

- users (email/nickname UNIQUE, role USER/ADMIN, status ACTIVE/DORMANT/DELETED, deleted_at)
- oauth_user (provider 정규화, UNIQUE(provider, provider_user_id))
- profile_image (object_key + storage_type, UNIQUE(user_id), 신규)

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

### Photo Storage (신규)

프로필 이미지 업로드/삭제 시 MinIO 또는 Local 저장소에 파일을 저장/삭제합니다. 별도
서비스 호출이 아니라 Cultivation Service와 동일한 방식으로 Auth Service가 직접 저장소
클라이언트를 사용합니다. Auth Service가 Photo Storage를 사용하는 것은 이번이
처음입니다.

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
- 프로필 이미지 저장소 업로드/삭제 실패
- 지원하지 않는 이미지 파일 형식

---

# 추후 개발 예정

- 계정 연동 (같은 사용자에 LOCAL + GOOGLE 동시 연동 플로우)
- 회원 탈퇴 취소(계정 복구)
- 관리자 계정 최초 생성/부여 절차 확정 (현재는 `role = ADMIN` 부여 방식이 별도
  문서화되어 있지 않음)
