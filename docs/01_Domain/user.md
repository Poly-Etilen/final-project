# User Service

## 역할

User Service는 사용자의 개인정보 및 프로필을 관리하는 서비스입니다.

회원가입 및 로그인과 같은 인증 기능은 Auth Service에서 담당하며,
User Service는 인증 이후 사용자의 정보를 조회 및 관리하는 역할을 수행합니다.

---

# 책임

- 사용자 정보 조회
- 사용자 정보 수정
- 프로필 관리
- 회원 탈퇴
- 재배 통계 조회

---

# 주요 기능

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

비밀번호 변경은 Auth Service를 통해 수행됩니다.

---

## 회원 탈퇴

회원 탈퇴를 요청합니다.

회원 탈퇴 시

- 사용자 정보 삭제
- 재배 데이터 비활성화
- Refresh Token 삭제

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

User Service는 별도의 PostgreSQL Database를 사용합니다.

### Table

- users

---

# Redis

사용하지 않습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

없음

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

- Auth Service (Refresh Token 삭제)
- Cultivation Service (재배 데이터 비활성화)

---

# Sequence

## 사용자 정보 조회

Client

↓

Gateway

↓

User Service

↓

PostgreSQL

↓

Client

---

## 회원 탈퇴

Client

↓

Gateway

↓

User Service

↓

User Soft Delete

↓

UserDeletedEvent 발행

↓

├── Auth Service → Refresh Token 삭제
└── Cultivation Service → 재배 데이터 비활성화

---

# 예외 상황

- 존재하지 않는 사용자
- 중복 닉네임
- 이미 탈퇴한 사용자

---

# 추후 개발 예정

- 프로필 이미지 업로드
- 사용자 활동 통계
- 재배 성과 랭킹
- 업적(Achievement) 시스템