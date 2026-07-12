# User Scenario

## 1. 회원가입

사용자는 이메일 또는 OAuth를 이용하여 회원가입을 진행한다.

---

## 2. 로그인

로그인 후 Workspace 목록 화면으로 이동한다.

---

## 3. Workspace 생성

사용자는

"새 Workspace"

버튼을 클릭한다.

---

## 4. AI 추천

사용자는

"몬스테라 키우기 좋은 환경으로 구성해줘."

라고 입력한다.

AI는

- 온도
- 습도
- CO₂
- pH

를 추천한다.

---

## 5. 환경 수정

사용자는 AI가 추천한 환경을 자신의 환경에 맞게 수정한다.

---

## 6. Workspace 생성

환경 구성 완료 버튼을 누르면

Workspace와 EnvironmentSetting이 저장된다.

사용자는 Workspace의 ADMIN이 된다.

---

## 7. Device 등록

MQTT 센서를 Workspace에 연결한다.

---

## 8. 실시간 모니터링

Dashboard에서

- 온도
- 습도
- CO₂
- pH

를 실시간으로 확인한다.

---

## 9. 이상 상황

Rule Engine이 이상 상태를 감지한다.

Notification Service가

- WebSocket
- Telegram
- Discord

알림을 전송한다.

---

## 10. AI Report

Storage Service가

주간 데이터를 집계한다.

AI Service는

주간 리포트를 생성한다.

Dashboard에서 조회할 수 있다.