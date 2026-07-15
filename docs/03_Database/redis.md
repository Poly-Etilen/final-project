# Redis

## 개요

Redis는 실시간 데이터와 임시 데이터를 저장하기 위해 사용합니다.

본 프로젝트에서는 영구적으로 저장할 필요가 없는 데이터와
빠른 조회가 필요한 데이터를 관리합니다.

---

# 사용 서비스

| Service | 용도 |
|----------|------|
| Auth Service | Refresh Token |
| Auth Service | 이메일 인증 |
| AI Service | AI 응답 캐시 |
| Rule Engine Service | 최신 센서 데이터 |

---

# Key 구조

## Auth

### Refresh Token

Key

```
refresh:{userId}
```

Example

```
refresh:15
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

### Email Verification

Key

```
email:{email}
```

Example

```
email:test@example.com
```

Value

```
845132
```

TTL

```
5분
```

---

# AI

## AI Response Cache

사용자의 동일한 질문에 대해
LLM 호출을 줄이기 위해 사용합니다.

Key

```
ai:{hash}
```

Example

```
ai:82ad7f19...
```

Value

```json
{
  "answer": "...",
  "createdAt": "2026-08-15T10:22:30"
}
```

TTL

```
24시간
```

---

# Sensor

## Current Environment

실시간 대시보드 조회를 위한
최신 센서 데이터입니다.

Key

```
cultivation:{cultivationId}:current
```

Example

```
cultivation:3:current
```

Value

```json
{
  "temperature":22.5,
  "humidity":91.2,
  "co2":820,
  "light":430,
  "updatedAt":"2026-08-15T10:22:30"
}
```

TTL

```
없음
```

최신 데이터가 들어올 때마다 덮어씁니다.

---

# Redis 사용 목적

## Auth Service

저장 데이터

- Refresh Token
- 이메일 인증번호

---

## AI Service

저장 데이터

- AI 응답
- 프롬프트 캐시

---

## Rule Engine Service

(기존 Sensor Service 역할 포함)

저장 데이터

- 최신 온도
- 최신 습도
- 최신 CO₂
- 최신 조도

---

# 데이터 흐름

## Refresh Token

```
Login

↓

JWT 발급

↓

Redis 저장
```

---

## Email Verification

```
인증번호 생성

↓

Redis 저장

↓

5분 후 자동 삭제
```

---

## AI Cache

```
질문

↓

Redis 조회

↓

있음

↓

바로 응답

----------------

없음

↓

LLM 호출

↓

Redis 저장
```

---

## Sensor Cache

```
MQTT 수신

↓

Rule Engine Service (규칙평가 + 저장을 함께 처리)

↓

Redis 저장

↓

Dashboard 조회
```

---

# 메모리 관리

TTL을 사용하는 데이터

- Refresh Token
- 이메일 인증
- AI Cache

TTL을 사용하지 않는 데이터

- 최신 센서 데이터

---

# 장애 대응

Redis 장애 발생 시

## Auth

- 로그인은 가능
- Refresh Token 재발급 불가

---

## AI

- Cache Miss 처리
- LLM 직접 호출

---

## Sensor

- InfluxDB에서 최신 데이터 조회
- 성능은 다소 저하될 수 있음

---

# 추후 개발 예정

- Redis Cluster
- Pub/Sub
- Stream
- 분산 Lock